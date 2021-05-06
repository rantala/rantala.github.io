---
layout: post
title:  "Intel HD Audio Linux Perf Story"
date:   2020-03-22 12:00:00 +0200
description: "Intel HD Audio Linux Perf Story"
---

While waiting for a Webex video meeting to begin, the fans on my laptop started
spinning, so fired up `top` to see how much CPU these kind of browser based
video meetings are burning:
```
Tasks: 373 total,   1 running, 372 sleeping,   0 stopped,   0 zombie
%Cpu(s): 10.2 us,  2.6 sy,  0.0 ni, 85.4 id,  0.1 wa,  1.4 hi,  0.3 si,  0.0 st
MiB Mem :  23966.1 total,   9074.0 free,   3827.5 used,  11064.7 buff/cache
MiB Swap:   8032.0 total,   8032.0 free,      0.0 used.  19462.6 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 7165 tommi     20   0 9278320 202256 101308 S  84.4   0.8   4:10.03 chrome
 2665 tommi     20   0 2240432  15968  11432 S  14.0   0.1   8:17.29 pulseaudio
 7221 tommi     20   0 1025080  58908  48300 S   4.7   0.2   0:18.00 chrome
26477 tommi     20   0 1433436 510428 360048 S   3.0   2.1   3:16.77 chrome
26540 tommi     20   0  790292 288040 113884 S   1.3   1.2   3:51.86 chrome
 2315 tommi     20   0  197452  70788  51336 S   0.7   0.3  15:48.60 Xorg
```

Quite high CPU usage of `chrome` and `pulseaudio`, considering that the meeting
is still fully idle and all microphones muted!

## chrome

It's the Google Chrome browser, version 80.

Let's look at `chrome` (PID 7165) per-thread CPU usages, 10 second average:
```
$ pidstat -t -p 7165 1 10
Linux 5.5.8-100.fc30.x86_64     03/15/20     _x86_64_    (8 CPU)
[...]

Average:   UID  TGID  TID    %usr %system    %CPU   CPU  Command
Average:  1000  7165    -   71.70   12.30   84.00     -  chrome
Average:  1000     - 7165    4.80    0.80    5.60     -  |__ chrome
Average:  1000     - 7166    0.90    0.50    1.40     -  |__ ThreadPoolServi
Average:  1000     - 7167    1.90    0.30    2.20     -  |__ ThreadPoolForeg
Average:  1000     - 7168    0.60    0.30    0.90     -  |__ Chrome_ChildIOT
Average:  1000     - 7171    3.10    0.60    3.70     -  |__ Compositor
[...]
Average:  1000     - 7274    0.30    0.30    0.60     -  |__ PacerThread
Average:  1000     - 7275   38.60    0.30   38.90     -  |__ AudioInputDevic
Average:  1000     - 7276    0.40    0.20    0.60     -  |__ ModuleProcessTh
Average:  1000     - 7280    4.80    0.50    5.30     -  |__ AudioOutputDevi
Average:  1000     - 7300    0.00    0.00    0.00     -  |__ MemoryInfra
```

So almost half of the time is spent in audio processing.
Can we get anything interesting out of the AudioInput thread?
Without symbols not much:
```
$ sudo perf top -t 7275 --percent-limit 1 --stdio

  PerfTop:  1720 irqs/sec  kernel: 2.2%  exact: 100.0% lost: 0/0 drop: 0/0 [4000Hz cycles], (target_tid: 7275)
--------------------------------------------------------------------------------------------------------------

    11.67%  chrome            [.] 0x0000000005a9a3f5
     3.53%  chrome            [.] 0x0000000005ac6084
     2.91%  chrome            [.] 0x0000000005a99fb6
     ...
```

## pulseaudio

Let's study `pulseaudio` (PID 2665) next -- start by checking 10 second average
per-thread CPU usages:
```
$ pidstat -t -p $(pgrep -n pulseaudio) 1 10
Linux 5.5.8-100.fc30.x86_64     03/15/20     _x86_64_    (8 CPU)

Average:      UID  TGID   TID    %usr %system   %CPU  CPU  Command
Average:     1000  2665     -    5.39    7.39  12.79    -  pulseaudio
Average:     1000     -  2665    1.90    3.40   5.29    -  |__ pulseaudio
Average:     1000     -  2729    0.00    0.00   0.00    -  |__ alsa-sink-HDMI
Average:     1000     -  2732    1.90    1.50   3.40    -  |__ alsa-sink-CX207
Average:     1000     -  2736    1.60    2.70   4.30    -  |__ alsa-source-CX2
```

We see that the `pulseaudio` process has multiple threads, presumably one "main"
thread (TID 2665), and then one thread per input and output device.
Can we now get anything interesting with `perf top` out of these threads?

#### pulseaudio, TID 2665

```
$ sudo perf top -t 2665 --percent-limit 1 --stdio
[...]
  PerfTop:   591 irqs/sec  kernel:70.2%  exact: 100.0% lost: 0/0 drop: 0/0 [4000Hz cycles], (target_tid: 2665)
--------------------------------------------------------------------------------------------------------------

     5.84%  [kernel]                [k] __fget
     4.97%  [kernel]                [k] do_syscall_64
     4.41%  [kernel]                [k] sock_poll
     3.82%  [kernel]                [k] _raw_spin_lock_irqsave
     3.44%  [unknown]               [.] 0000000000000000
     3.39%  [kernel]                [k] do_sys_poll
     2.89%  [kernel]                [k] psi_task_change
     2.70%  libpulse.so.0.20.3      [.] pa_mainloop_dispatch
     2.48%  [kernel]                [k] fput_many
     2.39%  [kernel]                [k] __pollwait
     2.09%  [kernel]                [k] eventfd_poll
     1.95%  [kernel]                [k] _raw_spin_unlock_irqrestore
     1.69%  [kernel]                [k] entry_SYSCALL_64
     1.53%  [kernel]                [k] syscall_return_via_sysret
     1.35%  [kernel]                [k] unix_poll

$ sudo perf trace -t 2665 -s -- sleep 10

 Summary of events:

 pulseaudio (2665), 66755 events, 100.0%

  syscall        calls  errors  total       min       avg       max      stddev
                                (msec)    (msec)    (msec)    (msec)       (%)
  ----------- --------  ------ -------- --------- --------- ---------    ------
  ppoll           6624      0  9107.751     0.000     1.375     5.108     1.41%
  read           17180   6609   143.715     0.001     0.008     0.235     0.56%
  write           9033      0   122.016     0.001     0.014     0.252     0.59%
  futex            462      4    59.509     0.003     0.129     0.396     2.93%
  uname             42      0     0.366     0.001     0.009     0.020     7.61%
  getuid            41      0     0.311     0.001     0.008     0.024     9.26%

```

So the thread is doing thousands of syscall per second, and most of it's CPU
time is spent in the kernel. We ought to generate a [flame
graph](http://www.brendangregg.com/flamegraphs.html) to better understand
what's happening here, but for now let's move on.

*[One curious detail in the `perf trace` output is that about &#8531; of `read`
calls fail. Also I have not earlier noticed that "errors" column -- turns out it was
added recently in
[v5.5-rc1](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8eded45fcd3420e0f08b1e7869781f0e12927637).]*

#### alsa-sink, TID 2732

```
$ sudo perf top -t 2732 --percent-limit 1 --stdio

   PerfTop:  228 irqs/sec  kernel:47.4%  exact: 100.0% lost: 0/0 drop: 0/0 [4000Hz cycles], (target_tid: 2732)
--------------------------------------------------------------------------------------------------------------

    17.30%  [snd_hda_intel]         [k] azx_get_pos_skl
     4.13%  [unknown]               [.] 0000000000000000
     2.85%  [kernel]                [k] do_syscall_64
     2.81%  libspeexdsp.so.1.5.0    [.] 0x000000000000a255
     1.83%  [snd_hda_codec]         [k] azx_get_pos_lpib
     1.49%  [kernel]                [k] psi_task_change
```

#### alsa-source, TID 2736

```
$ sudo perf top -t 2736 --percent-limit 1 --stdio

   PerfTop:  205 irqs/sec  kernel:59.0%  exact: 100.0% lost: 0/0 drop: 0/0 [4000Hz cycles], (target_tid: 2736)
--------------------------------------------------------------------------------------------------------------

    28.89%  [kernel]                [k] delay_tsc
    16.30%  [snd_hda_intel]         [k] azx_get_pos_skl
     5.26%  [unknown]               [.] 0000000000000000
     2.48%  libspeexdsp.so.1.5.0    [.] 0x000000000000a255
     1.69%  [kernel]                [k] do_syscall_64
     1.18%  [snd_hda_codec]         [k] azx_get_pos_lpib
```

Now these look more interesting! For both threads we have high overhead kernel
symbol `azx_get_pos_skl`. And what's up with that `delay_tsc`?

## azx_get_pos_skl

We can try to understand what's happening by studying the
[`azx_get_pos_skl` function in the v5.5 kernel
sources](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/sound/pci/hda/hda_intel.c?h=v5.5#n904).
Turns out it's a short function:
{% highlight C %}
/* get the current DMA position with correction on SKL+ chips */
static unsigned int azx_get_pos_skl(struct azx *chip, struct azx_dev *azx_dev)
{
	/* DPIB register gives a more accurate position for playback */
	if (azx_dev->core.substream->stream == SNDRV_PCM_STREAM_PLAYBACK)
		return azx_skl_get_dpib_pos(chip, azx_dev);

	/* For capture, we need to read posbuf, but it requires a delay
	 * for the possible boundary overlap; the read of DPIB fetches the
	 * actual posbuf
	 */
	udelay(20);
	azx_skl_get_dpib_pos(chip, azx_dev);
	return azx_get_pos_posbuf(chip, azx_dev);
}
{% endhighlight %}

This provides some explanation to the `perf top` results: when in playback mode
(`alsa-sink` thread), "position" can be returned directly from "DPIB register"
(whatever that is).

But when capturing audio (`alsa-source` thread), the kernel has to
`udelay(20)`, ie. twiddle thumbs for 20 &micro;seconds before being able to
proceed. As it's a "delay" and not a "sleep", it can cause the CPU to to be
effectively "stuck", not being able to execute other kernel code (except
interrupts) or user space programs...? While sleeping, the CPU would be
explicitly free to execute other code.

Exact details probably depend on how `udelay` is implemented for given
architecture, and how the kernel is configured. I'll leave that investigation
to another blog post.

*[Based on the code comments, this function is used on `SKL+`: Skylake
(microarchitecture) and later. And what's my CPU?  `lscpu` says `Intel Core
i7-6820HQ`. Quick internet search confirms it's indeed
[Skylake](https://ark.intel.com/content/www/us/en/ark/products/88970/intel-core-i7-6820hq-processor-8m-cache-up-to-3-60-ghz.html).]*


## ftrace

We can study what's happening in the kernel with
[`ftrace`](https://www.kernel.org/doc/html/latest/trace/ftrace.html).
Let's try to use it to see how `azx_get_pos_skl` gets called.

We can run it simply with `sudo perf ftrace -a`, which starts the trace and
opens a pager, allowing to scroll the captured trace (in a busy machine, there
will be a ton of output from all CPUs in the system). We can look for the
function by typing `/azx_get_pos_skl`.

Bingo, the process (`alsa-source`) is doing `ioctl` syscall on CPU7, entering
the kernel, and ends up calling `azx_get_pos_skl`:
```
 7)             | do_syscall_64() {
 7)             |   __x64_sys_ioctl() {
 7)             |     ksys_ioctl() {
 7)             |       __fdget() {
 7)             |         __fget_light() {
 7)   0.491 us  |           __fget();
 7)   1.389 us  |         }
 7)   2.079 us  |       }
 7)   0.375 us  |       security_file_ioctl();
 7)             |       do_vfs_ioctl() {
 7)             |         snd_pcm_ioctl [snd_pcm]() {
 7)             |           snd_pcm_common_ioctl [snd_pcm]() {
 7)   0.376 us  |             snd_power_wait [snd]();
 7)             |             snd_pcm_hwsync [snd_pcm]() {
 7)             |               snd_pcm_stream_lock_irq [snd_pcm]() {
 7)   0.344 us  |                 _raw_spin_lock_irq();
 7)   1.007 us  |               }
 7)             |               do_pcm_hwsync [snd_pcm]() {
 7)             |                 snd_pcm_update_hw_ptr [snd_pcm]() {
 7)             |                   snd_pcm_update_hw_ptr0 [snd_pcm]() {
 7)             |                     azx_pcm_pointer [snd_hda_codec]() {
 7)             |                       azx_get_position [snd_hda_codec]() {
 7)             |                         azx_get_pos_skl [snd_hda_intel]() {
 7)             |                           __const_udelay() {
 7) + 20.311 us |                             delay_tsc();
 7) + 21.024 us |                           }
 7) + 11.420 us |                           azx_get_pos_posbuf [snd_hda_codec]();
 7) + 33.425 us |                         }
 7)             |                         azx_get_delay_from_lpib [snd_hda_intel]() {
 7)   1.126 us  |                           azx_get_pos_lpib [snd_hda_codec]();
 7)   1.844 us  |                         }
 7) + 36.604 us |                       }
 7) + 37.308 us |                     }
 7)   0.410 us  |                     ktime_get_ts64();
 7)             |                     update_audio_tstamp.isra.0 [snd_pcm]() {
 7)   0.340 us  |                       ns_to_timespec();
 7)   0.410 us  |                       ktime_get_ts64();
 7)   1.901 us  |                     }
 7)   0.408 us  |                     snd_pcm_update_state [snd_pcm]();
 7) + 42.144 us |                   }
 7) + 42.821 us |                 } /* snd_pcm_update_hw_ptr [snd_pcm] */
 7) + 43.483 us |               }
 7)   0.348 us  |               snd_pcm_stream_unlock_irq [snd_pcm]();
 7) + 46.204 us |             }
 7) + 47.711 us |           }
 7) + 48.395 us |         }
 7) + 49.281 us |       }
 7)             |       fput() {
 7)   0.354 us  |         fput_many();
 7)   0.986 us  |       }
 7) + 54.551 us |     }
 7) + 55.298 us |   }
 7) + 57.050 us | }
```

So we see that in the C code we have `udelay(20)`, and based on the ftrace
output this results to `__const_udelay()` function call, which then calls
`delay_tsc()`. According to the trace, `delay_tsc` call took `20.311 us`, so it
was really sleeping just a little over 20 &micro;seconds.



## Digging Kernel Git History

Let's dig the kernel git history, to find clues why the `udelay` call was
added. The function was added in 2017 in commit [`ALSA: hda - Improved
position reporting on
SKL+`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f87e7f25893d7db4da465c6b50882197e518d4af):
```
  ALSA: hda - Improved position reporting on SKL+

  Apply the same methods to obtain the current stream position as ASoC
  Intel SKL driver uses.  It reads the position from DPIB for a playback
  stream while it still reads from the position buffer for a capture
  stream.  For a capture stream, some ugly workaround is needed to
  settle down the inconsistent position.
```
So even the patch author does not like how the recording is done, calling it an
*ugly workaround*!

The commit message mentions `ASoC Intel SKL driver`, which we find in
`sound/soc/intel/skylake/`, and in `skl-pcm.c` we find a [more extensive
comment, explaining why the delay is 20
&micro;seconds:](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/sound/soc/intel/skylake/skl-pcm.c?h=v5.5#n1191)
```
/*
 * Use DPIB for Playback stream as the periodic DMA Position-in-
 * Buffer Writes may be scheduled at the same time or later than
 * the MSI and does not guarantee to reflect the Position of the
 * last buffer that was transferred. Whereas DPIB register in
 * HAD space reflects the actual data that is transferred.
 * Use the position buffer for capture, as DPIB write gets
 * completed earlier than the actual data written to the DDR.
 *
 * For capture stream following workaround is required to fix the
 * incorrect position reporting.
 *
 * 1. Wait for 20us before reading the DMA position in buffer once
 * the interrupt is generated for stream completion as update happens
 * on the HDA frame boundary i.e. 20.833uSec.
 * 2. Read DPIB register to flush the DMA position value. This dummy
 * read is required to flush DMA position value.
 * 3. Read the DMA Position-in-Buffer. This value now will be equal to
 * or greater than period boundary.
 */
```


## pulseaudio utils

I want to learn little bit more about Pulseaudio. It would be nice to be able
to record audio from the command line without involving the web browser. As
Pulseaudio is established open source software, I'd expect that it has some
reasonable tooling available, for example for audio playback and recording, and
for runtime inspecting the pulseaudio daemon.

Quick `rpm` query shows some promising binaries in the `pulseaudio-utils`
package:
```
$ rpm -qa "*pulseaudio*" | sort
[...]
pulseaudio-12.2-9.fc30.x86_64
pulseaudio-libs-12.2-9.fc30.x86_64
pulseaudio-libs-glib2-12.2-9.fc30.x86_64
pulseaudio-module-bluetooth-12.2-9.fc30.x86_64
pulseaudio-module-gconf-12.2-9.fc30.x86_64
pulseaudio-module-x11-12.2-9.fc30.x86_64
pulseaudio-utils-12.2-9.fc30.x86_64

$ rpm -ql pulseaudio | grep /bin
/usr/bin/pulseaudio

$ rpm -ql pulseaudio-utils | grep /bin
/usr/bin/pacat
/usr/bin/pacmd
/usr/bin/pactl
/usr/bin/padsp
/usr/bin/padsp-32
/usr/bin/pamon
/usr/bin/paplay
/usr/bin/parec
/usr/bin/parecord
/usr/bin/pasuspender
/usr/bin/pax11publish
```

With some experimentation and after reading the manual pages for these
binaries, found what I was looking for: `pactl` to inspect, and `parec` to
record.


## parec

Let's try `parec` to record some audio, and see if we get similar CPU% numbers
compared to the browser based video conference:
<script id="asciicast-BIXsdurCAHsEVkC6pk3s7sJSH" src="https://asciinema.org/a/BIXsdurCAHsEVkC6pk3s7sJSH.js"
 data-loop="1" data-autoplay="1" data-rows="12" data-speed="1.1" async></script>

Executed `pidstat` from second terminal, and CPU% is much smaller than
previously, but why?
```
$ pidstat -t -p $(pgrep -n pulseaudio) 1 10
[...]
Average:      UID   TGID    TID    %usr %system  %CPU  CPU  Command
Average:     1000   2665      -    1.30    0.60  1.90    -  pulseaudio
Average:     1000      -   2665    0.20    0.30  0.50    -  |__ pulseaudio
Average:     1000      -   2732    0.00    0.00  0.00    -  |__ alsa-sink-CX207
Average:     1000      -   2736    1.00    0.30  1.30    -  |__ alsa-source-CX2
```

Looking again at the `parec` command output, it's reporting latency, and the
numbers are quite high, near one second. That would be way too high for a
conference call!

From `pactl list` command output we see that the "configured" latency
is even higher, `2000000 usec` ie. two seconds:
```
$ pactl list | less
Source #2
        State: RUNNING
        Name: alsa_input.pci-0000_00_1f.3.analog-stereo
        Latency: 835610 usec, configured 2000000 usec
```


Checking `parec` manual page, we discover that it has latency control options,
and some explanation for the high latency we see with the default settings:

```
       --latency-msec=MSEC
	      Explicitly configure the latency, with a time specified in
	      milliseconds. If left out the server will pick the latency,
	      usually relatively high for power saving reasons.
```


Quick experiment with the `--latency-msec` flag shows that indeed there's large
effect on CPU usages. Here's a summary of 10 second average CPU%
for the pulseaudio process, and the `alsa-source` thread:

| `parec --latency-msec` | alsa-source thread CPU % | total process CPU % |
| ---------------------- | -----: | ----: |
| 1000ms                 |  2.80  |  4.10 |
| 100ms                  |  3.60  |  5.20 |
| 10ms                   |  8.10  | 15.90 |


## usleep_range

Based on ["timers-howto" kernel
documentation](https://www.kernel.org/doc/html/latest/timers/timers-howto.html)
(written ~2010) `udelay` is generally good for "a few" &micro;seconds -- 10
&micro;seconds or more could use `usleep_range`.

Let's experiment what happens, if we modify the kernel and replace the
`udelay()` call with `usleep_range()`. The required kernel code change is
simple, and in theory should result to the process sleeping for 20-30
&micro;seconds:
{% highlight diff %}
diff --git a/sound/pci/hda/hda_intel.c b/sound/pci/hda/hda_intel.c
index 92a042e34d3e5..aa6d2285be1c7 100644
--- a/sound/pci/hda/hda_intel.c
+++ b/sound/pci/hda/hda_intel.c
@@ -913,7 +913,7 @@ static unsigned int azx_get_pos_skl(struct azx *chip,
         * for the possible boundary overlap; the read of DPIB fetches the
         * actual posbuf
         */
-       udelay(20);
+       usleep_range(20, 30);
        azx_skl_get_dpib_pos(chip, azx_dev);
        return azx_get_pos_posbuf(chip, azx_dev);
 }
{% endhighlight %}

After compiling and installing the patched kernel, and booting into it, let's
first confirm that the change is in effect. With ftrace we can see that
`usleep_range` is called, and results to the calling thread being scheduled out
for the duration of the sleep:
```
 3)               | do_syscall_64() {
 3)               |   __x64_sys_ioctl() {
 3)               |     ksys_ioctl() {
 3)               |       snd_pcm_ioctl() {
 3)               |         snd_pcm_common_ioctl() {
 3)   0.520 us    |           snd_power_wait();
 3)               |           snd_pcm_hwsync() {
 3)               |             do_pcm_hwsync() {
 3)               |               snd_pcm_update_hw_ptr() {
 3)               |                 snd_pcm_update_hw_ptr0() {
 3)               |                   azx_pcm_pointer() {
 3)               |                     azx_get_position() {
 3)               |                       azx_get_pos_skl() {
 3)               |                         usleep_range() {
 3)   0.685 us    |                           ktime_get();
 3)               |                           schedule_hrtimeout_range() {
 3)               |                             schedule_hrtimeout_range_clock() {
 3)               |                               schedule() {
 ------------------------------------------
 3)  alsa-so-2416  =>    <idle>-0
 ------------------------------------------
 3)   0.720 us    | switch_mm_irqs_off();
 ------------------------------------------
 3)    <idle>-0    =>  alsa-so-2416
 ------------------------------------------
 3)   0.558 us    |                                 finish_task_switch();
 3) + 63.434 us   |                               }
 3)               |                               hrtimer_try_to_cancel() {
 3)   0.522 us    |                                 hrtimer_active();
 3)   1.587 us    |                               }
 3) + 80.930 us   |                             }
 3) + 81.975 us   |                           }
 3) + 84.207 us   |                         }
 3) + 11.549 us   |                         azx_get_pos_posbuf();
 3) + 97.325 us   |                       }
 3)               |                       azx_get_delay_from_lpib() {
 3)   1.616 us    |                         azx_get_pos_lpib();
 3)   2.708 us    |                       }
 3) ! 102.308 us  |                     }
 3) ! 103.326 us  |                   }
 3) ! 110.055 us  |                 }
 3) ! 111.011 us  |               }
 3) ! 112.061 us  |             }
 3) ! 116.099 us  |           }
 3) ! 118.118 us  |         }
 3) ! 119.086 us  |       }
 3) ! 127.386 us  |     }
 3) ! 128.386 us  |   }
 3)   0.554 us    |   switch_fpu_return();
 3) ! 131.661 us  | } /* do_syscall_64 */
```

In the trace output we see that the `alsa-source` thread got scheduled out for
63 &micro;seconds -- more than we want, but ftrace probably also adds some
extra overhead to the numbers.

The `funcslower` BCC tool reports that the `usleep_range` calls are taking
about 40 &micro;seconds to complete, and `funclatency` produces a nice
histogram, showing that the sleeps are in most cases completing in under 64
&micro;seconds:
```
# /usr/share/bcc/tools/funcslower -u 10 -p $(pgrep -n pulseaudio) usleep_range
COMM           PID    LAT(us)             RVAL FUNC
alsa-source-CX 2371     38.12                0 usleep_range
alsa-source-CX 2371     42.80                0 usleep_range
alsa-source-CX 2371     40.62                0 usleep_range
alsa-source-CX 2371     45.67                0 usleep_range
alsa-source-CX 2371     40.48                0 usleep_range
alsa-source-CX 2371     42.16                0 usleep_range
alsa-source-CX 2371     40.29                0 usleep_range
alsa-source-CX 2371     38.36                0 usleep_range
[...]

# /usr/share/bcc/tools/funclatency -Fu -p $(pgrep -n pulseaudio) -d30 usleep_range
Tracing 1 functions for "usleep_range"... Hit Ctrl-C to end.

Function = b'usleep_range'
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 5        |                                        |
        32 -> 63         : 19086    |****************************************|
        64 -> 127        : 15       |                                        |
```

For reference, with the Fedora `5.5.8-100.fc30.x86_64` kernel
`delay_tsc` mostly completes in less than 32 &micro;seconds:
```
# /usr/share/bcc/tools/funclatency -Fu -p $(pgrep -n pulseaudio) -d30 delay_tsc
Tracing 1 functions for "delay_tsc"... Hit Ctrl-C to end.

Function = b'delay_tsc'
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 19157    |****************************************|
        32 -> 63         : 6        |                                        |
```

What about CPU usage? Comparing pulseaudio `pidstat` results on patched and
unpatched 5.6-rc5, there's actually very little change. *[But interestingly
CPU% is smaller in the kernel I compiled myself vs. the Fedora kernel.
Perhaps the Fedora kernel gets compiled with additional compiler hardenings, or
in my kernel some config option change is affecting performance.]*

It's an interesting experiment, but perhaps 20 &micro;seconds is too short time
to run other meaningful work on the CPU. The trace data shows that the original
20 &micro;second delay doubles into 40 &micro;second sleeps -- the additional
latency could cause other issues.


## USB Headset

We can sidestep the laptop builtin audio by using a USB headset.
Quick comparison of the pulseaudio CPU% *[but it's not totally apples-to-apples
comparison -- the laptop audio records stereo, the USB headset records mono]*:

| `parec --latency-msec` (builtin audio) | alsa-source thread CPU % | total CPU % |
| ---------------------- | -----: | ----: |
| 1000ms                 |  2.80  |  4.10 |
| 100ms                  |  3.60  |  5.20 |
| 10ms                   |  8.10  | 15.90 |

| `parec --latency-msec` (USB headset) | alsa-source thread CPU % | total CPU % |
| ---------------------- | -----: | ----: |
| 1000ms                 |  0.70  |  1.50 |
| 100ms                  |  2.10  |  5.50 |
| 10ms                   |  3.20  |  9.40 |


## Firefox

Testing the Webex conference with Firefox (version 74), interestingly the CPU
usages of Firefox and Pulseaudio are lower than what I saw with Google Chrome
(again idle meeting and microphone muted):
```
top - 17:06:38 up 1 day,  3:45,  7 users,  load average: 0,94, 0,53, 0,73
Tasks: 377 total,   1 running, 376 sleeping,   0 stopped,   0 zombie
%Cpu(s):  5,7 us,  3,9 sy,  0,0 ni, 88,7 id,  0,1 wa,  1,1 hi,  0,5 si,  0,0 st
MiB Mem :  23966,1 total,   8509,3 free,   4169,2 used,  11287,7 buff/cache
MiB Swap:   8032,0 total,   8032,0 free,      0,0 used.  19106,5 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 9601 tommi     20   0 3024256 291772 148348 S  44,7  1,2   0:25.15 Web Content
 9330 tommi     20   0 3330140 330436 154532 S  17,5  1,3   0:25.20 firefox
 2665 tommi     20   0 2240048  15724  11184 S   7,3  0,1  15:38.73 pulseaudio
```

Turns out that Firefox will stop the audio recording when the microphone is
muted in the meeting, and will start it again after unmuting the microphone.


## Summary

We started from `top` command output, dug deeper into the kernel, and
discovered an "ugly workaround" in Intel HD Audio drivers that applies to
recent (Skylake+) systems. We also learned little bit about Pulseaudio, and got
a chance to experimentally patch the kernel, and use various Linux tracing and
performance analysis tools.


## Update May 2021

Comment from user *L29Ah*:

*The following modprobe.d workaround seems to fix the insane busy waiting on my X210:*
```
options snd_hda_intel position_fix=1
```

&mdash;

Indeed the driver offers various parameters that may be helpful:
```
$ modinfo snd_hda_intel
description:  Intel HDA driver
[...]
parm:   index:Index value for Intel HD audio interface. (array of int)
parm:   id:ID string for Intel HD audio interface. (array of charp)
parm:   enable:Enable Intel HD audio interface. (array of bool)
parm:   model:Use the given board model. (array of charp)
parm:   position_fix:DMA pointer read method. (-1 = system default, 0 = auto,
           1 = LPIB, 2 = POSBUF, 3 = VIACOMBO, 4 = COMBO, 5 = SKL+, 6 = FIFO).
parm:   bdl_pos_adj:BDL position adjustment offset. (array of int)
parm:   probe_mask:Bitmask to probe codecs (default = -1). (array of int)
parm:   probe_only:Only probing and no codec initialization. (array of int)
parm:   jackpoll_ms:Ms between polling for jack events (default = 0,
           using unsol events only) (array of int)
parm:   single_cmd:Use single command to communicate with codecs
           (for debugging only). (bint)
parm:   enable_msi:Enable Message Signaled Interrupt (MSI) (bint)
[...]
```
