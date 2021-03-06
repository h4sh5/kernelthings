# Linux kernel development setup for beginners

So you want to hack on the Linux kernel? This article will guide you through the first steps to setting up, compiling and running the kernel. You will need a couple of things to get started:

- You should have used Linux before as a user - and have gotten quite familiar with it. For example, you will need to know how to install some of the dependency packages mentioned

- You know your C. The kernel is mainly written in C

- You know some git (not necessary but will be useful)

I am going to assume you are following these instructions on a Linux machine (bare metal, not Virtual Machine) of some sort. This is because we will need a bit of serious compute power to compile the kernel under an hour, and run `qemu` to boot up the kernel in a VM.

## Getting source 

Let's clone the git repo - this is around 5GBs on disk at the time of writing, so it might take a while to download.. (took me around 30 minutes, depends on your internet speed)

```
git clone http://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

Note that this will clone the ENTIRE git tree (with all the commit history going waaay back to 2005, Linux 2.6.12-rc2 and at that point it was already 3.2GB, as shown on the tail end of `git log`:

```
commit 1da177e4c3f41524e886b7f1b8a0c1fc7321cac2 (tag: v2.6.12-rc2)
Author: Linus Torvalds <torvalds@ppc970.osdl.org>
Date:   Sat Apr 16 15:20:36 2005 -0700

    Linux-2.6.12-rc2
    
    Initial git repository build. I'm not bothering with the full history,
    even though we have it. We can create a separate "historical" git
    archive of that later if we want to, and in the meantime it's about
    3.2GB when imported into git - space that would just make the early
    git days unnecessarily complicated, when we don't have a lot of good
    infrastructure for it.
    
    Let it rip!

```

If you just want a specific branch (e.g. `v5.18`), do this:

```
git clone --depth 1  http://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git -b v5.18
```

that will save you quite a lot of download time (and disk space)

Now `cd` into the new linux repo you just cloned.

## Compiling

First you need to configure the build. Make sure you have `gcc` and `binutils` installed

Run `make defconfig` to get a basic config that should work on your current machine.

(Optional) Run `make menuconfig` to change the kernel configuration. You will need the ncurses headers for this to run (Debian based systems: `apt install libncurses-dev`, Redhat based systems `dnf install ncurses-devel`, etc.) You will get presented with a pretty menu to go around changing configurations and save it.

Optionally, if you know the options you want to change (say you stumbled upon something on stackoverflow), just edit `.config`.

Now you're ready to run `time make -j8` or `time make -j16` or however many cpu cores you have `make -j$(nproc)`

> Pro tip: if you're on a hard disk drive, copy all the files over to a SSD (Solid State Drive), or better, if you have around 8G of RAM (which you probably will on a modern setup), copy all files over to `/dev/shm` and compile it there for max speed (because all the files will be directly in memory! They just won't be available when rebooting so make sure you save your changes/artifacts. Don't copy the `.git` repo if you want to save space. The difference between compiling on my SSD and in `/dev/shm` was around 33 minutse vs 10 minutes.

If you're on x86 your compiled kernel can be found in `arch/x86/boot/bzImage` inside your linux repo root

## Running it in a VM

Now you're done with compiling the kernel, it's time to ~~install it on your host machine's /boot/vmlinuz-5.18.something.x86_64, update grub config and reboot~~ run it on a Virtual Machine for testing!

To run it in a VM like `qemu` you want to have qemu installed first (refer to your distro documentation), and you will need a filesystem / image for your new kernel to run on.

Luckily our good friends over at Debian has a `qcow2` image we can use. It can usually be found here https://www.debian.org/distrib/ under "OpenStack provider", e.g. https://cloud.debian.org/images/cloud/bullseye/latest/debian-11-generic-amd64.qcow2

After downloading that, you can spin up your qemu VM:

`qemu-kvm -hda ~/Downloads/debian-11-generic-amd64.qcow2  -kernel arch/x86/boot/bzImage -append "console=ttyS0 root=/dev/sda1 init=/bin/bash" -serial stdio -no-reboot -display none -m 1G`

the boot options specified in `-append`:
- `console=ttyS0` shows console output in the first serial tty, and allows you to interact with the kernel via inputs
- `root=/dev/sda1` because the root partition is on the first partition of the qcow2 image
- `init=/bin/bash` runs bash on init so you get a shell


The options `-serial stdio` and `-display none` basically does not spin up a new window / GUI (qemu usually does) and instead spawns the new system directly in your current terminal. That way you can copy/paste kernel logs etc.

If you get rid of `-display none` you will get a cute little qemu window popup with the VM in it.

and here we go!

```
... (snipped)
[    2.370680]   No soundcards found.
[    2.510166] tsc: Refined TSC clocksource calibration: 1992.006 MHz
[    2.511082] clocksource: tsc: mask: 0xffffffffffffffff max_cycles: 0x396d5dac02a, max_idle_ns: 881590811122 ns
[    2.514123] clocksource: Switched to clocksource tsc
[    2.909784] input: ImExPS/2 Generic Explorer Mouse as /devices/platform/i8042/serio1/input/input3
[    2.915316] md: Waiting for all devices to be available before autodetect
[    2.915382] md: If you don't use raid, use raid=noautodetect
[    2.915792] md: Autodetecting RAID arrays.
[    2.915942] md: autorun ...
[    2.916067] md: ... autorun DONE.
[    2.972777] EXT4-fs (sda1): mounted filesystem with ordered data mode. Quota mode: none.
[    2.972988] VFS: Mounted root (ext4 filesystem) readonly on device 8:1.
[    2.977881] devtmpfs: mounted
[    3.025617] Freeing unused kernel image (initmem) memory: 1280K
[    3.025892] Write protecting the kernel read-only data: 24576k
[    3.029600] Freeing unused kernel image (text/rodata gap) memory: 2032K
[    3.030624] Freeing unused kernel image (rodata/data gap) memory: 972K
[    3.031350] Run /bin/bash as init process
bash: cannot set terminal process group (-1): Inappropriate ioctl for device
bash: no job control in this shell
root@(none):/# uname -a
Linux (none) 5.19.0-rc1-00262-g0885eacdc81f #3 SMP PREEMPT_DYNAMIC Mon Jun 13 09:08:48 AEST 2022 x86_64 GNU/Linux
[   37.223124] uname (74) used greatest stack depth: 14176 bytes left
root@(none):/# id
uid=0(root) gid=0(root) groups=0(root)
[   37.806913] id (75) used greatest stack depth: 13928 bytes left
root@(none):/# 

```

## Making a change - hello world

Obviously, adding a print hello world message into the kernel is not a real 'change' and we are doing this just for fun. No one should ever submit a useless patch like that.

But for the sake of making a change and seeing if things happen, let's do it!

So (part of) the kernel start up code is in `init/main.c`. If we browse this file we see a function	called `kernel_init` - that's where we will add our hello world.

It starts like this:

```c

static int __ref kernel_init(void *unused)
{
	int ret;

	/*
	 * Wait until kthreadd is all set-up.
	 */
	wait_for_completion(&kthreadd_done);

	kernel_init_freeable();
	/* need to finish all async __init code before freeing the memory */
	async_synchronize_full();

	system_state = SYSTEM_FREEING_INITMEM;
	kprobe_free_init_mem();
	ftrace_free_init_mem();
	kgdb_free_init_mem();
	exit_boot_config();
	free_initmem();
	mark_readonly();

	/*
	 * Kernel mappings are now finalized - update the userspace page-table
	 * to finalize PTI.
	 */
	pti_finalize();

	... (snipped)
```

It's always good practise to look around the file and see what other code looks like, in order to immitate existing code and keep things looking consistent.

For example, we see a couple examples of printing things in the same file:

```c
panic("%s: Failed to allocate %zu bytes\n", __func__, len); // that looks bad, don't want to kernel panic

pr_info("Run %s as init process\n", init_filename); // that looks nice!

pr_debug("  with arguments:\n"); // is debugging on? will this print if not?

pr_warn("Kernel memory protection not selected by kernel config.\n"); // so this must be a warning

// that looks like an error, lets not do that
if (ret && ret != -ENOENT) {
		pr_err("Starting init: %s exists but couldn't execute it (error %d)\n",
		       init_filename, ret);
	}

```

So it looks like the most "appropriate" print statement for a hello world is `pr_info`. Let's add this into kernel_init:

(around line 1513 at the time of writing in kernel 5.17, but can change)

```c
	exit_boot_config();
	free_initmem();
	mark_readonly();
	pr_info("Hello world!\n"); // <- here's our hello world

	/*
	 * Kernel mappings are now finalized - update the userspace page-table
	 * to finalize PTI.
	 */
	pti_finalize();

	system_state = SYSTEM_RUNNING;
	numa_default_policy();
```

Now let's recompile the kernel with `time make -j8` - only took < 30 seconds this time because we already compiled most of the kernel and only the changed bits has to be recompiled.

Booting it up again:

```
[    3.358701] Freeing unused kernel image (text/rodata gap) memory: 2032K
[    3.360064] Freeing unused kernel image (rodata/data gap) memory: 1208K
[    3.360381] Hello world!
[    3.360890] Run /bin/bash as init process
```

We see our hello world message in the boot logs, looking nice like the other ones!

## Generating a patch

Say you would like to submit this useless change to the Linux kernel, you must first generate a patch. Luckily, this functionality is already baked into git (because it was kind of invented around linux development!)

First let's branch out from the current branch we are on and commit the change:

```sh
git branch hack
git commit -m'hello world'
```

Note that if you are doing this for real you should have a better commit message that is precise and describes your change.

Now we can generate a patch email:

`git format-patch -1 HEAD`

The `-1` tells git how many commits to include in the patch (in this case 1)

https://www.kernel.org/doc/html/latest/process/submitting-patches.html

It will output `0001-hello-world.patch`, which is the patch file generated:

```
$ cat 0001-hello-world.patch 
From 1929db0d882106de573490b8baaf8164ee62ccde Mon Sep 17 00:00:00 2001
From: John Doe <john.doe@example.com>
Date: Mon, 1 Jun 2022 10:33:21 +0000
Subject: [PATCH] hello world

---
 init/main.c | 1 +
 1 file changed, 1 insertion(+)

diff --git a/init/main.c b/init/main.c
index 65fa2e41a..0421e68a0 100644
--- a/init/main.c
+++ b/init/main.c
@@ -1510,6 +1510,7 @@ static int __ref kernel_init(void *unused)
 	exit_boot_config();
 	free_initmem();
 	mark_readonly();
+	pr_info("Hello world!\n");
 
 	/*
 	 * Kernel mappings are now finalized - update the userspace page-table
-- 
2.36.1

```

Alternatively, if you don't want to generate an email and just want a diff patch, just diff your current HEAD against the previous commit/tag in `git log`:

```
$ git diff v5.17
diff --git a/init/main.c b/init/main.c
index 65fa2e41a..0421e68a0 100644
--- a/init/main.c
+++ b/init/main.c
@@ -1510,6 +1510,7 @@ static int __ref kernel_init(void *unused)
        exit_boot_config();
        free_initmem();
        mark_readonly();
+       pr_info("Hello world!\n");
 
        /*
         * Kernel mappings are now finalized - update the userspace page-table
```

To see how to *actually* submit patches to the linux kernel, see https://www.kernel.org/doc/html/latest/process/submitting-patches.html


## Extra resources

### Links

Some nice tips straight from kernel.log: https://www.kernel.org/doc/html/latest/kernel-hacking/hacking.html

If you want to start making drivers: https://linuxhint.com/linux-device-driver-tutorial/

Github book on linux internals: https://github.com/0xAX/linux-insides

Dive into some (potential) security bugs: https://syzkaller.appspot.com/

High quality linux news and blogs: https://lwn.net/

Mailing lists (Linux development is heavily driven around mailing lists, each sub-project/component has its own list): http://vger.kernel.org/vger-lists.html

Archive of mailing lists: https://lore.kernel.org/lists.html

The Linux Kernel Mailing List (LKML) FAQ: http://vger.kernel.org/lkml/




### Books

Linux Kernel Development by Robert Love

Linux Kernel Networking: Implementation and Theory by Rami Rosen

