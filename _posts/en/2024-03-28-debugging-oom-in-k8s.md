---
title: "Why my Kubernetes rails app returns 502 errors?"
layout: post
tags: [blog, til, k8s, oom, cgroups]
categories: [blog]
lang: 
---

Why one app can handle >10k requests per second without sweating a bit
_[pun intended]_ and another one with two orders of magnitude less traffic gets
sporadic 50X errors served by AWS ELB.

Kubernetes never stops surprising you the more you use it. This past week has
been riddled with learning *very* obscure shit about linux internals.

<!-- more -->

It all starts with the problem:

> Sometimes when accessing the workload, AWS ELB takes over and returns a 502
  response.

And a small piece of context: The workload in question is a [`Rails`][gh-rails]
application served by [`puma`][gh-puma]

Getting a 502 from AWS ELB's can be caused for a myriad of reasons, (just see
[AWS official KB][aws-elb-502-kb]).

They range from:

* Layer 4 issues (TCP RST, FIN, et all).
* TLS handshake problems.
* Lambda issues [not applicable to us], etc. 

Given the situation, your heart immediately says: _READ THE LOGS!_

_Aaaaaaaand_, the only observable consequence that we're seeing aside from the ELB taking over
the response is that puma logs:

```
[1] - Worker {0|1} (PID: \d+) booted in 0.01s, phase: 0
```

Nothing else, no stacktrace, no error message, just
`booted in «milliseconds», phase: 0`.

After striding through the interwebs across irrelevant answers (google results
are getting shittier these days...), I stumbled upon this discussion
[this discussion (puma/puma#3193) on the official puma repository][gh-puma-oom-discussion]
<small>(no idea how to shorten discussion links).</small>

The people in the discussion are experiencing the same issue as us and running
in similar conditions: 

Puma under Kubernetes behind a reverse proxy (nginx), the proxy is returning 502.

The discussion goes back and forth over but it raises a key thing: `phase: 0`
indicates that a puma worker is starting from scratch.

The people in the discussion eventually shown a memory consumption graph going
up and down and get to the core of the OOM.

Sadly I don't have the same metrics as them, but that gave me an idea: _What if I'm being hit by the OOM killer?_

The problematic workload has (as kubernetes best practices suggest) resource
limits, I don't know if those are the problem just right now, so I started with
a container where I can run as root, ran the workload and inspected `dmesg`

```
# dmesg | grep oom
# [...]
[217532.681497] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=cri-containerd-e7364a949ee841f15422656ccd50b92c5ae2ca81d5e1d124e37da3dc4f0da480.scope,mems_allowed=0,oom_memcg=/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod0057cc81_ea16_463a_96c2_fc74acfb9c47.slice/cri-containerd-e7364a949ee841f15422656ccd50b92c5ae2ca81d5e1d124e37da3dc4f0da480.scope,task_memcg=/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod0057cc81_ea16_463a_96c2_fc74acfb9c47.slice/cri-containerd-e7364a949ee841f15422656ccd50b92c5ae2ca81d5e1d124e37da3dc4f0da480.scope,task=ruby,pid=2138551,uid=1000
[217532.722510] Memory cgroup out of memory: Killed process 2138551 (ruby) total-vm:1313528kB, anon-rss:977464kB, file-rss:12800kB, shmem-rss:0kB, UID:1000 pgtables:2480kB oom_score_adj:969
```

That's a good sign that the OOM killer is acting up!

But it's difficult to know if it's affecting my container or not, since many
pods can run on the same node... it may be hitting something else in the node.
I need to isolate me out of the crowd. That slice name
(`cri-containerd-*.scope`) looks _unique-ish enough_, so I used the following
to get a hold of some needle:

```
cat /proc/1/cgroup | cut -f4,5 -d/ # get the cgroup of the main process of the container
kubepods-burstable-pod0057cc81_ea16_463a_96c2_fc74acfb9c47.slice/cri-containerd-e7364a949ee841f15422656ccd50b92c5ae2ca81d5e1d124e37da3dc4f0da480.scope
# [...]

CGROUP_SLICE=$(cat /proc/1/cgroup | cut -f4,5 -d/ | head -n 1) # save it for late
```

That represents my current cgroup slice and I'm 90% sure that's unique enough to
assert that messages including those identifiers must be related to my pod.

To verify that I'm not shouting at the air, I caught the last `dmesg` message:

```
[266006.845118] IPv6: ADDRCONF(NETDEV_CHANGE): eth0: link becomes ready
```

And will try to break the workload, when I get a:

```
[1] - Worker {0|1} (PID: \d+) booted in 0.01s, phase: 0
```

On the logs, I should see an `oom` message with a time (the number in brackets
at the beginning of the line) bigger than the latest `dmesg` that matches my
slice.

So I put on my metaphorical _Wreck-It Ralph_ gloves (wrote an apache bench
command, same thing) and ran:

```
ab -c 100 -n 500 'https://workload.under.test.internal/random/endpoint'
```

And sure enough, around ~20 secs later I saw in the puma logs:

```
[1] - Worker 1 (PID: 206) booted in 0.06s, phase: 0
```

A 502! Now let's hunt `dmesg`!

```
# dmesg | grep oom | grep "$CGROUP_SLICE"
[266207.275678] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=cri-containerd-28b9d205e931ff342192577daa3ba713418521f515c81bec863e7aceac4ee11c.scope,mems_allowed=0,oom_memcg=/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod8521910b_28a1_41b4_a87e_9a5df0311cb4.slice/cri-containerd-28b9d205e931ff342192577daa3ba713418521f515c81bec863e7aceac4ee11c.scope,task_memcg=/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod8521910b_28a1_41b4_a87e_9a5df0311cb4.slice/cri-containerd-28b9d205e931ff342192577daa3ba713418521f515c81bec863e7aceac4ee11c.scope,task=ruby,pid=2139881,uid=1000

[266216.657219] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=cri-containerd-28b9d205e931ff342192577daa3ba713418521f515c81bec863e7aceac4ee11c.scope,mems_allowed=0,oom_memcg=/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod8521910b_28a1_41b4_a87e_9a5df0311cb4.slice/cri-containerd-28b9d205e931ff342192577daa3ba713418521f515c81bec863e7aceac4ee11c.scope,task_memcg=/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod8521910b_28a1_41b4_a87e_9a5df0311cb4.slice/cri-containerd-28b9d205e931ff342192577daa3ba713418521f515c81bec863e7aceac4ee11c.scope,task=ruby,pid=2139869,uid=1000

[266217.445414] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=cri-containerd-28b9d205e931ff342192577daa3ba713418521f515c81bec863e7aceac4ee11c.scope,mems_allowed=0,oom_memcg=/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod8521910b_28a1_41b4_a87e_9a5df0311cb4.slice/cri-containerd-28b9d205e931ff342192577daa3ba713418521f515c81bec863e7aceac4ee11c.scope,task_memcg=/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod8521910b_28a1_41b4_a87e_9a5df0311cb4.slice/cri-containerd-28b9d205e931ff342192577daa3ba713418521f515c81bec863e7aceac4ee11c.scope,task=ruby,pid=2475481,uid=1000
```

**GREAT NEWS:** We're being killed by the OOM! 🪓 <small>(well not really but
  having certainty is great news (?))</small>

They events are newer than the last event I captured in `dmesg` (`266006 < 266207`).

So that settles it! **The OOM killer is killing the puma workers when they're
exceeding the max ram allowed for their container**

Now... we got questions:

- How can I make this more visible so I don't need to _conjure_ all of these
  commands?
- How to avoid this problem condition?

For the first question, I navigated all metrics I had available and sadly in my
case I lack resolution, I can only see when the container died, the memory does
not look like a memory leak (ram increasing but never decreasing) but rather
like spikes, and the moment the spike hits maximum ram defined: OOM Killer 🪓

The spikes are faster than the resolution that what the cluster monitoring can
see, I can't change that just now. It'll have to stay unanswered...

But in an ideal scenario, cAdvisor metrics scraped at a reasonable rate +
`kube_container_pod_*` metrics can probably bring a good grafana chart for
alerting close to be killed containers 🤔

As for the second, in the discussion upstream they go back and forth on how to
setup the worker-threads ration: One worker and a gazillion threads vs
a gazillion workers and a few threads.

There's no silver bullet here in all honesty, when we were doing Performance
Tuning for the first application _[the one that does not OOM]_, we established
that:

```
1 worker -> 24:48 threads
```

Was an optimal configuration for that workload. But the other workload
_[for historical reasons]_ also features [PumaWorkerKiller (PWK)][gh-pwk].

PWK kills puma workers that are consuming too much RAM, but it does it
gracefully _[Sending `SIGTERM` instead of `SIGKILL`]_ so current requests _are
finished_ before closing the worker process.

In this case is configured to start _🪓Killing Workers🪓_ when workers are
approaching around 75% of the memory limit, that in itself proved to be a very
solid safety net.

It took me _apachebenching_ some heavy export endpoints with a good amount of
concurrency to try hitting the PWK, I couldn't hit the OOM Killer at all.

Nevertheless it helped me understand the memory consumption behavior of our
problematic. More and more it seems that is mostly sudden spikes rather than a
gradual increase of consumption.

With this in mind I tried two approaches:

* Increase RAM limits.
* Started re-tweaking the worker/thread ratio again.

I increased RAM by (`1.5G` -> `2G`) 30% and did not see a benefit, Tried out
`2.5G` and gave me no visible result, I could still hit the `502` quite easily.

So I continued with the worker/thread ratio, I remembered that every thread
count had a inherent correlation with RAM. The thread pool used to be:

```
1 worker -> 24:48 threads [min:max]
```

I tried halving the number until I found that:

```
1 worker -> 4:6 threads # ok 6 is not half, but 8 still was flaky
```

Gave me a noticeable benefit (it took me twice to thrice the time under
apachebench traffic to see a 502).

Based off that, for this particular workload, the _rough_ rule of thumb for tuning
ram, Workers & threads I came up with is:

```
MAX(1G, 170~200mb per thread per worker)
```

That looks like the sweet spot in terms of resource allocation. The problem is
definitely not solved, but I learned:

- Spot if your current container processes are being killed by the OOM Killer.
- A _rule of thumb_ [YMMY] for puma tuning.
- I have more work now: Placing a Horizontal Pod Autoscaler (HPA) there to
  distribute the load a bit on those spikey requests.

[aws-elb-502-kb]: https://repost.aws/knowledge-center/elb-alb-troubleshoot-502-errors
[gh-rails]: https://github.com/rails/rails
[gh-puma]: https://github.com/puma/puma
[gh-puma-oom-discussion]: https://github.com/puma/puma/discussions/3193
[gh-pwk]: https://github.com/zombocom/puma_worker_killer