---
title: "Squashing a Ruby Bug"
layout: post
tags: [blog, ruby, gdb]
categories: [blog]
lang: 
---

The tale of how we found a very peculiar condition on ruby involving all the
greatest hits of software: Threads, Processes, & C Compilers.

<!-- more -->

It all begins on our efforts to move our platforms to run on latest and shiny
Ruby 3.

An _Azubi_ (apprentice) started looking at the problem at hand, seemed to be
very trivial, we didn't spot anything weird on [Release Notes for 3.3.0]
[rb-rn-330], or [3.3.1][rb-rn-331] (there's an [excellent summary with bugs and
references for the ruby 3.3 release][rb-ref-33] by [zverok][zverok]).

It was his very last task in our team, we were introducing him to how our code
gets tested and nothing better to know the CI pipeline that trying updating the
CI pipeline, right? :)

For context, all of our CI & Production systems is based around SUSE Linux
Enterprise (SLE). Nodes are running SLES [SLE Server] & Production Systems are
run on top of SLE Base Container Images [SLE-BCI]. Remember this, it's a key piece.

The apprentice was running the upgrade locally and everything looked smooth, then
went ahead to the CI nodes to install the new ruby version and start testing the
application.

In the middle of enabling tests, he ran into very weird errors with
`bundle install` commands:

```
/usr/lib64/ruby/site_ruby/3.3.0/rubygems/ext/builder.rb:93: [BUG] Segmentation fault at 0x0000000000000014
ruby 3.3.0 (2023-12-25 revision 5124f9ac75) [x86_64-linux-gnu]

-- Control frame information -----------------------------------------------
c:0024 p:---- s:0174 e:000173 CFUNC  :gets
c:0023 p:0034 s:0170 e:000169 BLOCK  /usr/lib64/ruby/site_ruby/3.3.0/rubygems/ext/builder.rb:93
c:0022 p:0054 s:0163 e:000162 METHOD /usr/lib64/ruby/3.3.0/open3.rb:540
c:0021 p:0094 s:0152 e:000151 METHOD /usr/lib64/ruby/3.3.0/open3.rb:522
c:0020 p:0124 s:0141 e:000140 METHOD /usr/lib64/ruby/site_ruby/3.3.0/rubygems/ext/builder.rb:91
c:0019 p:0089 s:0125 e:000124 METHOD /usr/lib64/ruby/site_ruby/3.3.0/rubygems/ext/ext_conf_builder.rb:28
c:0018 p:0072 s:0109 e:000108 METHOD /usr/lib64/ruby/site_ruby/3.3.0/rubygems/ext/builder.rb:193
c:0017 p:0017 s:0098 e:000097 BLOCK  /usr/lib64/ruby/site_ruby/3.3.0/rubygems/ext/builder.rb:227 [FINISH]
c:0016 p:---- s:0094 e:000093 CFUNC  :each
c:0015 p:0084 s:0090 e:000089 METHOD /usr/lib64/ruby/site_ruby/3.3.0/rubygems/ext/builder.rb:224
c:0014 p:0016 s:0085 e:000084 METHOD /usr/lib64/ruby/site_ruby/3.3.0/rubygems/installer.rb:852
c:0013 p:0034 s:0080 e:000079 METHOD /home/automat/.bundle-for-rb-33/gems/bundler-2.4.10/lib/bundler/rubygems_gem_installer.rb:72
c:0012 p:0060 s:0073 e:000072 METHOD /home/automat/.bundle-for-rb-33/gems/bundler-2.4.10/lib/bundler/rubygems_gem_installer.rb:28
c:0011 p:0344 s:0069 e:000068 METHOD /home/automat/.bundle-for-rb-33/gems/bundler-2.4.10/lib/bundler/source/rubygems.rb:200
c:0010 p:0025 s:0052 e:000051 METHOD /home/automat/.bundle-for-rb-33/gems/bundler-2.4.10/lib/bundler/installer/gem_installer.rb:54
c:0009 p:0003 s:0048 e:000047 METHOD /home/automat/.bundle-for-rb-33/gems/bundler-2.4.10/lib/bundler/installer/gem_installer.rb:16
c:0008 p:0034 s:0042 e:000041 METHOD /home/automat/.bundle-for-rb-33/gems/bundler-2.4.10/lib/bundler/installer/parallel_installer.rb:156
c:0007 p:0007 s:0033 e:000032 BLOCK  /home/automat/.bundle-for-rb-33/gems/bundler-2.4.10/lib/bundler/installer/parallel_installer.rb:147
c:0006 p:0009 s:0028 e:000027 METHOD /home/automat/.bundle-for-rb-33/gems/bundler-2.4.10/lib/bundler/worker.rb:62
c:0005 p:0030 s:0021 e:000019 BLOCK  /home/automat/.bundle-for-rb-33/gems/bundler-2.4.10/lib/bundler/worker.rb:57
c:0004 p:0018 s:0016 e:000015 METHOD <internal:kernel>:187
c:0003 p:0004 s:0011 e:000010 METHOD /home/automat/.bundle-for-rb-33/gems/bundler-2.4.10/lib/bundler/worker.rb:54
c:0002 p:0005 s:0006 e:000005 BLOCK  /home/automat/.bundle-for-rb-33/gems/bundler-2.4.10/lib/bundler/worker.rb:90 [FINISH]
c:0001 p:---- s:0003 e:000002 DUMMY  [FINISH]

-- Ruby level backtrace information ----------------------------------------
```

Never seen before an error like that! But out of ignorance we noticed that our
Gemfile*.log had references to ruby 3.2.x, and we were trying to operate them
with ruby 3.3.3. My initial thought was: "There must be an ABI incompatibility
here", it made sense to me then to have proper references there and retry.

So we did, updated the `Gemfile.lock` to:

```diff
RUBY VERSION
-   ruby 3.2.2
+   ruby 3.3.0p0
```

aaaaaaaaand "it worked"!✨... but Azubi ended his time with us, so we archived
the task for later.

<small>**Narrator:** _It actually never worked. Azubi and yours trully were
fooled..._</small>

We didn't pay much attention to the backtrace, I was particularly busy with
features, and we all agreed that the blocker (CI failing to install) was not
longer a problem, so we can continue later.

Until a couple of weeks ago when a colleague takes the task again and it. He hits
constantly and consistently that very segfault I saw when working with the
apprentice.

He spent a weeks time pretty much picaxing this until it was not worth it anymore.
The only progress he achieved was noticing that `bundle install` works if invoked
with `--jobs 1`.

This heavily hinted forking/threading was involved, none of us are low level C
developers, so we'all agreed that we'll wait for an update that may fix this.

And it may end there right? just wait for an update. But that wouldn't be me if
I didn't try to fix it.

I couldn't comprehend why if the environment is pretty much the same ruby
wouldn't work. Even in containers it would break segfault...

Segfaults are something I pretty much have not experience, maybe once or twice
in the university but as a PHP-first, segfaults are *quite* rare...

So I pulled the openSUSE variant of ruby3.3 from [devel:languages:ruby][d-l-r],
tried exactly what my colleague has tried and... it works?? for real???

No segfault, just works™. Colleague suggests:

> Why not using the openSUSE image instead?

And that's the moment where I have a strike of _pure genius_. One of the SLE-BCI
leads is really persistent on pretty much *EVERYTHING* to use SLE-BCI in the
spirit of _"we should try our own dog food"_.

Since we do use BCI, and we are having this very obscure bug, I just said:

> let's ask him! He must know something we dont!

I send my backtrace, this time the C-level informaton

```
-- C level backtrace information -------------------------------------------
/usr/lib64/libruby3.3.so.3.3(rb_print_backtrace+0x15) [0x7f1ebc87d8a5]
/usr/lib64/libruby3.3.so.3.3(rb_vm_bugreport+0x375) [0x7f1ebc87dc35]
/usr/lib64/libruby3.3.so.3.3(rb_bug_for_fatal_signal+0xff) [0x7f1ebc6e28cf]
/usr/lib64/libruby3.3.so.3.3(sigsegv+0x4d) [0x7f1ebc7f1e6d]
/lib64/libpthread.so.0(__restore_rt+0x0) [0x7f1ebcc76910]
/usr/lib64/libruby3.3.so.3.3(rb_enc_associate_index+0x80) [0x7f1ebc6cc790]
/usr/lib64/libruby3.3.so.3.3(rb_io_getline_0+0x979) [0x7f1ebc71f169]
/usr/lib64/libruby3.3.so.3.3(rb_io_getline_1+0x43) [0x7f1ebc71f233]
/usr/lib64/libruby3.3.so.3.3(rb_io_getline+0x3d) [0x7f1ebc71f2dd]
/usr/lib64/libruby3.3.so.3.3(rb_io_gets_m+0x6) [0x7f1ebc71f306]
/usr/lib64/libruby3.3.so.3.3(vm_call_cfunc_with_frame_+0xcd) [0x7f1ebc85727d]
/usr/lib64/libruby3.3.so.3.3(vm_exec_core+0x1ac) [0x7f1ebc86556c]
/usr/lib64/libruby3.3.so.3.3(rb_vm_exec+0x29d) [0x7f1ebc86b09d]
/usr/lib64/libruby3.3.so.3.3(invoke_block_from_c_bh+0x222) [0x7f1ebc86f3d2]
/usr/lib64/libruby3.3.so.3.3(rb_yield+0x78) [0x7f1ebc86fb68]
/usr/lib64/libruby3.3.so.3.3(rb_ary_each+0x3c) [0x7f1ebc64a4fc]
/usr/lib64/libruby3.3.so.3.3(vm_call_cfunc_with_frame_+0xcd) [0x7f1ebc85727d]
/usr/lib64/libruby3.3.so.3.3(vm_sendish+0xa7) [0x7f1ebc862d67]
/usr/lib64/libruby3.3.so.3.3(vm_exec_core+0xab8) [0x7f1ebc865e78]
/usr/lib64/libruby3.3.so.3.3(rb_vm_exec+0xf9e) [0x7f1ebc86bd9e]
/usr/lib64/libruby3.3.so.3.3(vm_invoke_proc+0x21b) [0x7f1ebc8702eb]
/usr/lib64/libruby3.3.so.3.3(rb_vm_invoke_proc+0x3d) [0x7f1ebc8704ad]
/usr/lib64/libruby3.3.so.3.3(thread_do_start_proc+0x195) [0x7f1ebc82cff5]
/usr/lib64/libruby3.3.so.3.3(thread_start_func_2+0x65a) [0x7f1ebc82d8aa]
/usr/lib64/libruby3.3.so.3.3(nt_start+0x1a7) [0x7f1ebc82ddb7]
/lib64/libpthread.so.0(start_thread+0xdc) [0x7f1ebcc6a6ea]
/lib64/libc.so.6(clone+0x41) [0x7f1ebbb2150f]
```

And link to the package to reproduce. It took him short of a minute until I got back
a:

> I already got a crash, it's a 0 pointer dereference in libpthread
>
> ```
> (gdb) bt
> #0  0x00007fce1dccc750 in rb_enc_associate_index (obj=obj@entry=140522671372720, idx=2) at encoding.c:998
> #1  0x00007fce1dccc7c7 in rb_enc_associate (obj=obj@entry=140522671372720, enc=<optimized out>) at encoding.c:1009
> #2  0x00007fce1dd1f159 in io_enc_str (fptr=0x7fcdbc203470, str=140522671372720) at io.c:3119
> #3  rb_io_getline_fast (chomp=0, enc=0x55560e03bde0, fptr=0x7fcdbc203470) at io.c:4003
> #4  rb_io_getline_0 (rs=rs@entry=140522807076160, limit=limit@entry=-1, chomp=chomp@entry=0, fptr=fptr@entry=0x7fcdbc203470) at io.c:4114
> #5  0x00007fce1dd1f223 in rb_io_getline_1 (rs=140522807076160, limit=-1, chomp=0, io=io@entry=140522634996640) at io.c:4209
> #6  0x00007fce1dd1f2cd in rb_io_getline (argc=<optimized out>, argv=<optimized out>, io=140522634996640) at io.c:4229
> #7  0x00007fce1dd1f2f6 in rb_io_gets_m (argc=<optimized out>, argv=<optimized out>, io=<optimized out>) at io.c:4326
> #8  0x00007fce1de57e2d in vm_call_cfunc_with_frame_ (ec=0x555612a3c280, reg_cfp=0x555612d1deb8, calling=<optimized out>, argc=0, argv=0x555612c1e918, stack_bottom=0x555612c1e910)
>     at vm_insnhelper.c:3490
> #9  0x00007fce1de6d5c6 in vm_call_method_each_type (ec=ec@entry=0x555612a3c280, cfp=cfp@entry=0x555612d1deb8, calling=0x7fcdf1ad54a0) at vm_insnhelper.c:4417
> #10 0x00007fce1de6dfa6 in vm_call_method (ec=0x555612a3c280, cfp=0x555612d1deb8, calling=<optimized out>) at vm_insnhelper.c:4569
> #11 0x00007fce1de661cf in vm_sendish (method_explorer=<optimized out>, block_handler=<optimized out>, cd=<optimized out>, reg_cfp=<optimized out>, ec=<optimized out>)
>     at vm_insnhelper.c:5581
> #12 vm_exec_core (ec=0x555612a3c280) at insns.def:834
> #13 0x00007fce1de6b9bd in rb_vm_exec (ec=ec@entry=0x555612a3c280) at vm.c:2486
> #14 0x00007fce1de6fc92 in invoke_block (captured=0x555612d1e090, opt_pc=<optimized out>, type=572653569, cref=0x0, self=140522662015920, iseq=0x7fcdf9dae2f0, ec=0x555612a3c280)
>     at vm.c:1509
> #15 invoke_iseq_block_from_c (me=0x0, is_lambda=0, cref=0x0, passed_block_handler=<optimized out>, kw_splat=<optimized out>, argv=<optimized out>, argc=<optimized out>, 
>     self=140522662015920, captured=<optimized out>, ec=0x555612a3c280) at vm.c:1579
> #16 invoke_block_from_c_bh (force_blockarg=<optimized out>, is_lambda=<optimized out>, cref=<optimized out>, passed_block_handler=<optimized out>, kw_splat=<optimized out>, 
>     argv=<optimized out>, argc=<optimized out>, block_handler=<optimized out>, ec=<optimized out>) at vm.c:1597
> #17 vm_yield_with_cref (is_lambda=0, cref=0x0, kw_splat=0, argv=<optimized out>, argc=1, ec=<optimized out>) at vm.c:1634
> #18 vm_yield (kw_splat=0, argv=<optimized out>, argc=1, ec=<optimized out>) at vm.c:1642
> #19 rb_yield_0 (argv=<optimized out>, argc=1) at vm_eval.c:1366
> #20 catch_i (tag=<optimized out>, _=<optimized out>, argc=<optimized out>, argv=<optimized out>, blockarg=<optimized out>) at vm_eval.c:2287
> #21 0x00007fce1de70308 in vm_yield_with_cref (is_lambda=0, cref=0x0, kw_splat=0, argv=0x7fcdf1ad57c8, argc=1, ec=<optimized out>) at vm.c:1634
> #22 vm_yield (kw_splat=0, argv=0x7fcdf1ad57c8, argc=1, ec=<optimized out>) at vm.c:1642
> #23 rb_yield_0 (argv=0x7fcdf1ad57c8, argc=1) at vm_eval.c:1366
> #24 rb_yield (val=<optimized out>) at vm_eval.c:1382
> #25 0x00007fce1dc4a55c in rb_ary_each (ary=140522635552080) at array.c:2538
> #26 0x00007fce1de57e2d in vm_call_cfunc_with_frame_ (ec=0x555612a3c280, reg_cfp=0x555612d1e078, calling=<optimized out>, argc=0, argv=0x555612c1e698, stack_bottom=0x555612c1e690)
>     at vm_insnhelper.c:3490
> #27 0x00007fce1de639c7 in vm_sendish (ec=0x555612a3c280, reg_cfp=0x555612d1e078, cd=<optimized out>, block_handler=<optimized out>, method_explorer=<optimized out>)
>     at vm_insnhelper.c:5585
> #28 0x00007fce1de66aec in vm_exec_core (ec=0x555612a3c280) at insns.def:814
> #29 0x00007fce1de6c6be in vm_exec_loop (tag=0x7fcdf1ad5b60, result=36, state=<optimized out>, ec=<optimized out>) at vm.c:2513
> #30 rb_vm_exec (ec=ec@entry=0x555612a3c280) at vm.c:2492
> #31 0x00007fce1de70a8b in invoke_block (captured=0x555612a3c030, opt_pc=<optimized out>, type=572653569, cref=0x0, self=140522658101280, iseq=0x7fce1d91a0d0, ec=0x555612a3c280)
>     at vm.c:1509
> #32 invoke_iseq_block_from_c (me=0x0, is_lambda=<optimized out>, cref=0x0, passed_block_handler=0, kw_splat=0, argv=<optimized out>, argc=<optimized out>, self=140522658101280, 
>     captured=0x555612a3c030, ec=0x555612a3c280) at vm.c:1579
> #33 invoke_block_from_c_proc (me=0x0, is_lambda=<optimized out>, passed_block_handler=0, kw_splat=<optimized out>, argv=0x0, argc=<optimized out>, self=140522658101280, 
>     proc=0x555612a3c030, ec=0x555612a3c280) at vm.c:1677
> #34 vm_invoke_proc (ec=0x555612a3c280, proc=proc@entry=0x555612a3c030, self=140522658101280, argc=argc@entry=0, argv=<optimized out>, kw_splat=<optimized out>, passed_block_handler=0)
>     at vm.c:1707
> #35 0x00007fce1de70c4d in rb_vm_invoke_proc (ec=<optimized out>, proc=proc@entry=0x555612a3c030, argc=argc@entry=0, argv=<optimized out>, kw_splat=<optimized out>, 
>     passed_block_handler=passed_block_handler@entry=0) at vm.c:1728
> #36 0x00007fce1de2da85 in thread_do_start_proc (th=th@entry=0x555612a3c060) at thread.c:592
> #37 0x00007fce1de2e33a in thread_do_start (th=0x555612a3c060) at thread.c:611
> #38 thread_start_func_2 (th=th@entry=0x555612a3c060, stack_start=<optimized out>) at thread.c:667
> #39 0x00007fce1de2e847 in call_thread_start_func_2 (th=0x555612a3c060) at thread_pthread.c:2186
> #40 nt_start (ptr=0x555612a3c4c0) at thread_pthread.c:2231
> #41 0x00007fce1da606ea in start_thread (arg=0x7fcdf1ad6700) at pthread_create.c:477
> #42 0x00007fce1d52158f in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:95
> ```
>
> I can share my screen
>
> Me: I'm running to your office!

We had a look together at the backtrace and try to find where in the ruby code could
the problem be.

After ca. two hours that involved reading looooooong long C code files from the ruby
source, and a great detailed information about compiler optimizations, GC's, debug
symbols we didn't got much farther, other than knowing where the problem rises.

We know is somewhere in `io.c` & `encoding.c`. But by this time I had a list of
packages to install, a fancy `gdb` console to throw commands at, not that I knew any
of them 😂.

Running more times I got a slightly different backtrace:

```
Thread 25 "Parallel Instal" received signal SIGABRT, Aborted.
[Switching to Thread 0x7fffd456e700 (LWP 390)]
__GI_raise (sig=6) at ../sysdeps/unix/sysv/linux/raise.c:51
(gdb)  bt
#0  __GI_raise (sig=6) at ../sysdeps/unix/sysv/linux/raise.c:51
#1  0x00007ffff75093e5 in __GI_abort () at abort.c:79
#2  0x00007ffff754dc87 in __libc_message (action=do_abort, fmt=0x7ffff7677138 "%s\n") at ../sysdeps/posix/libc_fatal.c:155
#3  0x00007ffff7555d2a in malloc_printerr (str=0x7ffff7674e0e "free(): invalid pointer") at malloc.c:5347
#4  0x00007ffff75577d4 in _int_free (av=<optimized out>, p=<optimized out>, have_lock=0) at malloc.c:4173
#5  0x00007ffff7c741f2 in objspace_xfree.isra.146.part () from /usr/lib64/libruby3.3.so.3.3
#6  0x00007ffff7d82b4d in rb_str_resize () from /usr/lib64/libruby3.3.so.3.3
#7  0x00007ffff7c8e02e in rb_io_getline_0 () from /usr/lib64/libruby3.3.so.3.3
#8  0x00007ffff7c8e223 in rb_io_getline_1 () from /usr/lib64/libruby3.3.so.3.3
#9  0x00007ffff7c8e2cd in rb_io_getline () from /usr/lib64/libruby3.3.so.3.3
#10 0x00007ffff7c8e2f6 in rb_io_gets_m () from /usr/lib64/libruby3.3.so.3.3
# ...
```

It originated in the same spot, but the messages were different. Distinguished
engineer reads this and says:

> That's memory corruption! Double free
> Exactly in that function that we looked at
> So something has already freed that `ptr`. Garbage collector?
> Can you get that with valgrind? It would give us the earlier free() backtrace

`valgrind` didn't produce anything useful for me [to be completely honest, no clue
how to read the logs, and the command was actually working, SLOW AS FUCK, but
managed to finish].

The day ended with pretty much a condensed master class over:

- How production-grade packages are optimized and stripped _[and thus not include
enough information to hook debugging tools]_
- Debug-info packages [that missing information]
- Lots of `gdb` output and explanations:
  * Memory Layout of Structs
  * Pointers to Stack
  * GC 101
- One run of `valgrind` that was *very* slow and verbose but somehow succeeded?
- We both having a crash course on ruby internal structures reading source code
trying to make sense of the stack trace.

We both continued our ways trying to make sense of the information.

Next day I start my work day with a DM:

> hey, I might have a fix.
>
> The earlier backtrace gave a hint. we had a double free during `rb_str_resize`. This
> is because it does a save of the string `ptr`, then resizes. so I think we're
> particularly unlucky that this very resize triggers a garbage collection run
> that cleans up our VALUE because the GC cannot find a reference on the stack.
>
> so I noticed this morning that in many other places they use a volatile macro to
> ensure there is actually a pointer on the stack
>
> I did 10 rounds without a single problem. I declare it fixed until you prove
> me otherwise

The patch looked like:

```diff
--- ruby-3.3.1.orig/io.c
+++ ruby-3.3.1/io.c
@@ -3980,6 +3980,7 @@ rb_io_getline_fast(rb_io_t *fptr, rb_enc
                 fptr->rbuf.len -= pending;
             }
             else {
+                StringValue(str);
                 rb_str_resize(str, len + pending - chomplen);
                 read_buffered_data(RSTRING_PTR(str)+len, pending - chomplen, fptr);
                 fptr->rbuf.off += chomplen;
```

Armed with a patch that solves this, I proceeded to test it all, and _boi it worked, it worked well!_

I ask him _"should we upstream this?"_

> of course, it's a real bug
> and of course this could still be entirely wrong

Then it's settled, we have a potential solution to this! I proceed to replicate
the build environment and the binaries.

I inspected the implementation of `StringValue`, it did more things with the actual
value besides having the `volatile` signal/symbol/notation (?), something didn't feel
quite right.

A couple of hours of RPM wrangling another colleague examines the situation, and gets
to pretty much the same conclusion BUT finds out about [the docs for ruby extensions][rb-ext-gc_guard]
showcasing an interesting paragraph:

> The following example illustrates the use of RB_GC_GUARD to ensure the contents of Z remain valid while the second invocation of `rb_str_new_cstr` is running.
> 
> ```ruby
> VALUE s, w;
> const char *sptr;
> 
> s = rb_str_new_cstr("hello world!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!");
> sptr = RSTRING_PTR(s);
> w = rb_str_new_cstr(sptr + 6); /* Possible GC invocation */
> 
> RB_GC_GUARD(s); /* ensure s (and thus sptr) do not get GC-ed */
> ```
> In the above example, `RB_GC_GUARD` must be placed after the last use of sptr. Placing `RB_GC_GUARD` before dereferencing sptr would be of no use. `RB_GC_GUARD` is only effective on the VALUE data type, not converted C data types.
> 
> `RB_GC_GUARD` would not be necessary at all in the above example if non-inlined function calls are made on the ‘s’ VALUE after sptr is dereferenced. Thus, in the above example, calling any un-inlined function on ‘s’ such as:
> ```
> rb_str_modify(s);
> ```
> Will ensure ‘s’ stays on the stack or register to prevent a GC invocation from prematurely freeing it.

This whole section, made absolutely no sense in my naive brain, how you signal the GC _"post fact"_??? What kind of sorcery is this?? [see the end]

I tried it out nevertheless, the implementation of `RB_GC_GUARD` was an empty assembler instruction
it felt like a more proper solution:

```diff
--- ruby3.3.orig/ruby-3.3.1/io.c
+++ ruby3.3/ruby-3.3.1/io.c
@@ -4004,6 +4004,7 @@ rb_io_getline_fast(rb_io_t *fptr, rb_enc
     ENC_CODERANGE_SET(str, cr);
     fptr->lineno++;
 
+    RB_GC_GUARD(str);
     return str;
 }
```

aaaaand it worked, which made me understand even less! How? Why? what kind of magic is happening????

In any case, I had a better understanding of the problem:

- `rb_io_getline_fast` in `io.c` falls in a race condition that Garbage Collects in an unexpected
moment
- `RB_GC_GUARD` macro enforces this `volatile` thing to keep pointers visible in the stack.

Also:

- Got a reproducer image in [home:josegomezr:branches:ruby:containers/base-ruby33][obs-ruby-containers-branch]
- Got a patched ruby package [home:josegomezr:branches:ruby/ruby3.3][obs-ruby-patched]

Time to bug report this! Which ends up with [bug#20493][rb-bug-20493]! We got an official report!

Next day `kjtsanaktsidis` took it and shed the ultimate light on the bug. It's *way* deeper than we were prepared for.

Quoting from the bug:

> The crash is happening because:
> 
> * One thread is calling `rb_io_getline_fast`
> * The VALUE variable `str` is saved in register `%r15`
> * `rb_io_getline_fast` calls a chain of functions that eventually yields the GVL and performs a syscall
> * None of these other functions spill `%r15` to the stack. It's present only in `%r15` for the duration of the blocking system call.
> * Another Ruby thread runs and performs a GC
> * As part of that, it marks `ec->machine` for all other threads. This includes a representation of that thread's register state in `jmpbuf`.
> * However, the value for `str` seems not to be in `jmpbuf` when it performs GC marking.

That just confirms that hunch about threading/forking involved! `kjtsanaktsidis` went over with a remarkable explanation of all the moving pieces behavior. Explanation that I couldn't fully grasp [I really recommend reading it, it's just mind blowingly good!]

TL;DR: The fix I suggested would've only "fixed" the segfault on a reduced scope [IO#read* methods] but wouldn't solve it across the board because the root cause lies on how the context is saved/restored when yielding execution to C-land functions.

<small>yes, I yanked my own question out of the bug, sue me.</small>

There's a PR ([ruby/ruby#10795][ruby/ruby#10795]) that fully addresses this problem, in record time!

As for me, I think I had enough C/Linux/Threads/Forks/Valgrind/GCC/RPM/OBS/Ruby for today 😅

ps.: `RB_GC_GUARD` works because the assembly instruction includes a "memory lock" that forces the pointer to remain visible in the stack AT THE POINT WHERE THE MACRO IS PLACED, which explains why must be in the last usage of a variable. Now I can sleep in peace.

[ruby/ruby#10795]: https://github.com/ruby/ruby/pull/10795
[rb-ext-gc_guard]: https://github.com/ruby/ruby/blob/8acec5b6e8a8ed102885da467d47c30657a9a73f/doc/extension.rdoc#label-Appendix+E.+RB_GC_GUARD+to+protect+from+premature+GC
[obs-ruby-containers-branch]: https://build.opensuse.org/package/show/home:josegomezr:branches:ruby:containers/base-ruby33
[obs-ruby-patched]: https://build.opensuse.org/package/show/home:josegomezr:branches:ruby/ruby3.3
[d-l-r]: https://build.opensuse.org/project/show/devel:languages:ruby
[rb-rn-330]: https://www.ruby-lang.org/en/news/2023/12/25/ruby-3-3-0-released/
[rb-rn-331]: https://www.ruby-lang.org/en/news/2024/04/23/ruby-3-3-1-released/
[rb-ref-33]: https://rubyreferences.github.io/rubychanges/3.3.html
[zverok]: https://zverok.space/
[rb-bug-20493]: https://bugs.ruby-lang.org/issues/20493

