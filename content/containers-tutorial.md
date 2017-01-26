Title: Container support for oVirt
Date: 2017-01-26 10:52
Modified: 2017-01-26 10:52
Category: oVirt
Tags: oVirt, Vdsm, containers, release-4.1
Slug: containers-in-ovirt
Authors: Francesco Romani
Summary: How to try the experimental container suppiort for oVirt

# How to try the experimental container support for Vdsm.

In oVirt 4.1, Vdsm (4.19) gained the *experimental* support to run containers alongside VMs.
Vdsm had since long time the ability to manage VMs which run containers,
and recently gained support for
[atomic guests](http://www.projectatomic.io/blog/2015/01/running-ovirt-guest-agent-as-privileged-container/).

With the new support we are describing, you will be able to manage containers
with the same, proven infrastructure that let you manage VMs.
The scope of the further development depend on the interest on this feature.

## What works, aka what to expect

You can run any docker image on the public docker registry.
The ability of using file-based storage for persistent volumes is currently under work.

## What does not yet work, aka what NOT to expect

Few things are planned and currently under active development:
1. Monitoring. Engine will not get any update from the container besides "VM" status (Up, Down...)
   One important drawback is that you will not be told the IP of the container from Engine,
   you will need to connect to the Vdsm host to discover it using standard docker tools.
2. Proper network integration. So far the feature relies on existing docker network configuration,
   which requires manual intervention and/or settings outside oVirt. We are currently evaluating
   the introduction of a new [docker network plugin](https://gerrit.ovirt.org/#/c/67229/).

## 1. Introduction and prerequisites

Trying out container support affects only the host and the Vdsm.
Besides add few custom properties (totally safe and supported since early
3.z), there are zero changes required to the DB and to Engine.
Nevertheless, we recommend to dedicate one oVirt 4.1.z environment,
or at least one 4.1.z host, to try out the container feature.

To get started, first thing you need is to setup a vanilla oVirt 4.1.z
installation. We will need to make changes to the Vdsm host,
so hosted engine and/or oVirt node may add extra complexity,
better to avoid them at the moment.

The reminder of this tutorial assumes you are using two hosts,
one for Vdsm (will be changed) and one for Engine (will require zero changes);
furthermore, we assume the Vdsm host is running on CentOS 7.y.

We require:

- one test host for Vdsm. You will need to configure ahead of time the docker networking as you see fit.

- docker >= 1.10

- oVirt >= 4.1.0 (Vdsm >= 4.19.1)

- CentOS >= 7.3


### 1.1. Note if you use packages from docker.com

Up to date docker packages for centos are available [here](https://docs.docker.com/engine/installation/linux/centos/).

Caveats:
1. docker from official rpms conflicts con docker from CentOS, and has a different package name: docker-engine vs docker.
   Please note that the kubernetes package from CentOS, for example, require 'docker', not 'docker-engine'.
2. you may want to replace the default service file
   [with this one](https://github.com/mojaves/convirt/blob/master/patches/centos72/systemd/docker/docker.service)
   and to use this
   [sysconfig file](https://github.com/mojaves/convirt/blob/master/patches/centos72/systemd/docker/docker-engine).
   Here I'm just adding the storage options docker requires, much like the CentOS docker is configured.
   Configuring docker like this can save you some troubleshooting, especially if you had docker from CentOS installed
   on the testing box.

## 2. Augment Vdsm to support containers

Make sure you have the `vdsm-containers` package installed, and that the docker service is running.

to install the `vdsm-containers` package you are gonna need to run something like.

    
    # rpm -ivh vdsm-containers-4.19.2-5.gitfcfb64d.el7.centos.noarch.rpm
    

make sure Engine still works flawlessly with patched Vdsm.
This ensure that no regression is introduced, and that your environment can run VMs just as before.
Now we can proceed adding the container support.

If you don't have already, install docker:

    
    # yum install docker
    

start the docker service

    
    # systemctl start docker
    

most likely, you want to make sure

    
    # systemctl enable docker
    

As last step, restart *both* Vdsm and Supervdsm,

    
    # systemctl restart supervdsmd vdsmd
    

Now we can check if Vdsm detects docker, so you can use it:
still on the same Vdsm host, run

    
    $ vdsClient -s 0 getVdsCaps | grep containers
        containers = true
    

This means this Vdsm can run containers!

Now we need to make sure the host network configuration is fine.

### 2.1. Configure the docker network for Vdsm

Currently, the configurability of the networking of container run through Vdsm is very limited.
Vdsm will put all the containers it runs in the network configured in /etc/vdsm.conf.
Check the `network_name` option in the `containers` section.

Please note that the network name you configure must be the network name of the container engine.
In other words, you should use the *docker network name*, as you see it in the output of `docker ls`.

Example:

    
    $ docker network ls
    NETWORK ID          NAME                DRIVER
    d807ac6fa7b7        bridge              bridge
    f962d0d1018a        none                null
    752accd64c06        host                host
    

Valid names in this example are `bridge` `none` and `host`. For obvious reasons, the only real option
is `bridge`.

You must configure the docker network ahead of time using the
[standard docker](https://docs.docker.com/engine/userguide/networking/)
[network tools](https://docs.docker.com/v1.5/articles/networking/).

One simple and effective way to configure docker is to dedicate one NIC in your host to docker and
to use the [macvlan driver](https://docs.docker.com/engine/userguide/networking/get-started-macvlan/).
Please note that the macvlan driver was included in docker >= 1.12.

*CAUTION* oVirt still needs its networking settings, most notably the `ovirtmgmt` network available
to interact with the host.

At the moment, there is no way to configure the docker networking through the Engine UI, or to integrate
into the oVirt networking. The later option is being considered as future extension.

## 3. Configure Engine

As mentioned above, we need now to configure Engine. This boils down to:
Add a few custom properties for VMs:

In case you were already using custom properties, you need to amend the command
line to not overwrite your existing ones.

    
    # engine-config -s UserDefinedVMProperties='volumeMap=^[a-zA-Z_-]+:[a-zA-Z_-]+$;containerImage=^[a-zA-Z]+(://|)[a-zA-Z]+$;containerType=^(docker|none)$' --cver=4.1
    

It is worth stressing that while the variables are container-specific,
the VM custom properties are totally inuntrusive and old concept in oVirt, so
this step is totally safe.

Now restart Engine to let it use the new variables:

    
    # systemctl restart ovirt-engine
    

The next step is actually configure one "container VM" and run it.

## 4. Create the container "VM"

To finally run a container, you start creating a VM much like you always did, with
few changes

  1. most of the hardware-related configuration isn't relevant for container "VMs",
     besides cpu share and memory limits; this will be better documented in the
     future; unneeded configuration will just be ignored

  2. You need to set some custom properties for your container "VM". Those are
     actually needed to enable the container flow, and they are documented in
     the next section. You *need* to set at least `containerType` and `containerImage`.

### 4.2. Custom variables for container support

The container support needs some custom properties to be properly configured:

  1. `containerImage` (*needed* to enable the container system).
     Just select the target image you want to run. You can use the standard syntax of the
     container runtimes.
     For example, if you set the value to `redis`, the container will use the docker redis image
     from the docker registry.

  2. `containerType` (*needed* to enable the container system).
     Selects the container runtime you want to use. So far only docker is supported, so there is no
     real choice; but please note that if you *do not* explicitely select the docker runtime,
     the container infrastructure will not work, and you will have a regular VM instead.

Example configuration:

    
    containerImage = redis
    containerType = docker
    

### 4.2. A little bit of extra work: preload the images on the Vdsm host

This step is not needed by the flow, and will be handled by oVirt in the future.
The issue is how the container image are handled. They are stored by the container
management system on each host, and they are not pre-downloaded.

To shorten the duration of the first boot, you are advised to pre-download
the image(s) you want to run. For example

    
    ## on the Vdsm host you want to use with containers
    # docker pull $image

  example:

    
    # docker pull redis
    

## 5. Run the container "VM"

You are now all set to run your "VM" using oVirt Engine, just like any existing VM.
Some actions doesn't make sense for a container "VM", like live migration.
Engine won't stop you to try to do those actions, but they will fail gracefully
using the standard errors.

## 6. Next steps

What to expect from this project in the future?
For the integration with Vdsm, we want to fix the existing known issues, most notably:

  * add proper monitoring/reporting of the container health

  * ensure proper integration of the container image store with oVirt storage management

  * streamline the network configuration

What is explicitely excluded yet is any Engine change. This is a Vdsm-only change at the
moment, so fixing the following is currently unplanned:

  * First and foremost, Engine will not distinguish between real VMs and container VMs.
    Actions unavailable to container will not be hidden from UI. Same for monitoring
    and configuration data, which will be ignored.

  * Engine is NOT aware of the volumes one container can use. You must inspect and do the
    mapping manually.

  * Engine is NOT aware of the available container runtimes. You must select it carefully

Proper integration with Engine may be added in the future once this feature exits
from the experimental/provisional stage.

Thanks for reading, make sure to share your thoughts on the oVirt mailing lists!
