---
layout: post
title:  "Fixing XFS getdents regression in v5.5"
date:   2021-12-29 12:00:00 +0200
description: "Fixing XFS getdents regression in v5.5"
---

This happened in March 2020, after upgrading the Fedora kernel from v5.4.x to
v5.5.x in a server processing [GitLab CI/CD jobs (GitLab Runner)](https://docs.gitlab.com/runner/).
One CI job started failing right after the upgrade with GitLab Runner
"built-in" `git clean` (doing workspace cleanup) erroring out with message:

> warning: failed to remove .vendor/pkg/mod/golang.org/x/net@v0.0.0-20200114155413-6afb5195e5aa/internal/socket: Directory not empty

Quick Google search revealed similar problems reported by a few other users
running GitLab CI/CD with Linux v5.5.x (and various reports from Windows and
MacOS users):
[https://gitlab.com/gitlab-org/gitlab-runner/-/issues/3185](https://gitlab.com/gitlab-org/gitlab-runner/-/issues/3185)

[User @Ries reports](https://gitlab.com/gitlab-org/gitlab-runner/-/issues/3185#note_278136191):
> In my case it seems to be realted to the linux kernel - i just downgraded to 5.4.xx and all gitlab-ci pipelines are running again!
>
> broken: kernel 5.5.0-1.el7.elrepo.x86_64 working: 5.4.14-1.el7.elrepo.x86_64

*[Note: Linux 5.4 was released Nov 2019, Linux 5.5 was released Jan 2020]*

### How to debug the failing `git clean` command?

It's not immediately clear (to me) how to debug or trace such short running
`git` program, that is launched via GitLab CI, executed automatically by GitLab
Runner (`gitlab-runner` process) and not directly user controlled, and running
in (docker) container.

After some experimentation, found a way to `strace` the `git` process, simply
by attaching to the `containerd` process and tracing all child processes, and
then triggering the CI job. Repeated this with both v5.4.x and v5.5.x kernels
to be able to compare the results.

Based on the collected data, when `git clean` is processing the particular
directory, it's calling `getdents` several times, and after each call, it's
removing all the files that `getdents` returns. The difference between v5.4.x
and v5.5.x was that the `getdents` syscalls were returning in total one less
entry in the new kernel. This then caused the directory removal (`rmdir`
syscall) to fail, as one file was not removed from the directory.

### getdents syscall

So it looked like a regression in XFS `getdents` in v5.5. But this is highly
suspicious -- it's such a basic filesystem operation, how could it have a
regression?

I did not find other reports of this issue on the mailing lists, so
[decided to report the problem on LKML](https://lore.kernel.org/lkml/72c5fd8e9a23dde619f70f21b8100752ec63e1d2.camel@nokia.com/)
explaining the difference in my results (below slightly edited):

> Collected some data with strace, and it seems that getdents is not returning
> all entries:
>
> 5.4 getdents64() returns 52+50+1+0 entries  
> => all files in directory are deleted and rmdir() is OK
> 
> 5.5 getdents64() returns 52+50+0+0 entries  
> => rmdir() fails with ENOTEMPTY
> 
> Working 5.4 strace:
> ```
> 10:00:12 getdents64(10<…/.vendor/…/internal/socket>, /* 52 entries */, 2048) = 2024
> 10:00:12 unlink(".vendor/…/internal/socket/cmsghdr.go") = 0
> 10:00:12 unlink(".vendor/…/internal/socket/cmsghdr_bsd.go") = 0
> […]
> 10:00:12 getdents64(10<…/.vendor/…/internal/socket>, /* 50 entries */, 2048) = 2048
> 10:00:12 unlink(".vendor/…/internal/socket/sys_linux_386.s") = 0
> […]
> 10:00:12 getdents64(10<…/.vendor/…/internal/socket>, /* 1 entries */, 2048) = 48
> 10:00:12 unlink(".vendor/…/internal/socket/zsys_solaris_amd64.go") = 0
> 10:00:12 getdents64(10<…/.vendor/…/internal/socket>, /* 0 entries */, 2048) = 0
> 10:00:12 rmdir(".vendor/…/internal/socket") = 0
> ```
> 
> Failing 5.5 strace:
> ```
> 10:09:15 getdents64(10<…/.vendor/…/internal/socket>, /* 52 entries */, 2048) = 2024
> 10:09:15 unlink(".vendor/…/internal/socket/cmsghdr.go") = 0
> […]
> 10:09:15 getdents64(10<…/.vendor/…/internal/socket>, /* 50 entries */, 2048) = 2048
> 10:09:15 unlink(".vendor/…/internal/socket/sys_linux_386.s") = 0
> […]
> 10:09:16 getdents64(10<…/.vendor/…/internal/socket>, /* 0 entries */, 2048) = 0
> 10:09:16 rmdir(".vendor/…/internal/socket") = -1 ENOTEMPTY (Directory not empty)
> ```

### Response on LKML

XFS maintainer *Dave Chinner* soon [explained that it looks more like a
userspace problem](https://lore.kernel.org/lkml/20200310221406.GO10776@dread.disaster.area/):
> Yup, that's a classic userspace TOCTOU race.
>
> Remember, getdents() is effectively a sequential walk through the
> directory data - subsequent calls start at the offset (cookie) where
> the previous one left off. New entries can be added between
> getdents() syscalls.
>
> [...]
>
> IOWs, this is exactly what you'd expect to see when there are
> concurrent userspace modifications to a directory that is currently
> being read. Hence you need to rule out an application and userspace
> level issues before looking for filesystem level problems.

### Bisection

It was time to bisect the kernel to find out exactly which commit between
v5.4.x and v5.5.x introduced the problem, also proving that it's really a
regression in the kernel, and not a problem in userspace.

After spending some time to build kernels to run the bisection (rebooting
the server is painfully slow, taking several minutes), could confirm the
regression is introduced by a "cleanup" commit
[`xfs: cleanup xfs_dir2_block_getdents`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=263dde869bd09b1a709fd92118c7fff832773689):
```
commit 263dde869bd09b1a709fd92118c7fff832773689
Author: Christoph Hellwig <hch@lst.de>
Date:   Fri Nov 8 15:05:32 2019 -0800

    xfs: cleanup xfs_dir2_block_getdents
    
    Use an offset as the main means for iteration, and only do pointer
    arithmetics to find the data/unused entries.
    
    Signed-off-by: Christoph Hellwig <hch@lst.de>
    Reviewed-by: Darrick J. Wong <darrick.wong@oracle.com>
    Signed-off-by: Darrick J. Wong <darrick.wong@oracle.com>
```

### Fixing the regression

If you check the above commit, note that there is a slight change in how the
`offset` variable is used in the `while` loop compared to the previous `ptr`
and `dep` pointers. Once I understood this, it was straightforward to fix and
test it locally. With a fix in place, the CI job including the `git clean`
command was now passing fine!

*Christoph Hellwig* suggested improvement on my initial patch, and
[the fix was accepted and included in v5.7-rc1 (later also included in v5.6.7)](
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3d28e7e278913a267b1de360efcd5e5274065ce2):
```
commit 3d28e7e278913a267b1de360efcd5e5274065ce2
Author: Tommi Rantala <tommi.t.rantala@nokia.com>
Date:   Thu Mar 12 07:43:53 2020 -0700

    xfs: fix regression in "cleanup xfs_dir2_block_getdents"
    
    Commit 263dde869bd09 ("xfs: cleanup xfs_dir2_block_getdents") introduced
    a getdents regression, when it converted the pointer arithmetics to
    offset calculations: offset is updated in the loop already for the next
    iteration, but the updated offset value is used incorrectly in two
    places, where we should have used the not-yet-updated value.
    
    This caused for example "git clean -ffdx" failures to cleanup certain
    directory structures when running in a container.
    
    Fix the regression by making sure we use proper offset in the loop body.
    Thanks to Christoph Hellwig for suggestion how to best fix the code.
    
    Cc: Christoph Hellwig <hch@lst.de>
    Fixes: 263dde869bd09 ("xfs: cleanup xfs_dir2_block_getdents")
    Signed-off-by: Tommi Rantala <tommi.t.rantala@nokia.com>
    Reviewed-by: Christoph Hellwig <hch@lst.de>
    Reviewed-by: Darrick J. Wong <darrick.wong@oracle.com>
    Signed-off-by: Darrick J. Wong <darrick.wong@oracle.com>
    Reviewed-by: Dave Chinner <dchinner@redhat.com>
```

## Conclusions

‣ [Comment from user @stratusjerry](https://gitlab.com/gitlab-org/gitlab-runner/-/issues/3185#note_796964227)
just when I was writing this blog post:
> thank you for this comment! I'm running Linux Gitlab Runner in Docker on a
> Linux host and found upgrading kernels to one that included this patch fixed
> all the failure to delete build artifact directory/files issues we were
> seeing!

Thanks, you're very welcome!

‣ There will be occasional regresssions when running distributions like
Fedora that ships with the latest upstream kernels.

‣ I could have found the offending commit easier by checking XFS changes between
v5.4 and v5.5, instead of the slow bisection:
```
$ git log --oneline v5.4..v5.5 | grep -i xfs.*getdents
2f4369a862b6 xfs: cleanup xfs_dir2_leaf_getdents
263dde869bd0 xfs: cleanup xfs_dir2_block_getdents
```

‣ Booting (most?) server hardware is slow, should look into
[kexec](https://fedoraproject.org/wiki/Kernel/kexec).
![Supermicro server](/assets/Supermicro_SBI-7228R-T2X_blade_server.jpg)

*[Image by Dmitry Nosachev - Own work, CC BY-SA 4.0](https://commons.wikimedia.org/w/index.php?curid=46899884)*
