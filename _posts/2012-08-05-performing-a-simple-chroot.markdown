---
title: Performing a simple chroot
layout: post
---

# {{page.title}}

Figured I'd write this as an experimental howto on a task which is (I think...) often overlooked by a lot of sysadmins as being particularly important, which is in fact pretty useful for the odd problem.

When you run the command `chroot directory`, your shell will attempt change it's  root directory (previously "/") to whatever's supplied as `directory`. This means, in effect, that it will attempt to execute whatever your shell is (e.g `/bin/bash`) from inside the chrooted directory, e.g directory/bin/bash.

The shell itself, and any child processes spawned from it will operate solely inside the directory you provided, and will not have access to the rest of the server's filesystem.

Obviously a valid shell binary must exist within directory, as must a whole set of other files (libraries, other binaries etc). It's not possible to chroot into an empty directory, it'd be as sane as trying to boot up an empty root filesystem.

Normally, when you actually do chroot, it'll be into a pre-existing filesystem, for example a pre-installed Linux server's hard disk which has been already mounted under a directory. That's the case we'll be documenting here.

I'm assuming here for the purposes of this post, that you've got a physical linux server with a problem which stops it from booting, for example, someone has managed to mess up grub, the fstab or the bootloader somehow.

## Steps

1. First of all, we'll need to boot the server off another medium, perhaps a Linux LiveCD, or a network boot environment provided by your host. You'll need to make sure that whatever you boot is at least the same architecture as the system you're going to chroot into, meaning that if you have a 64-bit system, a 32-bit network boot will not be sufficient. I'll explain why shortly.
2. Once the server's booted into the netboot/LiveCD, you should be able to access the server's real disks under /dev/sdX (or /dev/cciss/cNdXpY for a HP server). You'll want to create a folder in the netboot environment (this will be in memory), such as "/target". You could also use /mnt.
3. You'll then need to mount the server's original disks under the directory. I'm assuming the following partition layout on the original server's disks:
   <code><pre>
    /dev/sda1    ext3 Boot partition
    /dev/sda2    swap partition
    /dev/sda3    extended partition (not a real partition)
    /dev/sda4    root filesystem (large)
   </pre></code>

    If your server's partitioned like this, I'd mount it as follows:

    <code><pre>
    mount /dev/sda4 /target
    mount /dev/sda1 /target/boot
    </pre></code>

    You can ignore the swapfile and extended partitions. If you've got any other partitions, such as a separate /home or /var partition, they can be mounted in the appropriate places like /boot.
4. You should now have an ok-ish looking chroot under `/target`, and presuming the mounts worked ok, you should be able to browse your files as normal (e.g check that `/target/etc/hostname` matches what you'd expect). 

    Before you carry on, there's something else important to note. Modern Linux distributions use a number of special filesystems which aren't stored on disk to access configuration for hardware, and for various internal kernel functions. The first of these two, /proc and /sys will need to be mounted as follows:
    
    <code><pre>
    mount -t sysfs none /target/sys
    mount -t proc none /target/proc
    </pre></code>
    
    These will be needed if you're trying to do anything hardware specific once you're in your chroot (e.g fixing grub), without them the various utilities will fail to work.

    Another important filesystem is /dev, it's not as simple to mount this as /proc or /sys, instead of mounting "none" with a special type, we'll just perform a bind mount to the real /dev of the netboot, as it should be good enough for what we need:

    `mount -o bind /dev /target/dev`

5. Everything you need should now be mounted, you can now try and perform the chroot:chroot /target

    Your shell should return straight away with another prompt. If it does, congratulations, you're now chrooted into the target system and can pretty-much use it as you would if the real system had booted. You can modify /etc/fstab, run "update-grub" in Debian, and do what you need to fix the problem at hand.
If you got a message about not being able to execute the shell, or something along the lines of "wrong ELF class", you're probably trying to chroot into a 64-bit install from a 32-bit netboot or LiveCD kernel. Anything else and you probably messed up the mount somehow.
To exit, press Ctrl+D or type 'exit' to terminate the shell.
Further Topics

While this covers chrooting into an already-installed system, in Debian it's possible to create a barebones chroot in a directory using debootstrap. It's a great tool for creating scratch chroots.

The package schroot will allow you to share a number of chroots between normal (non-root) users on your server, this can be used for a number of useful purposes