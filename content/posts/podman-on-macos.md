---
title: "Podman on MacOS"
date: "2022-10-13"
draft: false
description: "In my use-cases, Podman on MacOS is a viable alternative to Docker Desktop. This is a story about the journey testing it and getting it working with a project we use at work."
---

I just recently got a new laptop at work, a MacBook Pro with the M1Pro chip, and decided that I wanted to give Podman a try as a full-fledged Docker replacement. This is a story about my journey.

This article is going to be long. I'm going to go into a lot of detail about the struggles that I went through in order to get all of this working. It might end up seeming like the ramblings of a madman at times. I'm going to try to summarize the main problems here at the top. 

My main goal for this article is to try to put as much of the random problems that I encountered into a single article, to hopefully prevent those who are trying to do this after me from having to dive into each rabbit hole that I dove into.

## The TL;DR

Without having to bore you with all of the details the TL;DR is that Podman on MacOS does about 99% of what I need it to to use this as a daily driver, and that last 1% is something that I'm quite okay with working around at the moment. The key to preventing most of the problems that I ran into in the section titled “The Journey” is to just make sure that the base image you're using has an arm64-based build in its manifest and any of the binaries you're using are built for arm64.

So for all of this struggle, that single fact - using an image with the correct build architecture - was the solution to 99% of the problems.

## Main Drawbacks

### Volume Mounts Disappearing

The main drawback at this point is that the podman machine volume mounts are only mounted via the `podman machine start` command, so when the vm restarts (whether due to automatic updates or you've installed something that requires the machine to restart) the mounts disappear.  This leads to `Error: statfs: no such file or directory` errors when seemingly nothing on my end changed.

I luckily had a `podman machine ssh` session initiated that I left overnight, and when I came back the next day I was seeing the statfs error because the podman machine auto-updated and rebooted overnight. I also had noticed this functionality a day or two before and [opened an issue](https://github.com/containers/podman/issues/15976) so I happened to already know why this happened.  If I didn't already have some experience with this problem I have no idea how long I would have spent debugging it thinking everything should still be working.

### Volume Mounting a socket

I haven't figured out how to get podman-machine to successfully mount my ControlMaster sockets for SSH, so as I connect through our bastion servers I have to reauthenticate every time. It's not a huge issue for me, though, and it's still better than not having my toolbox container.

### Cross-architecture support for interactive containers

The other main issue that I've noticed is that while you _can_ run amd64-based images by installing `qemu-user-static` on the podman machine host (which was actually how I figured out that the mounts don't get remounted during a machine reboot) if you're trying to do things like set up ssh tunnels and change iptables rules within the container, you'll see error messages like `iptables/1.8.4 Failed to initialize nft: Protocol not supported`. All of this ended up coming down to what I believe to be a difference in architectures between the image and the podman-machine and disappeared once I changed the base image to use an image with arm64 support.

For non-interactive containers (which most use-cases probably are) you probably won't run into these issues.

—

# The Journey

This is the long part.  This is where I might just start rambling. If you just want the pertinent details you can stop reading here.

It all started when I got my new laptop from work. I finally was able to upgrade my laptop, and I chose the MacBook Pro with a M1Pro chip.  This thing is FAST. It's COLD all the time, and one of my main “complaints” that I have is that I can't use the fans spinning down to know when a long-running command is done.

![XKCD #1172 - Workflow](https://imgs.xkcd.com/comics/workflow.png)  
[XKCD #1172 - Workflow](https://xkcd.com/1172/)

As an avid supporter of FOSS products, my first thought was “Let's see if Podman is ready for M1s”.  A little over a year ago when Docker went the route of needing an enterprise license just to do my basic tasks, I had originally gone in search for an alternative. At the time nothing was really “production ready” for the tasks, at least not on MacOS. Podman didn't support being able to mount anything from the `/Users` directory on a Mac, which unfortunately was a hard requirement for me (and probably most other users) at the time.

When I started this undertaking, I had heard a rumor that Podman was now able to support read-only mounts from the `/Users` directory.  This turned out to be wrong.  Podman supports full read-write mounts now!  How happy I was to learn this.  Podman on my Mac seemed like it could be a very attainable goal for me now.

So I install Podman, run through the [Getting Started](https://podman.io/getting-started/installation#macos) guide and run `podman run quay.io/podman/hello`, and am excited to see that it works without any issues. Then, instead of doing something simple like creating a new container from scratch and building on top of it to test everything, I grab a “known good” container configuration that I use regularly as an ephemeral access environment: [ocm-container](github.com/openshift/ocm-container).

`ocm-container` provides our team a shared set of tools in order to support the clusters that we need to support, in a mostly-ephemeral environment to protect customer data by not persisting it past the container exiting. However, ocm-container's base image did not at the time have an arm64 build. This, though, wasn't apparent.

So as I go to first run `podman build .` I ran into several issues.  The first was that it couldn't find my `pwd`. This was a pretty simple issue to fix, though it wasn't apparent at first. I first thought “How does it _not_ see my `pwd`? I'm literally looking at it right now.”  But this is due to the architecture of podman-machine on MacOS.  Because Podman is made to run on Linux, we need to create a vm with `podman machine init` first. With docker-desktop, `/Users` is automatically mounted in the docker-machine without any user input. The same is not true in Podman, so we need to run the machine init command with a flag in order to mount the volume.  So for me it looks something like `podman machine init -v ${HOME}:${HOME}`. Now podman can find the directory I need it to find, because it's mounted on the machine itself.

The next problem I had trying to build the container was the lack of support for cross-architecture builds.  These specifically presented as issues trying to download some binaries for arm64 as they're just not available. I was able to eventually figure out that I needed to install `qemu-user-static` in the podman-machine with the following commands:

```bash
$ podman machine ssh
> sudo rpm-ostree install qemu-user-static
> sudo systemctl reboot
```

With that problem out of the way I go to run the `podman build` command again, and it fails with the same `statfs: file not found` error!  One step forward and two steps back.  A bit of debugging later, and come to find out that the volume mounts are created by the `podman machine start` command via SSH commands into the vm, so if you reboot the vm outside of `podman machine stop && podman machine start` it will not re-mount the volumes inside the vm. I [opened an issue](https://github.com/containers/podman/issues/15976) with the podman team about this. We're currently working together to move this functionality into the `machine init` command.

So with that out of the way, and now the container builds (albeit with a bunch of warnings about `WARNING: image platform ({amd64 linux  [] }) does not match the expected platform ({arm64 linux  [] })`).  But yes, this is progress! I've only spent about a day or two debugging this at this point. Until I go to log in, and am greeted by `exit status: 99` when trying to SSH into a bastion.  Okay, well it's the end of the workday, I'll tackle this tomorrow.

The next day comes, and I'm ready to dig back in. The first thing I wanted to tackle, though, was the issue with the image platform not matching the expected platform.  Can I get a native arm64 build?  So I check the manifest of the image that we're using as the base - it doesn't have an arm64 option.  So I change that to start with another minimal base image that has an arm64 option.  That error message disappears… at first.  It sneaks back in though.  Why? Honestly, I'm not entirely sure. I think it's due to our multi-stage build process, as we have some internal tooling that gets built on top of the public ocm-container build after the fact.

This multi-stage build process and these architecture issues start to really come to light when we try to use the internal tool that we built, and are greeted with just `<nil>` as output from the command.  I can't seem to reproduce this again, which is the strangest thing. Normally when I try to run an x86_64 binary on the m1 you'd see an error like `cannot execute binary file: Exec format error`, but we're just getting `<nil>`.  If I move the binary out of the container to my local machine I see that `Exec format error` but I can't definitively say why, as we're running a Linux container on MacOS, so it's expected that it won't work on both my host and inside the container.

The internal tool is just wrapping sshuttle for us, so let's turn on verbose mode on `sshuttle` and oh, this looks like it might be relevant: `iptables/1.8.4 Failed to initialize nft: Protocol not supported`.

Y'all, I can not tell you how much time was wasted at this point trying to go down this rabbit hole. Just about Every…Single…Article… that I read trying to debug this part said something about rebooting the machine because this is due to a kernel update. Well, I'm in a container, so that's almost _entirely_ useless to me. A few articles suggested downgrading iptables because now iptables uses nftables under the hood. 

On Debian-based systems this is accomplished by just installing `iptables-legacy` but being RHEL-based I had to uninstall iptables and then target a specifically older version.  This led to a new error message I went down a pretty deep rabbit hole trying to figure out: 

```
iptables v1.6.2: can't initialize iptables table `nat': iptables who? (do you need to insmod?)
Perhaps iptables or your kernel needs to be upgraded.
```

I've been spending almost my entire workday on this for about 4 whole working days at this point, and was ready to give up.  In fact, at some point I had given up on Podman and installed Docker just to be greeted with a bunch of the same build errors I got before installing `qemu-user-static`, with even less of a path forward to figure out how to get that working on Docker.  Another attempted step forward but a few steps backwards.

So I disengaged from this iptables stuff to try to get my mind right. I'm starting to feel serious imposter syndrome at this point, and have had to revert back to my old laptop to help people get their jobs done because I just can't use this new laptop for anything but meetings at this point. Trying to support my team _and_ debug my own problems at this point is taking a huge mental toll on me.

The next thing I wanted to focus on was getting this internal tool working, as this tool automagically sets up all of our necessary ssh tunnels and handles logging us into particular clusters, so it's probably the most important of all of the tools in this container. It's also now where I decide that I'm going to start from as much of a scratch container as I can, so I rip out everything but the bare minimum packages I need out of my main docker build.

I decide that I'm going to rip out everything, — like, EVERYTHING — and hand install what I need into the container. We're going to do this the hard way and hit every single error and tackle it in real time and _THEN_ try to fix the automated build. (Honestly, this is what I should have done from the start).  I get everything in, and EVERYTHING WORKS!?!?!  Well, SSH is still failing, but hey, at least I can get these commands all running.

I build this container and as I'm looking at the Dockerfile I realize that all of the binaries that I'm downloading to get this working are downloading as amd64 or x86_64 and not arm64.  So I start looking for pre-built binaries and find that only a few of them actually have the prebuilt binaries that I need.  So I start adjusting the Dockerfile to use the pre-built arm64 binaries where appropriate, trying to change as much as I can to be referenced from a build-arg so that I can just try to change this on the fly.

At this point I'm starting to get excited again. I have binaries working, I have mounts working, and at this point it's well past the end of my work day but I'm working with one of my amazing coworkers who's located in Hawaii who's taken quite the interest in my struggles. I've again switched to docker from podman and get to the point where I'm trying to start my ssh tunnels. I'm no longer using ANY of the existing ocm-container code at this point.  The error that I get at about 6pm local time for me: `iptables v1.8.4 (nf_tables): Could not fetch rule set generation id: Permission denied (you must be root)`.  Ugh, my brain is mush for fighting this issue for over a week at this point. I'm root in the container! How can I run the command as root when I'm already root?!?!?!

> Chris  5:53 PM  
> can you try adding --cap-add=NET_ADMIN when you run the container, and see what happens?  
> Kirk  5:55 PM  
> That might have done it!

``` # /usr/bin/sshuttle
Starting sshuttle proxy.
…
c : Connected.
```

It was as simple as me forgetting to port the `--privileged` argument at this point.  It's finally all starting to come together. I stop docker, start podman, run the build and I get the same result!  I can tunnel. I can log in. 

At this point, I've been debugging this off-and-on for almost a week and a half. There's finally a light at the end of the tunnel. I branch ocm-container and start re-adding everything I can back to the Dockerfile in order to get it working. I write a quick “how to” for my co-workers who want to also get this running for themselves.

It's kind of depressing to see a whole week of work boil down to a very small few changes, but hey, sometimes it's about the journey.  I know way more about podman internals now than I did a few weeks ago. I know a lot more about arm/amd architectures and some of our internal build processes from upstream engineering teams. I've put in the DAYS of pain so that those that come after me don't have to.

There's still a lot of work to get done to get ocm-container into a place where it's just a drop-in install for our team to use, but that will come with time.  As for now, if you're running an M1 Mac I've added some easy-enough install steps to follow. 

If you've gotten to this point, thanks for reading.  Hopefully you don't have to ever dive down these rabbit holes, but you take away anything from this, I have two key points for you to take away:

1. Podman will almost probably suit your needs as a daily driver on MacOS
2. When something doesn't work on arm64 that works on x86_64, try starting from as minimal as possible so you don't waste your time digging down arcane rabbit holes

Thanks again for reading.

