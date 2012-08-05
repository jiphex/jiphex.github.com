---
title: Controlling a Service with Runit
layout: post
---

[runit](http://smarden.org/runit/) is <q>a cross-platform Unix init scheme with 
service supervision, a replacement for sysvinit, and other init schemes.</q>.

Quite simply, it can make a program run on boot, and make sure that it gets 
automatically restarted on death.

The Runit daemon can be run in two modes, either as PID 1, controlling all 
processes on your system, or, like we're using it, running as a normal PID and 
forking sub-processes as needed. This post will not cover replacing SysV Init 
with runit.

    [james@server] $ pstree -Aap
    init,1
    |
    ... abridged
    |
    |-runsvdir,2943 -P /etc/service...
    |   |-runsv,2944 xkcd-pwgen
    |   |   |-node,2947 /usr/local/bin/xkcd-pwgen.node.js
    |   |   |   `-{node},2952
    |   |   `-svlogd,2946 -tt /var/log/xkcd-pwgen
    |   `-runsv,2945 git-daemon
    |       |-git-daemon,2949 --verbose --reuseaddr
    |       `-svlogd,2948 -tt /var/log/git-daemon
    
This output shows a section of `pstree`'s output, showing runit controlling a 
pair of services on my server (an [XKCD #936](http://xkcd.com/936/) password
[generator](http://drax.tlyk.eu:1337) and the `git` daemon).

As long as you have a program which runs on the command-line (in the 
foreground), it's trivially easy to make Runit daemonize it and keep it running:

1. Install `runit`, it's a package on Debian in the main repository.
2. Create a directory in `/etc/sv/` named after the service you're going to 
control, e.g `/etc/sv/someapp`.
3. In that directory, create a script named `run`. This will usually be only a 
few lines, and must do the following:
    * Control any UID/GID changes, with sudo or chest
    * Start your process and run it in the foreground
    * **Fork** to the application, i.e by running `exec`
    
    A sample script for running a Node.js app (such as the password server 
    above) is here:
    
        #!/bin/sh
        exec 2>&1
        exec chpst -u nobody /usr/local/bin/xkcd-pwgen.node.js
        
    Make the script executable. You can test it by running the script directly
    on the command-line.
4. Create a symlink in `/etc/service` which links back to the `/etc/sv/` 
directory. This "activates" the service, and means you can disable the service 
by just removing the symlink.
5. That's basically it, the service should just be running. You can check it
with the 'sv' utility:
    
        sv s someapp # (status) shows the service
        sv u someapp # (up) starts the service
        sv d someapp # (down) stops the service

    
Logging
-------

Runit comes with built-in support for logging the stdout and stderr of your 
process, including log rotation and potentially even sending the logs to a
remote syslog receiver.

Adding logging to your service control scripts is also very simple:

1. Create a subdirectory under your someapp directory in /etc/sv named `log`.
2. Add a run script inside the log directory, this will control the logging 
daemon and should follow the same rules as described above. You'll probably
want to use **this** run script, which uses runit's own log daemon:

        #!/bin/sh
        exec svlogd -tt /var/log/xkcd-pwgen
    
    This will send stdout and stderr to `/var/log/xkcd-pwgen/current`, and
    automatically rotate the files according to runit's slightly arcane rules.
    
    More information on `svlogd` can be found [here](http://smarden.org/runit/svlogd.8.html).

## Debugging

It's mildly difficult to debug what's happening with Runit. Really, you'll need 
to check both the logged output of stdout/stderr, but if the process won't 
start, there may never be any. You can get extra information from the process 
description reported by `ps`, and obviously by trying to execute the `run`
script yourself on the command-line and observing the output.