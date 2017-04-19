Title: Performance monitoring of Vdsm
Date: 2017-04-19 09:12
Modified: 2017-04-19 16:55
Category: oVirt
Tags: oVirt, Vdsm, release-4.1
Slug: performance-monitoring-of-vdsm
Authors: Francesco Romani
Summary: Monitoring the performance of the Vdsm monitoring

# Monitoring the performance monitor of Vdsm

Vdsm, the [oVirt](http://www.ovirt.org) node management daemon, provides
monitoring and telemetry reporting for the host on which runs, and for the VM
which manages. Up until version 4.1, included, Vdsm includes custom-built monitoring
code. In 4.2 and beyond, we are transitioning to [collectd](http://www.collectd.org).

In this document we'll see how this switch affects the performance of Vdsm, and what
improvements are in the pipeline.

## What Vdsm monitors

There are four major monitoring duties that Vdsm performs.
While all four are important, the performance cost of all of those varies greatly.

1. monitoring of the VMs - for external consumption.
   Provide metrics about the hypervisor resource consumption, both physical and virtual.
   e.g. real CPU usage per-hypervisor, VCPU usage inside the hypervisor, RAM used by the
   hypervisor, network traffic per-hypervisor. This task is done by [libvirt](http://www.libvirt.org),
   Vdsm just consumes the result of one API call. The metrics are exported to the external clients (like
   time-series DB, oVirt Engine, oVirt data warehous).

2. monitoring of the VMs - for *internal* consumption.
   Vdsm also monitor some key VM metrics to implement some of its flows. The most notable
   of those flows is the automatic extension of thin-provisioned disks. This gives the
   illusion of fully provisioned disks for VMs, while actually they are thin-provisioned
   and extended on the fly when the space is used, without any external client intervention.
   To do so, Vdsm continuously monitor the space consumption of thin-provisione disks,
   but this metric is used only internally.

3. monitoring of the Host.
   Provide the metrics about the host performance and resource allocation, like cpu usage,
   memory consumption, network traffic, hugepage allocation, numa statistics.
   Those stats are gathered by Vdsm itself scanning files under the linux procfs or sysfs.

4. Vdsm health-check.
   Vdsm reports metrics about its own behaviour and health, like its CPU and RAM consumption,
   number of active threads, liveness of the libvirtd connection.
   It is of course debatable that Vdsm reports its own health: if Vdsm becomes unresponsive, or
   slow, the report could be missing or worse misleading.


## Replacements for the Vdsm monitoring

The collectd projects offers a good alternative for the Vdsm monitoring.
Collectd is a mature and extensible project with a great array of plugin to collect a lot of metrics.

Collectd version 5.7.0 and onwards offers straightforward replacement for the monitoring areas #1 and #3.
However, we do have some gaps between the collectd virt plugin and the metrics reported by Vdsm.
The author submitted some patches to bring the collectd virt plugin up to speed, and those enhancement
are expected to be delivered in collectd >= 5.8.0.
Furthermore, [a new collectd plugin is available](https://github.com/fromanirh/collectd-ovirt)
rewritten (almost) from scratch with a clean and modern codebase. Unfortunately this plugin is not suitable
for submission upstream, being a complete rewrite of it.

libvirt >= 3.2.0 gained support for disk threshold events: libvirtd can emit one event if the disk
usage exceeds a configurable threshold, so we can leverage this in oVirt >= 4.2 ang replace the
monitoring area #2, which is currently implemented polling each thin-provisioned disk of each VM.

Finally, the extensible nature of collectd allows us to replace the self-health Vdsm monitoring with
[minimal glue code](https://github.com/fromanirh/procwatch), avoiding the need of a custom built plugin
with a very narrow use case. This plugin, and the detailed monintoring flow, will be described in a future
post.

## Test configuration and methods

All tests will run on one up-to-date RHEL 7.3 (you can get identical result with CentOS 7.3) installed
on a box with one core i7 3770 CPU and 32 GiB of RAM.

The software stack is the aforementioned stock RHEL 7.3 plus:

- collectd 5.7.0 from sources + backport of the virt plugin from master branch

- procwatch version 8e579d0

- a snapshot of the last Vdsm from the 4.1 branch, which will become 4.1.2

- oVirt Engine version 4.0.6

- collectd 5.7.0 + virt plugin backported from master (no further changes)


The test consist in evaluating the resource consumption of the software stack while running a medium-to-heavy
load of 200 VMs on a single host. We don't care about what the VM are doing, we just want to assess how our
stack. So we will run very simple VMs, each with 16 MiB (yes, sixteen megabibytes - we use such low RAM to
pack as many VMs as possible), 1 vcpu, 1GiB ISCSI disk, no OS running thus no oVirt Guest Agent.

Again, such minimal VMs are used only because resource constraints. The test is still significant to assess
the performances of the host side.

We will evaluate the [long term load average](https://en.wikipedia.org/wiki/Load_(computing)) and the CPU consumption
of the processes.
In each of the following graphs, if not otherwise specified, we will have

- the time of the day on the X axis

- the CPU load, percentage, on the *left* Y axis

- the amount of the load average on the *right* Y axis

The color code of the software components is:

- green line: libvirt graph

- red line: Vdsm graph (unfortunately in one graph we will see also one turquoise line)

- brown line: Collectd graph (where relevant)

- purple line: load average graph

The test starts around the 11:25 local time, when we boot the VMs. Around the 11:30 local time the system
is stable and the measuremenents begin.


## Experiment 1: the simplest approach, Vdsm and collectd side by side

![figure 1: baseline, collectd and vdsm side by side]({filename}images/perf-mon-fig1.png)

[download the full image](images/perf-mon-fig1.png)

The first 30m (11:30 -> 12:00) are the evaluation of the stock configuration.
We have out-of-the-box Vdsm, and collectd is just monitoring the host (no vm monitoring,
virt plugin unloaded).

Libvirt load is ~14% (green line)
Vdsm load is a bit higher, ~16% (red line)
UNIX load is ~0.6 (light purple)
collectd load is ~0

This is the baseline reference load and the benchmark for all the coming experiments.

Then we test the simplest possible upgrade path: just install and run collectd alongside Vdsm.
Around 12:00 we enabled virt plugin and restarted collectd. The collectd
dark purple line disappears (PID change), the new line is the brown one.
Vdsm load is unaffected, as expected.
Collectd CPU load becomes noticeable, floating behind 2 and 3% CPU load
Libvirt load jumps to a bit less than 30%. As expected, we have libvirtd doing twice the work,
because both collectd and Vdsm duplicates the work.

As expected, doing twice the work does not come for free on the libvirtd side, so the simplest
approach is not really viable.

## Experiment 2: the impact of the disk usage polling

![figure 2: avoiding the disk usage polling]({filename}images/perf-mon-fig2.png)

[download the full image](images/perf-mon-fig2.png)

Around the 13:03 localtime we disabled the the high water monitoring; we also disabled again
the VM monitoring in collectd (the virt plugin).
This load simulate the impact of the switch of the new high water event, solving the monitoring
task #2 of Vdsm described above; it is a good estimation of how oVirt 4.2 with libvirt >= 3.2.0
could look like.

Considering the baseline load (from ~11:35 to ~12:05), we see
approximatively -50% of the libvirt load and -30% of the Vdsm load.
This strongly suggests that the switch to the libvirt disk usage notification will be really a
big win performance wise.

Please note that after 13:03 the Vdsm line switches from red to turquoise. This glitch is fixed in the following
graphs.

Please note that from now on the high water mark is *DISABLED* on all
the upcoming experiment, assuming Vdsm will use the high watermark event,

## Experiment 3: moving all the monitoring to collectd

![figure 3: moving all the monitoring to collectd]({filename}images/perf-mon-fig3.png)

[download the full image](images/perf-mon-fig3.png)

We can now observe what happens if we outsource the VM and host monitoring to collectd.
(the monitoring tasks #1 and #3 described above)

From ~14:05 we have:

- Vdsm does not monitor anymore not the VMs nor the host (nor the high water for disks)

- Collectd uses the upstream virt plugin.

The collectd configuration is

`/etc/collectd.conf`:
~~~
LoadPlugin syslog
LoadPlugin cpu
LoadPlugin interface
LoadPlugin load
LoadPlugin memory
Include "/etc/collectd.d"
~~~

`/etc/collectd.d/virt.conf`:
~~~
LoadPlugin virt
<Plugin virt>
    Connection "qemu:///system"
    Instances 5
    ExtraStats "disk pcpu"
</Plugin>
~~~

The libvirt load is a little higher than the baseline, on average around 18% now over 15% baseline
Vdsm load drops to ~5%, while the collectd load is in the ~3% range.
(*PLEASE NOTE* those are estimations not actual averages)

The overall resource consumption of the stack is a little lower than the
baseline, with further room for optimzation. It is worth to note that the unix system load increases;
further investigation is needed to explain this increase.

## Experiment 4: alternative collectd virt plugin

![figure 4: alternative collectd plugin]({filename}images/perf-mon-fig4.png)

[download the full image](images/perf-mon-fig4.png)

We now try out the [out of tree virt2 plugin](https://github.com/fromanirh/collectd-ovirt) plugin,
configured like that:
`/etc/collectd.d/virt2.conf`
~~~
LoadPlugin virt2
<Plugin virt2>
	RefreshInterval 15
	Connection "qemu:///system"
	Instances 5
	ExtraStats "pcpu disk"
</Plugin>
~~~

Of course the upstream `virt` plugin is disabled - only one between `virt` and `virt2` could run
at any given time.

Around the ~14:40 mark we switched the collectd plugin and restart collectd
We can observe the spiky load profile typical of when we use bulk stats
from a single thread. The lLibvirt load is roughly cut in half, while the collectd load
slightly increases. This could be one byproduct of the measurement, the expected load profile
should be spikes of high load and near zero load between spikes.

We have room to do some optimizations here to shave some more load, but
the overall improvement is already there and the UNIX system load
(purple line) confirms that.

In the last experiment, started around the 15:00 mark, we relaxed the
collectd interval from the default 10s to the Vdsm default 15s, gaining
a bit more performance with no regression to our current state.

## Summary and takeaways

![figure 5: summary]({filename}images/perf-mon-fig5.png)

[download the full image](images/perf-mon-fig5.png)

To summarize and wrap up, we now provide one summary of the load measured
during the experiments.

The bars are the cpu percent for the components, while the line is
the UNIX load average.

*PLEASE NOTE* that The averages are *not* exact averages, but rather extrapolation from
the graphs; yet this is a good enough approximation.
However, this is just one initial exploration study, and those results needs to be
reproduced and extensively tested in more test conditions.

Please note the last column is one estimate of the load using the
upstream virt plugin relaxing the sampling interval from 10s (collectd
default) to 15s (vdsm default).
A ~10m test was run to gather some data.

The takeaways from this test run:

1. adoption of the disk usage notification event from libvirt offers a significant performance benefit,
   and comes for free in libvirt >= 3.2.0

2. the aformentioned event, plus the adoption of collectd for monitoring makes the oVirt stack
   gain ~33% of system performance (cumulative load from ~30% to ~18%).

3. there is low-hanging further optimizations, early quick tests suggest that the oVirt stack can
   reach ~50% system performance improvent (cumulative load from ~30% to ~16%) using
   code which is ready *today*, like the virt2 plugin. More to come in a future post.

4. No accurate profiling or optimization was recently made on the libvirtd or collectd side,
   so more performance gains are definitevely possible.
