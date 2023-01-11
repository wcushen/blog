---
layout:     post
title:      "Shipping Windows Containers to OpenShift"
subtitle:   ""
description: "Shipping Windows Containers to OpenShift"
excerpt: "Secondary Scheduler for OpenShift"
date:       2022-11-26
author:         "Will Cushen"
image: "/img/2022-11-openshift-windows/MSC-Vessel.jpeg"
published: true
tags:
    - Kubernetes
    - Windows
    - Containers
#categories: [ Tech ]
URL: "/2022-11-openshift-windows"
---

## Windows Containers

At the inecption of Docker continaers in 2013, the technology was very much exclusive to Linux, made possibe by the userspace interface LXC. This process-level islaotaiotn as the backbone of container tehcnolgoy has transformed the industry's way of developing and dpeloying applications. Windows were eager not to miss the boat and native Docker support for Windows came 'realtively soon after in its Server 2016 release and offered Windows Cotnainers as a service on Azure around this time as well. 

Since 2016, Windows supports three modes of isolation:

- Process Containers (Also known as Windows Server Containers — WSC)
- Hyper-V Containers - Containers  are isolate each container via a lightweight virtual machine (VM).
- HostProcess - new to Windows Server 2022 and  allowing containers to be created within the host's network namespace instead of their own, this mode is primed ot manage WIndows nodes in Kubenrtes (which is what this blog is all about!)

There is plenty of other blogs out there that get into the weeds of Windows COntainers, the intetion of this post is to show their opertaiblity with OpenShift. But one major distinction to note is the this concept of isoaltion that is ietched into the DNA of containers. For the most part, Linu and WINdows hadnle this in different ways. 

The former may look like an isoaltion sysnoymous with that of Linux, yyet unlike making direct system calls pplications don’t use system calls directly but rather call API functions that are delivered through internal DLLs. The implciaiotns here are that you are restircted to runing containerized worklaods on the WIndows OS from which you have built from (i.e. running a WSC on Server 2016 must have been built from a Server 2016 base). So any benefits of portability are really shot as a result. 



### Windows COntainers on openshift - Unpacking the'Why'?

We've come a long way since the days of Steve Balmer calling Linux a cncer back in 2001 . The Microsoft-Red Hat partnership continues to go from strength to strength since the lanuch of Red Hat Entperise Linux on Azure [back in 2015](https://www.redhat.com/en/blog/strengthening-power-collaboration-why-red-hat-and-microsoft-are-extending-our-partnership). But consdieirng this collabaroation in the Kubernetes space, why do we consider it important to target Windows containers for OpenShift?

Fundamentally, Windows Server still maintinains a considerable footpring amonsgt operatiing systems in the data center of the entperise today in addiotn to .NET continues to be widely used [as an applciaiotn framewokr](https://www.statista.com/statistics/1124699/worldwide-developer-survey-most-used-frameworks-web/)

Given .NET Framework 4.7.2 and above are supported on Windows Server 2019 (the minimum OS version compatible for OpenShift worklaods), we can follow the below decision tree to gauge if porting is required or direct containerization is possible. 

![](/img/2022-11-openshift-windows/dot-net-decision-tree.png)


Source: [Red Hat](https://cloud.redhat.com/blog/strategies-for-moving-.net-workloads-to-openshift-container-platform)

Ultiamtley, the afomrentioned patterns all tie in to blueprints populaurzed by Amazon Web Servcies's ['6 Rs' streatgy](https://aws.amazon.com/blogs/enterprise-strategy/6-strategies-for-migrating-applications-to-the-cloud/) that has been around for some time now, those of which fit in nicely to Red Hat's overall vision of adopting an open bybrid cloud platform.  

- Minimize the infrastrcuutre fooprting tpyically rrquired for Windows applciaiotns by deplyoing them as lightwieght containers
- We can champion portability, a adopt a run anywhere paradigm that hybrid bloud champions
- Develoeprs can relaize greater velocity in applciaiotn development made possible by Kubernetes and its ecocystsme, leading to faster time to market. 


### Windows Machine Config Operator (WMCO)


The Windows Machine Config Operator (WMCO) is the pivtoal feature that allows a cluster administrator to add a Windows worker node as a day 2 operation. 

We have a choice of either giving the OpenShift contorl place the reigns and spning up a Mahcine machine via a MAchineSet on any of the supportwds infrasrtrcuture platforms (vSphere, AWS or Azure at the time of writing) or alternaitvely, we can leverage the [Bring-Your-Own-Host](https://docs.openshift.com/container-platform/4.10/windows_containers/byoh-windows-instance.html) feature introdcued in Openshift 4.8.

The WMCO will perform all the necessary steps to configure the virtual amching so it can join the OpenShift cluster



{{% notice info %}}
The Windows Machine Config Operaotr (WMCO) is NOT repsosible for updating the operaitng systems of the Windows machine. The image upon creation is provided by the cluster administrator, and thereafter the machine's update and application of any WIndows CVE/securtiy patches is firmly the respobisility of the adminsitrator as well. 
{{% /notice %}}

https://www.zdnet.com/article/ballmer-i-may-have-called-linux-a-cancer-but-now-i-love-it/

## Wrap Up


