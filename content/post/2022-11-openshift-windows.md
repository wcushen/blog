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

## Containerizing Windows

At the inception of Docker containers in 2013, the technology was very much exclusive to Linux. Made possible by LXC, a userspace interface for the Linux kernel containment features, this process-level isolation has served as the backbone of container technology and transforming the industry's way of developing and deploying applications in the process. Microsoft were eager not to miss the boat and native Docker support for Windows came _'relatively'_ soon after in its Server 2016 release and offered Windows Containers as a Service (CaaS) on Azure around this time as well. 

Since 2016, Windows has supported three **modes of isolation**:

- **Process Containers** - also known as Windows Server Containers (WSC)
- **Hyper-V Containers** - containers are isolated via a lightweight virtual machine 
- **HostProcess Containers** - new to Windows Server 2022 allowing containers to be created within the host's network namespace instead of their own. This mode is primed to manage Windows nodes in Kubernetes (although it won't be covered here in this article)

There are plenty of other blogs out there that get into the weeds of Windows Containers, so the intention of this post is rather to show their operability on OpenShift. 

One major distinction to note however, is this _concept of isolation_ that is etched into the DNA of containers that ultimately separates Linux and Windows.

Windows' _process isolation_ may look like isolation comparable to that of Linux, yet unlike making system calls directly, Windows calls API functions that are delivered through internal DLLs. The implications here are that you are restricted to running containerized workloads on the Windows OS from which you have built from (i.e. running a WSC on Server 2016 must have been built from a Server 2016 base). So any real benefits of **portability** are really shot as a result. 

#### Windows Containers on OpenShift - _Unpacking the Why_

We've come a long way since the days of Steve Balmer calling Linux a [cancer](https://www.theregister.com/2001/06/02/ballmer_linux_is_a_cancer/) back in 2001. The Microsoft-Red Hat partnership has continued to go from strength to strength since the launch of Red Hat Enterprise Linux on Azure [back in 2015](https://www.redhat.com/en/blog/strengthening-power-collaboration-why-red-hat-and-microsoft-are-extending-our-partnership). But considering a need to collaborate in the Kubernetes space, _why would we deem it important to target OpenShift for Windows containers?_

Fundamentally, Windows Server still maintains a considerable footprint amongst operating systems in the data centers of the enterprise today. What's more, .NET continues to be widely used [as an application framework](https://www.statista.com/statistics/1124699/worldwide-developer-survey-most-used-frameworks-web/) amongst developers. 

Given _.NET Framework 4.7.2 and above_ are supported on Windows Server 2019 (_the minimum OS version compatible for OpenShift workloads_), we can follow the below decision tree to gauge if a path of **porting** or **direct containerization** should be taken for applications traditionally developed on Windows. 

![**_Source:_** [Red Hat](https://cloud.redhat.com/blog/strategies-for-moving-.net-workloads-to-openshift-container-platform)](/img/2022-11-openshift-windows/dot-net-decision-tree.png)
<center> **_Source:_** [Red Hat](https://cloud.redhat.com/blog/strategies-for-moving-.net-workloads-to-openshift-container-platform)</center>

Ultimately, the aforementioned patterns all tie in to blueprints popularized by Amazon Web Services' ['6 Rs' strategy](https://aws.amazon.com/blogs/enterprise-strategy/6-strategies-for-migrating-applications-to-the-cloud/) that has been around for some time now, and so too align with Red Hat's overall vision of adopting an [open hybrid cloud](https://www.redhat.com/en/topics/cloud/open-hybrid-cloud-approach).

**The net benefit being here**:

- Minimizing our infrastructure footprint typically required for Windows applications by deploying them as lightweight containers
- Championing _portability_ and adopting a **'run anywhere'** model that is inherent to hybrid cloud
- Developers can realize greater velocity in application development made possible by Kubernetes and its ecosystem, resulting in faster time to market. 

### Windows Machine Config Operator (WMCO)


The Windows Machine Config Operator (WMCO) is the pivotal feature that allows a cluster administrator to add a Windows worker node as a day 2 operation. 

We have a choice of either giving the OpenShift control plane the reigns to spin up a Windows machine via a `MachineSet` on any of the supported infrastructure platforms (vSphere, AWS or Azure at the time of writing) or alternatively, we can leverage the [Bring-Your-Own-Host](https://docs.openshift.com/container-platform/4.10/windows_containers/byoh-windows-instance.html) feature introduced in Openshift 4.8.

The WMCO will perform all the necessary steps to configure the virtual machine so it can join the OpenShift cluster.

{{< figure src="/img/2022-11-openshift-windows/wmco.png" link="/img/2022-11-openshift-windows/wmco.png" >}} {{< load-photoswipe >}}
<center>ztunnel 四层处理完整流程</center>

{{% notice info %}}
The Windows Machine Config Operator (WMCO) is **NOT** responsible for updating the operating systems of the Windows machine. The image upon creation is provided by the cluster administrator, and thereafter the machine's OS upgrade and application of any Windows CVE/security patches are firmly the responsibility of the administrator as well. 
{{% /notice %}}

####  Deploying a Windows Web Application

With a bit of theory out the way, we can now deploy a sample application running a webservice on the Windows Node. This application will be running inside a Windows Container on the Windows Node.

The WMCO spins up a special ssh container for us to remote shell into and then from there we can run the provided script to drop us into a PowerShell prompt on the Windows host. 

```yaml
$ oc -n openshift-windows-machine-config-operator rsh $(oc get pods -n openshift-windows-machine-config-operator -l app=winc-ssh -o name)
```

Running a `systeminfo` we can confirm we have a Windows 2019 node here as part of the cluster. 

```powershell
PS C:\Users\Administrator> systeminfo | Select-String "OS Name"

OS Name:                   Microsoft Windows Server 2019 Datacenter
```

Let us try and pull a **Windows Server 2022** container image to this machine.

```powershell
PS C:\Users\Administrator> docker pull mcr.microsoft.com/windows/servercore:ltsc2022-amd64
ltsc2022-amd64: Pulling from windows/servercore
a Windows version 10.0.20348-based image is incompatible with a 10.0.17763 host
```

**FAIL!** As we covered earlier, this is expected and in theory could be circumnavigated by resorting to [Hyper-V isolation](https://learn.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/version-compatibility?tabs=windows-server-2022%2Cwindows-11), but we know the WMCO hasn't configured [our machine in this way](https://docs.openshift.com/container-platform/4.10/windows_containers/understanding-windows-container-workloads.html). 

A bit of self-acknowledgement from Microsoft themselves stating that _"decoupling the User/Kernel boundary in Windows is a monumental task and highly non-trivial"_ and is something they're looking to improve in future releases of Windows Server and Client.

Given that it's 2022, I'm eager for my infrastructure to remain relevant and up-to-date, so _what I can do_ is perform an in-place upgrade of my Windows Server OS to 2022 and that should make this container image I will try to ultimately use for my application, all happy and compliant.

This Windows node is one that I have rolled out as a Bring-Your-Own-Instance (BYOH) node as [explained here](https://docs.openshift.com/container-platform/4.10/windows_containers/byoh-windows-instance.html). Irrespective however if BYOH or a `MachineSet` is made use of, the upgrade of the OS is handled **outside** OpenShift and the WMCO, so this is all on me to do!

Sparing those [details](https://learn.microsoft.com/en-us/windows-server/get-started/perform-in-place-upgrade) from this post, once upgraded I can log back into the host to first confirm my new OS version and re-attempt the pulling of the 2022 image. 

{{% notice warning %}}
Windows Container images can be VERY large. In some cases, a small base image can be 8GB in size, and therefore it is suggested to pre-pull any base images you need. 
{{% /notice %}}

```powershell
PS C:\Users\Administrator> systeminfo | Select-String "OS Name"

OS Name:                   Microsoft Windows Server 2022 Datacenter
```

Now, I can grab ready made Deployment and Service for my Windows web server although with one minor change. I will update the image tag to reflect the 2022 Core image. 

```yaml
$ wget -O - https://gist.githubusercontent.com/suhanime/683ee7b5a2f55c11e3a26a4223170582/raw/d893db98944bf615fccfe73e6e4fb19549a362a5/WinWebServer.yaml | sed -e 's/ltsc2019/ltsc2022/g' | oc apply -f -
```

{{% notice info %}}
Remember to note that the Deployment has a toleration so it can run on the Windows Node.
{{% /notice %}}

Let's confirm we're looking all good from a Pod perspective.

```yaml 
[lab-user@bastion ~]$ oc get pod
NAME                             READY   STATUS    RESTARTS   AGE
win-webserver-5db7f85d96-9vm96   1/1     Running   0          10s
```
And with that we can expose the Service and retrieve the Route to access our welcome page. 

```yaml
$ oc expose svc/win-webserver
route.route.openshift.io/win-webserver exposed

$ oc get route
NAME            HOST/PORT                                               PATH   SERVICES        PORT   TERMINATION   WILDCARD
win-webserver   win-webserver-windows-workloads.apps.cluster.test.com          win-webserver   80                   None
```
![](/img/2022-11-openshift-windows/webserver-page.png)

And there we have it, a Windows Sever 2022 container image serving a web page on a Windows Server 2022 worker node! :smile:

## Wrap Up

Hopefully this post has demonstrated the ease in which we can bring Windows workloads to OpenShift with very low friction in order to capitalize on the benefits of containerization. And we don't have to stop there, we can think about bringing other OpenShift hosting features like [OpenShift Virtualization](https://www.redhat.com/en/technologies/cloud-computing/openshift/virtualization) to cater for further use cases.

Microsoft's "love" for Linux has existed for almost a decade now (even Steve's [come around](https://www.zdnet.com/article/ballmer-i-may-have-called-linux-a-cancer-but-now-i-love-it/)) which has really paved the way for developments like these covered in this post. Ultimately, they've really moved the needle for the hybrid cloud strategies of Red Hat and Microsoft's customers alike and highlight just how much organisations can benefits from large consolidation efforts across their infrastructure.