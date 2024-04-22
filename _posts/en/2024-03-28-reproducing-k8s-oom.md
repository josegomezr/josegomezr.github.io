---
title: "Reproducing OOM Errors with Docker/Podman/*Containers"
layout: post
tags: [blog, til, k8s, oom, cgroups]
categories: [blog]
lang: 
---

From the previous adventures in: [Why my Kubernetes rails app returns 502 errors?]({{ 'blog/2024/03/28/debugging-oom-in-k8s' | relative_url }}), do I really need a Kubernetes cluster
to experience an OOM in my workload? That seems excessive...

<!-- more -->

I couldn't let go this problem as a "deployment environment limitation", I
needed to have something I could reproduce faster at my reach.

That workload is mostly developed using [`docker-compose`][docker-compose], I do
remember that docker has memory limits:

```
docker run --help | grep memory
      --kernel-memory bytes            Kernel memory limit
  -m, --memory bytes                   Memory limit
      --memory-reservation bytes       Memory soft limit
      --memory-swap bytes              Swap limit equal to memory plus swap: '-1' to enable unlimited swap
      --memory-swappiness int          Tune container memory swappiness (0 to 100) (default -1)
```

Maybe I could get to the same conditions in the cluster if
[`docker-compose`][docker-compose] passes `--memory` to the containers it spins... 🤔
but _how?_🤔🤔🤔

And it just so happens that... it's possible!

From the [horse's mouth][compose-file-v3]:

> **deploy**
>
> [...]
>
> Specify configuration related to the deployment and running of services. The
> following
> sub-options only takes effect when deploying to a swarm with docker stack
> deploy, and is ignored by `docker-compose up` and `docker-compose run`, except
> for `resources`.

It looks like it's a matter of adding:

```yaml
version: '3.0' # this is what the workload had
services:
  workload:
    image: registry.internal/workload:base
    # vvv this part
    deploy:
      resources:
        limits:
          memory: 1G
        reservations:
          memory: 512M
    # ^^^ this part
```

To the `service` definition and we get the sweet sweet OOM locally! Let's give
it a try!

```bash
docker-compose -f docker-compose.yml run workload bin/puma -C config/puma.rb | grep 'Worker ' 
# Just to keep those lines of puma worker restart
``` 

It seems to boot just fine but no matter how hard I hit the workload I can't
seem to kill it... Maybe the memory limits aren't properly set?

{% raw %}
```bash
# i'm lazy, this gives me the container name I need
# docker ps --filter name=container --format "{{ .Names }}"

docker inspect $(docker ps --filter name=container --format "{{ .Names }}") | grep -i memory
            "Memory": 1073741824,
            "MemoryReservation": 536870912,
            "MemorySwap": 2147483648,
            "MemorySwappiness": null,
```
{% endraw %}

Hmm, that swap... it seems that swap is counted as usable memory, then we're not
having really a `memory: 1G` limit, but rather `3G`...

Maybe setting it to half of what you expect to have `512M vs 1G` could get us to
an exploitable values, but I went a tad deeper into this container beast... and
went with:

```bash
# Run the workload...
docker-compose -f docker-compose.yml run workload bin/puma -C config/puma.rb | grep 'Worker ' 
```

In another shell then:

{% raw %}
```bash
docker container update --memory 1G --memory-swap 1025M $(docker ps --filter name=container --format "{{ .Names }}")

docker inspect $(docker ps --filter name=container --format "{{ .Names }}") | grep -i memory
            "Memory": 1073741824,
            "MemoryReservation": 536870912,
            "MemorySwap": 1074790400,
            "MemorySwappiness": 0,

```
{% endraw %}

Aha! That certainly looks better :D

But I still couldn't hit that OOM behavior that easily, so I brought a last
quick tweak to this whole setup: Import the OOM behavior from the cluster.

I still was seeing:
```
[1] - Worker 1 (PID: 206) booted in 0.06s, phase: 0
```

And nothing else.

When I was `ls`-ing shit around in the previous post, I found some `oom_*` files
in the `/proc/` filesystem out of random. Didn't pay much attention to them,
but now after reading [the kernel documentation about those `oom_*` files][kernel-proc-oom]

> These files can be used to adjust the badness heuristic used to select which process gets killed in out of memory (oom) conditions.
> 
> The badness heuristic assigns a value to each candidate task ranging from 0 (never kill) to 1000 (always kill) to determine which process is targeted.
>
> [...]
>
> The value of `/proc/<pid>/oom_score_adj` is added to the badness score before it is used to determine which task to kill. Acceptable values range from -1000 (`OOM_SCORE_ADJ_MIN`) to +1000 (`OOM_SCORE_ADJ_MAX`)
>
> [...]
>
> For backwards compatibility with previous kernels, `/proc/<pid>/oom_adj` may also be used to tune the badness score. Its acceptable values range from -16 

I went ahead and checked the cluster for values and noticed that when the
workload is running in the cluster under the memory limits, the proc files
related to OOM have some values that my local machine don't have.

```
# cat /proc/1/oom_score
«something close to 800»

# cat /proc/self/oom_score_adj
«something close to 1000»
```

Following the docs and this, then I should **write** to
`/proc/<pid>/oom_score_adj` the value of `1000` to make the OOM unforgiving
(just like the cluster).

---

I ended up with the following script:

```bash
# Run the workload...
docker-compose -f docker-compose.yml run workload bin/puma -C config/puma.rb | grep 'Worker ' 
```

In another shell then:

{% raw %}
```bash
docker container update --memory 1G --memory-swap 1025M $(docker ps --filter name=container --format "{{ .Names }}")

docker inspect $(docker ps --filter name=container --format "{{ .Names }}") | grep -i memory
            "Memory": 1073741824,
            "MemoryReservation": 536870912,
            "MemorySwap": 1074790400,
            "MemorySwappiness": 0,

docker exec -it $(docker ps --filter name=container --format "{{ .Names }}") bash
# inside the container: adjust oom proc file for all puma processes (even the started ones).
for pid in $(ps aux | grep puma | grep -v grep | perl -lane 'print "@F[1]"'); do echo 1000 > /proc/$pid/oom_score_adj; done
# Yes, perl is everywhere
```
{% endraw %}

And sure enough, I finally could do it! I saw puma restarting! 🎉
```
[1] - Worker 1 (PID: 206) booted in 0.06s, phase: 0
[1] - Worker 1 (PID: 301) booted in 0.06s, phase: 0
```

[docker-compose]: https://docs.docker.com/compose/
[compose-file-v3]: https://docs.docker.com/compose/compose-file/compose-file-v3/#resources
[kernel-proc-oom]: https://docs.kernel.org/filesystems/proc.html#proc-pid-oom-adj-proc-pid-oom-score-adj-adjust-the-oom-killer-score
