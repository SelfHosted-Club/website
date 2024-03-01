---
title: Optimising power consumption with cpupower
description: "Unravel the power of cpupower in Linux! Learn its history, how to use it effectively, and understand the various CPU governor states."
date: 2023-07-26T09:00:38.993Z
last_updated: 2024-02-27
preview: /img/banners/CPUPower_banner.jpeg
image: /img/banners/CPUPower_banner.jpeg
tags: [Linux, General]
categories: [General]
---

## Optimising power consumption with cpupower

There are several ways that I try to watch and reduce the electric consumption of the devices in my home lab, some are upgrading the devices to more efficient ones, but that pretty much only happens when the device I have is completely gone and can't be fixed, even then it is mostly the parts themselves that are being replaced or upgraded.

One of the things I started lately implementing, and it is a shame I did not do this much much earlier - using [cpupower package][cpupower] - something that has been around for a while.

cpupower is a linux package that allows you to optimize your system's power management. It is a Linux utility tool, part of the kernel itself, which is used to fine-tune CPU settings on managing power consumption and performance.

I did research a bit of history as I always do on everything that I use and the power management infrastructure in the Linux kernel has evolved quite a bit over time. In the early days, CPU frequency scaling was managed by individual programs like `cpufreqd`, `cpudyn`, and `powersaved`. But as the kernel evolved, these separate utilities were unified and migrated into the kernel, which provided better efficiency and control. Eventually, this coalesced into what we now call `cpupower`.
<!-- FM:Snippet:Start data:{"id":"Affiliate warning","fields":[]} -->
> Some links may be affiliate links that keep this site running.
{: .prompt-info}
<!-- FM:Snippet:End -->

### Getting hands on with cpupower

First thing first, you need to have cpupower installed on your system. For most Linux distributions, the command would be something like this on debian systems:
<!-- FM:Snippet:Start data:{"id":"Shell code block","fields":[]} -->
```shell
sudo apt install cpupower
```
{: .nolineno }
<!-- FM:Snippet:End -->

and on Ubuntu systems you'll need to run:
<!-- FM:Snippet:Start data:{"id":"Shell code block","fields":[]} -->
```shell
sudo apt install linux-tools-common linux-tools-`uname -r`
```
{: .nolineno }
<!-- FM:Snippet:End -->

Once you have `cpupower` installed, let's check the current frequency settings using the following command:
<!-- FM:Snippet:Start data:{"id":"Shell code block","fields":[]} -->
```shell
cpupower frequency-info
```
{: .nolineno }
<!-- FM:Snippet:End -->
This command will display a wealth of information, including the driver in use, the current and available CPU frequencies, the governor in use, and more.

![CPUpower frequency info](/img/posts/cpupower/cpupower_frequency_info.png)
_CPUpower frequency info_

<!-- FM:Snippet:Start data:{"id":"VPS Links","fields":[]} -->
<div class="alert alert-info" style="background-color: lavender; border: none; color: black;">
  <!--<i class="fas fa-info-circle"></i><br>--> I host majority of my cloud instances on <a href="https://cloud.hosthatch.com/a/3785" target=_blank>HostHatch VPS (Virtual Private Server) Instance</a> (In Asia) for a steal. Some of the other hosts I use are <a href="https://my.racknerd.com/aff.php?aff=9825" target=_blank>RackNerd</a> (US) and <a href="https://clients.webhorizon.net/?affid=27" target=_blank>WebHorizon</a> (Asia+Europe) VPS, and decided that it is time to move away from Linode - which is a Great service, but I am looking to reduce the billing on instances. For comparison, I save more than 50% on <a href="https://cloud.hosthatch.com/a/3785" target=_blank>HostHatch</a> compared to <a href="https://www.linode.com/lp/refer/?r=1ba3d23e62b7803c1d317de14a21a9f96a9a1b97" target=_blank>Linode</a> ($3.33 compared to $8) - Don't get me wrong, if this was an extremely (like REALLY) critical application, I would keep it on <a href="https://www.linode.com/lp/refer/?r=1ba3d23e62b7803c1d317de14a21a9f96a9a1b97" target=_blank>Linode</a>.
</div>
<!-- FM:Snippet:End -->

The Governors’ Conference: Understanding The Different States (I tried to make a joke here)
When it comes to CPU frequency scaling, the different settings, or "governors", determine how your CPU will adjust its frequency in response to the system load. Each governor follows a different policy which suits different scenarios. Let's take a look at these governors:
- **Performance:** This governor sets the CPU to its maximum frequency, ensuring the best possible performance. It is perfect for high-demand tasks, like gaming or video editing, but it consumes the most power.
- **Powersave:** Quite the opposite of the Performance governor, the Powersave governor sets the CPU at its lowest possible frequency, thus conserving power. However, this comes at the expense of performance.
_ **Userspace:** This is an interesting one! The Userspace governor allows the user to set the CPU's operating frequency manually. This provides the most control but requires a deep understanding of your system to use effectively.
- **OnDemand/Conservative:** These two governors dynamically adjust the CPU frequency based on the system load. The OnDemand governor rapidly increases the frequency when the load goes up, providing a balance between power saving and performance. On the other hand, the Conservative governor is more cautious and gradually scales up the frequency, focusing more on power saving.
- **Schedutil:** The newest kid on the block, the Schedutil governor, utilizes the kernel's scheduler utilization data to make its frequency decisions, aiming for a balance between power and performance.
To change the governor, you can use the cpupower command followed by the frequency-set option and -g flag. For instance, to set the governor to performance mode, you would use:

To change the governor, you can use the cpupower command followed by the frequency-set option and -g flag. For instance, to set the governor to performance mode, you would use:
```shell
sudo cpupower frequency-set -g ondemand
```
{: .nolineno }
[cpupower]: https://linux.die.net/man/1/cpupower
