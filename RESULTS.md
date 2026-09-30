# Results

What broke in the shared stack, how it showed up, and the numbers before and
after. Everything here was found by running something, not by reading the
config: the whole class of defect below answers with a plausible number instead
of an error.

## Scraping and discovery

**A target disappeared because its timeout exceeded its interval.** A
per-container `observability.interval` shorter than the default 10 s scrape
timeout makes Prometheus reject the target outright rather than clamp it. The
timeout is now derived from the interval whenever the interval is set. The
class: a rule that rejects, rejects silently.

**All three nodes vanished from discovery after the network was renamed.**
`docker_sd` reports a target per network and which networks it reports depends
on the container's network mode, so a config that kept only `observability`
dropped every container whose primary network was its own. Containers are now
addressed by name - every duplicate target collapses to the same address and
docker's embedded DNS resolves it:

```yaml
- source_labels: [__meta_docker_container_name, __meta_docker_container_label_observability_port]
  regex: /?(.+);(\d+)
  replacement: $1:$2
  target_label: __address__
```

The class: discovery that depends on incidental names. Nothing alerted, because
a target that was never discovered is not a target that went down.

**The machine was counted as part of the stack.** cAdvisor exports the root
cgroup (`id="/"`) next to the containers. It has no compose labels, so nothing
overrode the scrape target's own `project=observability`, and every sum over
that project quietly included the whole host:

| | before | after |
|---|---|---|
| stack memory panel | 9.67 GB | 2.01 GB |
| stack CPU panel | 0.73 cores | 0.063 cores |

For reference the host itself was using 9.42 GB at the time, which is what the
panel had really been showing. The root series is dropped rather than
relabelled - the host has its own source in node-exporter - and labels inherited
by containers started outside compose are reset to `unlabeled`, since they fall
into the same trap for the same reason.

## Metrics that lie rather than break

**Span metrics counted every request twice.** An inner span carried the same
name as the server span, so every rate derived from span names was doubled.
Inner spans are now named `call <method>`.

**`provider_budget_remaining` read 0 on nodes that have no budget.** A gauge
that is absent and a gauge that is zero mean different things, and zero is the
more dangerous default: it reads as "exhausted". Nodes without quotas now
publish `NaN`, which graphs as a gap.

**k6 exports seconds, its console prints milliseconds.** Trend metrics arriving
by remote write are in seconds, so a panel built to match the run summary is off
by 1000. Documented rather than converted.

## Alerting on the wrong source

**The only service-level alert was built on sampled data.**
`ServiceHighErrorRate` divided two span-metric rates. That works only while
every span is kept:

- a sampler that keeps all errors and 1% of the rest makes the ratio read near
  100% on a healthy service;
- a uniform 1% leaves about 12 spans in a 2-minute window at 10 RPS, which
  cannot distinguish 5% errors from noise.

Measured cost of keeping 100% instead: 41% of the traced service's CPU and
0.488 cores across the pipeline against 0.296 spent serving, plus 4 TB/month of
trace data from a single node at 2000 RPS. So sampling is not optional, and the
alert had to move first.

Rate, errors and the alert now read normalised recording rules over the
services' own scraped counters (`prometheus/rules/red.yml`); a project joins by
adding one group. Traces keep the per-operation table, labelled as sampled.
Latency stays on the raw histogram, because exemplars are attached to samples at
ingest and a recording rule writes new samples without them - moving that panel
to a rule would have removed the click-through to a trace without any sign.

**Zero and absent are different, inside a rule too.** Each recorded rule adds
`or` a zeroed copy of its own label sets. Without it the error series is simply
missing while nothing is failing, the ratio has nothing to divide, and both the
alert and the panel go blank - indistinguishable from a dead target. Verified in
both states: `NaN` before, `0` after.

## Operations

**Only Prometheus stayed down after a docker restart, and the stack looked
healthy.** It had been stopped by hand an hour earlier (`signal=terminated`,
exit 0). With `restart: unless-stopped`, docker deliberately does not bring back
what a human stopped, so everything else returned and it did not. Grafana
opened, the services answered, nothing alerted - because the alerts live in the
process that was down. A gap in the graphs was the only symptom.
