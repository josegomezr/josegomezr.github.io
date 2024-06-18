---
title: Instrumenting `rake` tasks... with `bash`
layout: post
tags: [blog, helm, k8s]
categories: [blog]
lang: en
extras: 
  mermaid: yes
---

Because ruby in this case: just doesn't cut it. And (ab-)using `at_exit` is not an option.

<!-- more -->

It all begins with a noble goal: Collecting Metrics after a rake task runs. What I want is:

<pre class="mermaid">
flowchart TD
  TASK_ONE[First Task] -->|then| TASK_TWO[Second Task]
  TASK_TWO[Second Task] -->|then| PUSH_PROM[Push to Prometheus]
  PUSH_PROM[Push to Prometheus] -->|then| END[FINISH]
</pre>

You might see it from afar and say: _izy-pizy_ just run your task and your collector at last and call it a day!

```bash
bin/rake my:task prometheus:push
```

Until you realize that if `my:task` raises... no metrics will be sent. Our shiny solution only works on the Goldilocks scenario. 

On the doomsday scenario what ends up happening is something closer to:

<pre class="mermaid">
flowchart TD
  TASK_ONE[First Task] -->|then| TASK_TWO[Second Task]
  TASK_TWO[Second Task] -->|raised and| END[FINISH]
  PUSH_PROM[Push to Prometheus]
</pre>

<small>📝: Note the fact that _Push to Prometheus_ is skipped</small>

And that _may_ be good enough, the lack of metrics is also a reasonable sign to
raise an alarm; Fair enough... not for me!

I want consistency! I want to get anything that may have been recorded during
the failed execution!


I couldn't find a "readable technique" to achieve that, the _best_ so far was
from [a big stick by @ozydingo][wrapping-rake]:

1. Prepend a `setup` lambda into the task actions: it'll mark the beginning of
the task.

2. Create a `log_task_stats` [or for our own matter: `prometheus:push` task from
now on]: this will register an `#at_exit` to mark the exit of a task and emit
the metrics.

3. Enhance all but `environment` and `prometheus:push` to call `prometheus:push`
afterwards AND execute a last block that marks the end of the task.

It *is* convoluted, because rake was not designed for this. I particularly
didn't feel quite confortable by (ab-)using the `at_exit` hook [side note:
[I totally agree with this Arkency blog post][abusing-at-exit]]

Trust me, I tried to innovate over `@ozydingo`'s approach and didn't get to a
prettier place, this has been "the simplest" so far:

1. Have a _needle_ task

2. Extend `Rake::Task` to have an `ensure` block (so I get to run code even if
the task itself raises) and test if the current task includes my desired
`needle`.

Something along the lines of:

```ruby
# In your Rakefile
namespace :prometheus do
  desc 'Mark a task to forward metrics when it finishes'
  task :push do
    # no-op. we just need a needle
  end
end

if ENV['PROMETHEUS_ENABLED'].present?
  mod = Module.new do
    def invoke(...)
      super(...)
    ensure
      send_metrics_to_prometheus! if self.prerequisites.include?('prometheus:push')
    end  
  end
  
  mod.define_method(:send_metrics_to_prometheus!) do
    # whatever means necessary to push your metrics
    # it's key that this is not inside the module block, here you have
    # access to things outside the Rake::Task context.
  end
  Rake::Task.prepend mod
end
```

Still ugly, and probably write-only ™️ code.

After hours down into `ruby/rake` and more ever creative solutions I just sank
for a reasonable alternative... Just use bash ™️!

I mean it can't be simpler, wrap rake into a small bash script like:

```bash
# in your proj's: bin/instrumented-rake
#!/bin/bash
bin/rake $@ prometheus:push || (bin/rake prometheus:push; exit 1)
```

Or a more verbose version:

```bash
# in your proj's: bin/instrumented-rake
#!/bin/bash
bin/rake $@ prometheus:push

if [[ $? -ne 0 ]]; then
  # If previous command did not exit cleanly, 100% chance that our metrics weren't pushed.
  # And make sure to exit with a non-zero status too.

  bin/rake prometheus:push
  exit 1
fi
```

Or if you really care about precise exit codes:

```bash
# in your proj's: bin/instrumented-rake
#!/bin/bash
bin/rake $@ prometheus:push
EXIT_CODE=$?

if [[ $EXIT_CODE -ne 0 ]]; then
  # If previous command did not exit cleanly, 100% chance that our metrics weren't pushed.
  # And make sure to exit with a non-zero status too.

  bin/rake prometheus:push
fi

exit $EXIT_CODE
```

straight forward, no monkeypatches, just a separate entrypoint. And sure enough, it behaves exactly like I want:

<pre class="mermaid">
flowchart TD
  TASK_ONE[First Task] -->|then| TASK_TWO[Second Task]
  TASK_TWO[Second Task] -->|then| PUSH_PROM[Push to Prometheus]
  PUSH_PROM[Push to Prometheus] -->|then| END[FINISH]
</pre>

<small>_I don't love either, but between timing `at_exit` hooks and monkey patching `Rake::Task` and having an extra explicit entrypoint I'm just picking the lesser evil_</small>

[wrapping-rake]: http://www.abigstick.com/2021/03/21/wrapping-rake.html
[abusing-at-exit]: https://blog.arkency.com/2013/06/are-we-abusing-at-exit/
