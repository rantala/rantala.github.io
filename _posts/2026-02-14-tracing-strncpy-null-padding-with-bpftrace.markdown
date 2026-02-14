---
layout: post
title:  "Tracing strncpy() null padding with bpftrace"
date:   2026-02-14 12:00:00 +0200
description: "Using bpftrace to trace strncpy() null-padding"
---

The `strncpy()` function has a well-known performance gotcha: when the source
string is shorter than the buffer size, it zero-pads the entire destination
buffer for all remaining bytes:
```c
char buf[4096];
strncpy(buf, short_string, sizeof(buf));  // Will zero-fill 4KB
```

Even if `short_string` is only 5 bytes, `strncpy()` will write zeros to all
remaining 4091 bytes. This can be a significant performance issue, especially
in hot code paths or with large buffers.

## Tracing `strncpy` with bpftrace

Let's write a [bpftrace](https://github.com/bpftrace/bpftrace) script to trace
this padding behavior in running programs.

### First attempt

My first attempt to trace `uprobe:libc:strncpy` with bpftrace curiously
produces bogus results (argument values) on Ubuntu 24.04.

Checking with GDB gives us a clue: `strncpy` uses **ifunc** (indirect function)
to select the optimal implementation at runtime based on CPU capabilities:
```
$ gdb /lib/x86_64-linux-gnu/libc.so.6
[...]
(gdb) disassemble strncpy
Dump of assembler code for function strncpy_ifunc:
   0x00000000000b4fb0 <+0>:     endbr64
   0x00000000000b4fb4 <+4>:     mov    0x14def5(%rip),%rcx        # 0x202eb0
   0x00000000000b4fbb <+11>:    lea    0x1160e(%rip),%rax        # 0xc65d0 <__strncpy_sse2_unaligned>
   0x00000000000b4fc2 <+18>:    mov    0xb8(%rcx),%edx
   0x00000000000b4fc8 <+24>:    test   $0x20,%dl
   0x00000000000b4fcb <+27>:    je     0xb4ffd <strncpy_ifunc+77>
   0x00000000000b4fcd <+29>:    mov    0x1c4(%rcx),%ecx
   0x00000000000b4fd3 <+35>:    test   $0x2,%ch
   0x00000000000b4fd6 <+38>:    je     0xb4ffd <strncpy_ifunc+77>
   0x00000000000b4fd8 <+40>:    test   %edx,%edx
   0x00000000000b4fda <+42>:    js     0xb5000 <strncpy_ifunc+80>
   0x00000000000b4fdc <+44>:    lea    0xe0a1d(%rip),%rax        # 0x195a00 <__strncpy_avx2_rtm>
   0x00000000000b4fe3 <+51>:    and    $0x8,%dh
   0x00000000000b4fe6 <+54>:    jne    0xb4ffd <strncpy_ifunc+77>
   0x00000000000b4fe8 <+56>:    and    $0x8,%ch
   0x00000000000b4feb <+59>:    lea    0xd7f0e(%rip),%rax        # 0x18cf00 <__strncpy_avx2>
   0x00000000000b4ff2 <+66>:    lea    0x115d7(%rip),%rdx        # 0xc65d0 <__strncpy_sse2_unaligned>
   0x00000000000b4ff9 <+73>:    cmovne %rdx,%rax
   0x00000000000b4ffd <+77>:    ret
   0x00000000000b4ffe <+78>:    xchg   %ax,%ax
   0x00000000000b5000 <+80>:    lea    0xe80f9(%rip),%rax        # 0x19d100 <__strncpy_evex>
   0x00000000000b5007 <+87>:    test   $0x40000000,%edx
   0x00000000000b500d <+93>:    je     0xb4fdc <strncpy_ifunc+44>
   0x00000000000b500f <+95>:    ret
End of assembler dump.
```

On my system, the resolver selects `__strncpy_avx2`, so we trace that directly.
Depending on your CPU and libc, you may need to trace a different variant.

### bpftrace script

Code repository here: [strncpy-bpftrace](https://github.com/rantala/strncpy-bpftrace)

The tracing script is below. Unfortunately we have to hardcode the max source
string size, here 10000 bytes.

```c
uprobe:libc:__strncpy_avx2 {
	@count[comm] = count();

	$dst = (int8 *)arg0;
	$src = (int8 *)arg1;
	$n = arg2;

	$padding = 0;
	$srclen = 0;
	for ($i : 0..10000) {
		if ($src[(uint64)$i] == 0) {
			$srclen = $i;
			$padding = (int64)$n - (int64)$i;
			break;
		}
		if ((uint64)$i == $n) {
			$srclen = $i;
			$padding = 0;
			break;
		}
	}

	if ($padding > 0) {
		printf("%s pid=%d dst=%p src=%p n=%d srclen=%d padding=%d\n",
		       comm, pid, $dst, $src, $n, $srclen, $padding);
	}
}
```

To test it you need:
- Linux system with eBPF support
- [bpftrace](https://github.com/bpftrace/bpftrace) installed
  (tested with bpftrace version v0.24.2)
- Root privileges (required for bpftrace)

The script monitors all `strncpy()` calls system-wide and prints information
when padding is detected, for example:
```bash
$ sudo bpftrace trace.bt
[...]
test pid=12345 dst=0x7ffc12345678 src=0x7ffc12345680 n=64 srclen=5 padding=59
```

Output fields:
- **comm**: Process name
- **pid**: Process ID
- **dst**: Destination buffer address
- **src**: Source string address
- **n**: Destination buffer size passed to strncpy
- **srclen**: Source string length (max `n`)
- **padding**: Number of null bytes written


## Real-world Performance Problem in `pgrep`

There was relatively poor performance in `pgrep` from procps-ng v3.3.16
(released 8 Dec 2019).

Our bpftrace scripts reveals it's zero-padding a 2MB buffer repeatedly:
```
[...]
pgrep pid=52573 dst=0x7f22efbfd010 src=0x7ffc8496c768 n=2097151 srclen=25 padding=2097126
pgrep pid=52573 dst=0x7f22efdfe010 src=0x7ffc8496c768 n=2097151 srclen=25 padding=2097126
pgrep pid=52573 dst=0x7f22efbfd010 src=0x7ffc8496c768 n=2097151 srclen=29 padding=2097122
pgrep pid=52573 dst=0x7f22efdfe010 src=0x7ffc8496c768 n=2097151 srclen=29 padding=2097122
pgrep pid=52573 dst=0x7f22efbfd010 src=0x7ffc8496c768 n=2097151 srclen=4 padding=2097147
pgrep pid=52573 dst=0x7f22efdfe010 src=0x7ffc8496c768 n=2097151 srclen=4 padding=2097147
pgrep pid=52573 dst=0x7f22efbfd010 src=0x7ffc8496c768 n=2097151 srclen=4 padding=2097147
pgrep pid=52573 dst=0x7f22efdfe010 src=0x7ffc8496c768 n=2097151 srclen=4 padding=2097147
pgrep pid=52573 dst=0x7f22efbfd010 src=0x7ffc8496c768 n=2097151 srclen=8 padding=2097143
pgrep pid=52573 dst=0x7f22efdfe010 src=0x7ffc8496c768 n=2097151 srclen=8 padding=2097143
pgrep pid=52573 dst=0x7f22efbfd010 src=0x7ffc8496c768 n=2097151 srclen=8 padding=2097143
pgrep pid=52573 dst=0x7f22efdfe010 src=0x7ffc8496c768 n=2097151 srclen=8 padding=2097143
```

The excessive padding contributes to relatively poor performance:
```
$ time ./pgrep foo
real    0m0,316s
user    0m0,251s
sys     0m0,070s
```

Fortunately, things are better in newer versions. For example procps-ng v4.0.4
uses a smaller buffer, and the performance is naturally better:
```
pgrep pid=57373 dst=0x7f2e140aa010 src=0x55c2b4fab670 n=131071 srclen=8 padding=131063
pgrep pid=57373 dst=0x7f2e14089010 src=0x55c2b4fae080 n=131071 srclen=17 padding=131054
pgrep pid=57373 dst=0x7f2e140aa010 src=0x55c2b4fae080 n=131071 srclen=17 padding=131054
```
```
$ time pgrep foo
real    0m0,054s
user    0m0,010s
sys     0m0,044s
```

## Links

- [strncpy-bpftrace sample code](https://github.com/rantala/strncpy-bpftrace)
- [bpftrace](https://github.com/bpftrace/bpftrace)
- [strncpy() man page](https://man7.org/linux/man-pages/man3/strncpy.3.html)
- [strncpy: not the function you are looking for](https://meyering.net/crusade-to-eliminate-strncpy/)

To support my work and to motivate me:
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="trantala" data-color="#FFDD00" data-emoji="☕"  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
