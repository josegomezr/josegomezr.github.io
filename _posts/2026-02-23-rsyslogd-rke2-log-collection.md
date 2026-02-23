---
title: Collecting Kube Logs (RKE2) with rsyslog
layout: post
tags: [blog, helm, k8s, rsyslogd, logs]
categories: [blog]
lang: en
extras: 
  mermaid: yes
---

# Da problem

You're running your RKE2 cluster you installed in _the most secure way_ :tm:
from the quickstart guide:

https://docs.rke2.io/install/quickstart

<small>_(for the sake of this post I'll assume you didn't `curl | sudo sh -` this
 situation and really installed it with an RPM or unpacking a tarball... tbh for
 my own sanity...)_</small>

Now you wanna check your deployment logs, you notice something odd and just at
that moment the deployment decided to spawn a new pod and deleted your pod!

You try getting your pod logs in
`/var/log/pods/<namespace>_<pod-name>_<pod-uuid>/<container>/<restart-num>.log`
(path structure not documented anywhere btw, so expect this to change in the
future) aaaaaand it's not there anymore :clown:

All of those logs... lost!

Or even worse, you're on "The Enterprise" and leadership asks you to integrate
your cluster with the SIEM, they offer you a rsyslog interface... but your
nodes only log to files...

If only... there was something that can take those log files, mix them into a
single directory structure... or even better, send it to the SIEM directly!

What was that rsyslog thing again?

# How does rsyslog works?

Rsyslog is basically an ETL, it ingest log messages (usually delimited by
newlines) from a variety of sources (system journal, kernel msgs, log files,
etc.), apply transformations and then outputs the messages to a destination
(file, no file, another rsyslog server, http request, etc.)

## Modules

Rsyslog has four module categories (closely tied to the ETL phases):

- **Input Modules:** They begin with `im`. All input modules can have a `ruleset` attached to it.
- **Parser Modules:** They begin with `pm`.
- **Message Modification Modules:** They begin with `mm`.
- **Output Modules:** They begin with `im`. All output modules may have a `template` attached to it.

## Rsyslog config syntax 101

Rsyslog has a complicated history (to my eyes anyway, not that it matters),
there are two syntaxes for configuration and they can live together which makes
it fun. One looks like perl written in Kanji (five characters build a whole sentence)
and the other one looks like lua without loops but braces.

Going through the internetz will land you in working snippets on _both_
syntaxes, but not so much of an explanation on how one syntax is equivalent to
the second.

Since the older syntax is deprecated, everything written here will be using the
new syntax (Ranierscript).

We'll need to know what the following statements:

- `input`: Declares a message input source.
- `action`: describes what to do with a message. Usually used with output modules.
- `template`: defines how to transform data before output.
  + `constant`: yep, straight forward
  + `property`: Reference to a property/variable.
    * `$!`: Root message JSON object variable.
    * `$.foo`: local `foo` variable.
    * `name`: rsyslog property.
- `ruleset`: The logic after input until output. They call actions and filters
  and can halt processing of the messages.

Properties

# The easy way out

To solve our magical problem _the easy way_, we just need to take out the log
files from the kube nodes into a remote syslog.

For that, the rsyslog pipeline is _very_ simple, it's just:

<pre class="mermaid">
flowchart LR
  LOGFILES[Log Files in /var/log/pod/*] -->|load| LOCAL_RSYSLOGD[Node-local RSYSLOGD]
  LOCAL_RSYSLOGD[Node-local RSYSLOGD] -->|extract| RSYSLOG[Remote SIEM]
</pre>

Let's implement it!

## Dependencies

- Install rsyslog

Just that, really.

## Configuration

On every kube node, let's create our snippet in: `/etc/rsyslog.d/10-rke2-logs.conf`.

```ruby
global(workDirectory="/var/spool/rsyslog" preserveFQDN="on")
module(load="imfile" PollingInterval="10")

input(
    type="imfile"
    file="/var/log/pods/*/*/*.log"
    tag="rke2-pods"
    facility="local0"
    ruleset="rke2-pod-logs"
)

ruleset(name="rke2-pod-logs") {
  action(
    type="omfwd"
    target="target-host"
    protocol="tcp"
    port="514"
    template="rke2-pod-logs"
    action.resumeRetryCount="-1"
    queue.type="LinkedList"
    queue.saveOnShutdown="on"
  )
}
```

Let's dissect this snippet.

### Input

```ruby
input(
  type="imfile"                  # Input Module: file
  file="/var/log/pods/*/*/*.log" # Location of the log files, imfile does not recurse over directories
  tag="rke2-pods"                # Tag for the server to identify our messages
  facility="local0"              # Facility: indicates the type of software that emitted the message, see [0]
  ruleset="rke2-pod-logs"        # ruleset: messages coming from this input must undergo through
                                 #          actions defined in the specified ruleset
)
```

Take all files from `/var/log/pods/*/*/*.log` mark them with `rke2-pods` tag

### Output

```ruby
action(
  type="omfwd"                 # Output Module: forward
  target="target-host"         # host
  protocol="tcp"               # proto
  port="514"                   # port
  action.resumeRetryCount="-1" # Retry count, -1 means infinity
  queue.type="LinkedList"      # configure a queue
  queue.saveOnShutdown="on"    # make it a durable queue
)
```

Send messages to `target-host:514` via tcp.

## Not so fast...

The easy way out keeps our SIEM needs happy, and our management happy... until
you yourself have to debug something...

The logs you're sending are great, but... your SCIEM only see:
```
2026-02-23T16:31:05+00:00 kubenode1.cluster.corp rke2-pods: 2026-02-23T16:31:05.304868587Z stderr F {"level":"info","ts":"2026-02-23T16:31:05Z","logger":"goldmane-controller","msg":"Reconciling Goldmane","Request.Namespace":"","Request.Name":"periodic-5m0s-reconcile-event"}
2026-02-23T16:31:05+00:00 kubenode2.cluster.corp rke2-pods: 2026-02-23T16:31:05.315608909Z stderr F {"level":"info","ts":"2026-02-23T16:31:05Z","logger":"controller_installation","msg":"Patching nftables mode","Request.Namespace":"","Request.Name":"periodic-5m0s-reconcile-event","nftablesMode":"Disabled"}
2026-01-28T12:30:33+00:00 kubenode3.cluster.corp rke2-pods: 2026-01-28T12:30:33.021378066Z stderr F I0128 12:30:33.021337       1 shared_informer.go:357] "Caches are synced" controller="client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
2026-02-23T16:30:29+00:00 kubenode3.cluster.corp rke2-pods: 2026-02-23T16:30:29.692875215Z stderr F I0223 16:30:29.692758       1 cidrallocator.go:277] updated ClusterIP allocator for Service CIDR 10.43.0.0/16
2026-02-23T16:33:58+00:00 kubenode2.cluster.corp rke2-pods: 2026-02-23T16:33:58.662208924Z stderr F {"level":"info","ts":"2026-02-23T16:33:58.662103Z","caller":"mvcc/kvstore_compaction.go:71","msg":"finished scheduled compaction","compact-revision":10936318,"took":"29.992945ms","hash":1435110330,"current-db-size-bytes":27926528,"current-db-size":"28 MB","current-db-size-in-use-bytes":13709312,"current-db-size-in-use":"14 MB"}
```

We know what machine emitted the log, but we've lost key information: what
container did this??

In the kube node, logs are saved in:
- `/var/log/pods/<namespace>_<pod-name>_<pod-uuid>/<container>/<restart-num>.log`

Which through file organization preserves the container information
(`ns`, `pod-name`, `pod-uuid`, `container`, `restart-num`) available.

But since we sent the content of the files, this is not longer available.

There are, of course, _multiple_ ways to solve this problem, we'll go with a
rather simple approach: embed the missing container information in the message.

# Putting in some more elbow grease

To put the missing information that lives in the file structure on the cluster
node files directly in each message we'll need to transform the messages prior
to sending.

## Configuration

```ruby
module(load="imfile" PollingInterval="10") # needs to be done just once
input(
    type="imfile"
    file="/var/log/pods/*/*/*.log"
    tag="rke2-pods"
    facility="local0"
    ruleset="rke2-pod-logs"
    # new vvvv
    addMetadata="on"
    # new ^^^^
)

# This can approximate more the RFC 3164 something something format
# [RFC 3164](https://datatracker.ietf.org/doc/html/rfc3164.html)
# [RFC 3339](https://datatracker.ietf.org/doc/html/rfc3339.html)
# Extended from: https://docs.rsyslog.com/doc/reference/templates/templates-examples.html#standard-template-for-forwarding-to-a-remote-host-rfc3164
template(name="rke2-pod-logs" type="list") {
    constant(value="<")
    property(name="pri")
    constant(value=">")
    property(name="timestamp" dateFormat="rfc3339")
    constant(value=" ")
    property(name="hostname")
    constant(value=" ")
    property(name="syslogtag" position.from="1" position.to="32")
    constant(value=" ")
    property(name="$!namespace")
    constant(value=" ")
    property(name="$!pod_name")
    constant(value=" ")
    property(name="$!container")
    property(name="msg" spIfNo1stSp="on")
    property(name="msg")
}

ruleset(name="rke2-pod-logs") {
  set $!fname = $!metadata!filename;
  set $!ns_pod = field($!metadata!filename, "/", 5);
  set $!namespace = field($!ns_pod, "_", 1);
  set $!pod_name = field($!ns_pod, "_", 2);
  set $!container = field($!metadata!filename, "/", 6);

  # Forward to another syslog
  action(
    type="omfwd"
    target="target-ip"
    protocol="tcp"
    port="514"
    action.resumeRetryCount="-1"
    queue.type="LinkedList"
    queue.saveOnShutdown="on"
    # new vvvv
    template="rke2-pod-logs"
    # new ^^^^
  )
  stop
}
```

Let's dissect this snippet.

### Input

```ruby
input(
    type="imfile"
    file="/var/log/pods/*/*/*.log"
    tag="rke2-pods"
    facility="local0"
    ruleset="rke2-pod-logs"
    # new vvvv
    addMetadata="on" # populate $!metadata variable with file info (name & offset)
    # new ^^^^
)
```

Take all files from `/var/log/pods/*/*/*.log` mark them with `rke2-pods` tag,
but now every message comes with a new variable `$!metadata` that holds some
extra information about the current file where the message was sourced.

This is gonna be crucial for the last stage.

### Transform

```ruby
# This can approximate more the RFC 3164 something something format
# [RFC 3164](https://datatracker.ietf.org/doc/html/rfc3164.html)
# [RFC 3339](https://datatracker.ietf.org/doc/html/rfc3339.html)
# Extended from: https://docs.rsyslog.com/doc/reference/templates/templates-examples.html#standard-template-for-forwarding-to-a-remote-host-rfc3164
template(name="rke2-pod-logs" type="list") {
    constant(value="<")
    property(name="pri")
    constant(value=">")
    property(name="timestamp" dateFormat="rfc3339")
    constant(value=" ")
    property(name="hostname")
    constant(value=" ")
    property(name="syslogtag" position.from="1" position.to="32")
    constant(value=" ")
    property(name="$!namespace")
    constant(value=" ")
    property(name="$!pod_name")
    constant(value=" ")
    property(name="$!container")
    property(name="msg" spIfNo1stSp="on")
    property(name="msg")
}
```

This is a template that once executed will return an RFC 3164 compliant message
line embedding the `$!namespace`, `$!pod_name` and `$!container` variables just
before the `msg` property.

The transform will be "called" (sort of speak) when the output action gets
processed.

### Output

```ruby
ruleset(name="rke2-pod-logs") {
  set $!fname = $!metadata!filename;
  set $!ns_pod = field($!metadata!filename, "/", 5);
  set $!namespace = field($!ns_pod, "_", 1);
  set $!pod_name = field($!ns_pod, "_", 2);
  set $!container = field($!metadata!filename, "/", 6);

  # Forward to another syslog
  action(
    type="omfwd"
    target="target-ip"
    protocol="tcp"
    port="514"
    action.resumeRetryCount="-1"
    queue.type="LinkedList"
    queue.saveOnShutdown="on"
    # new vvvv
    template="rke2-pod-logs"
    # new ^^^^
  )
  stop
}
```

Here we tell the output module to apply the transformation given by the template
`rke2-pod-logs` before forwarding the message to the remote syslog.


# Final result

Logs are generated on every node as:

```
# /var/log/pods/tigera-operator_tigera-operator-6977d6696f-477rm_4be391c4-5470-4c1d-b2d2-8486a33f1b2f/tigera-operator/3.log
2026-02-23T16:31:05.304868587Z stderr F {"level":"info","ts":"2026-02-23T16:31:05Z","logger":"goldmane-controller","msg":"Reconciling Goldmane","Request.Namespace":"","Request.Name":"periodic-5m0s-reconcile-event"}
2026-02-23T16:31:05.315608909Z stderr F {"level":"info","ts":"2026-02-23T16:31:05Z","logger":"controller_installation","msg":"Patching nftables mode","Request.Namespace":"","Request.Name":"periodic-5m0s-reconcile-event","nftablesMode":"Disabled"}
```

```
# /var/log/pods/kube-system_cloud-controller-manager-pasadena01_c3617a0545dfd93e783671b93825c565/cloud-controller-manager/0.log
2026-01-28T12:30:33.021378066Z stderr F I0128 12:30:33.021337       1 shared_informer.go:357] "Caches are synced" controller="client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
```

```
# /var/log/pods/kube-system_kube-apiserver-pasadena01_384eda8038ab144e166ab1c0fa0772e7/kube-apiserver/0.log
2026-02-23T16:30:29.692875215Z stderr F I0223 16:30:29.692758       1 cidrallocator.go:277] updated ClusterIP allocator for Service CIDR 10.43.0.0/16
```

Every node's rsyslog process transform the message before sending to:


```
tigera-operator tigera-operator-6977d6696f-477rm tigera-operator 2026-02-23T16:31:05.304868587Z stderr F {"level":"info","ts":"2026-02-23T16:31:05Z","logger":"goldmane-controller","msg":"Reconciling Goldmane","Request.Namespace":"","Request.Name":"periodic-5m0s-reconcile-event"}
tigera-operator tigera-operator-6977d6696f-477rm tigera-operator 2026-02-23T16:31:05.315608909Z stderr F {"level":"info","ts":"2026-02-23T16:31:05Z","logger":"controller_installation","msg":"Patching nftables mode","Request.Namespace":"","Request.Name":"periodic-5m0s-reconcile-event","nftablesMode":"Disabled"}
```

```
kube-system cloud-controller-manager-pasadena01 cloud-controller-manager 2026-01-28T12:30:33.021378066Z stderr F I0128 12:30:33.021337       1 shared_informer.go:357] "Caches are synced" controller="client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
```

```
kube-system kube-apiserver-node01 kube-apiserver 2026-02-23T16:30:29.692875215Z stderr F I0223 16:30:29.692758       1 cidrallocator.go:277] updated ClusterIP allocator for Service CIDR 10.43.0.0/16
```

Notice the:

```
<namespace> <pod-name> <container-name>
```

Prefixed at every message. Now the rsyslog server can interpret decide what to
do with the messages, we got ourselves a stable-enough solution to embed the
running container information in the log message for a remote syslog server.

## Plus: local file

If you just want to save the logs to a more durable/persistent place you can
take these in the output phase instead:

```ruby
template(name="rke2-pod-filename" type="list") {
    constant(value="/var/log/kube/")
    property(name="$!cluster")
    constant(value="/")
    property(name="$!namespace")
    constant(value="/")
    property(name="$!pod_name")
    constant(value="/")
    property(name="$!container")
    constant(value=".log")
}

ruleset(name="rke2-pod-logs") {
  set $!cluster   = field($msg, " ", 2);
  set $!namespace = field($msg, " ", 3);
  set $!pod_name  = field($msg, " ", 4);
  set $!container = field($msg, " ", 5);

  action(type="omfile" dynafile="rke2-pod-filename")
  stop
}
```

And it'll take nodes from:

`/var/log/pods/<namespace>_<pod-name>_<pod-uuid>/<container>/<restart-num>.log`

into a more human readable:

`/var/log/pods/<namespace>/<pod-name>/<container>.log`


[0]: https://en.wikipedia.org/wiki/Syslog#Message_components
