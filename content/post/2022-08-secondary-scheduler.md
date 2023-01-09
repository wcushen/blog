---
layout:     post
title:      "BYO Scheduler for greater Kubernetes Scheduling efficiency"
subtitle:   ""
description: "BYO Scheduler for greater Kubernetes Scheduling efficiency"
excerpt: "Secondary Scheduler for OpenShift"
date:       2022-08-10
author:         "Will Cushen"
image: "/img/2022-08-secondary-scheduler/mso-conductor.jpeg"
published: true
tags:
    - Kubernetes
    - Scheduling
#categories: [ Tech ]
URL: "/2022-08-secondary-scheduler"
---

## Kubernetes Default Scheduling

The orshcrestaion capabilities of Kubernets when it graced the enteprrise back in 2014 were truly remarkable. The ability to assign Pods to Nodes in Kubernetes allows users of the platform to consitently satisfy various levels of Quality of Service (QoS).

The Request valuer assigned by the developer of the Kuberlentes workload is ultimately what gets fed into the dfault scheudler's twop step oepration of filteirng and scoring. 

This presesents a number of issues, all of which lead to a overall low-tuilization of the cluster:

- It's diffuclt (particular in ealry releases) to ascertain an accurare resource usage for any given application, and thus any user
- Considering all the 3 QoS Classes available: Guarantreed, Bustable and Best-Effort - in a mission crtiical Production evnironemtn, the tendency for Develoeprs is almost always to assign a Qos of Quarantees (i.e. Request = Limits) and therefore it introduces this mentality of of 'high-balling' the Request
- The default scheudlking plugins available out-of-the-box don't consider any live (Promethues) data to determine the node's real time node utilization 

By introducing a new scheudling mechanicsm, we'd be effictvely solving the inefficienciy of Kubernetes' standard scheudling and really should be at the top of any amdinsitrator's list when we consider any cost otpimzation/recovery activities on the platform. 

### Secondary Scheduler Operator

OpenShift allow us to customize how worklaods are scheduling via the Seconadry Scheudler Operator. Through this BYO Schduler paradigm, we can leverage Secondary Scheduler Operator to manage the workloads of our choice yet the control plane components would still use the default scheduler shipped with OpenShift.

A Pod or Deployment must opt-in to be schuedled by an alternaitve scheulder to the default provided by OpenShift. 

We have a host of alternative [scheuler plugins](https://github.com/kubernetes-sigs/scheduler-plugins) at our disposal base on the [scheudler framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/).

Keeping in mind the problem stament above of nodes being under-utilised, it'd be wise of us to employ a scheudler plgiuin that seeks to pack nodes more effieicently. Certainly there are pitalls to just considering CPU Utilizaiton of the node oultined in this Red Hat blog [here](https://cloud.redhat.com/blog/improving-the-resource-efficiency-for-openshift-clusters-via-trimaran-schedulers), but as an introduction to secondary scheduling let us take the basic goal of maintaining node utlizaiotn at a certina level and consider the Trimaran scheulder.

Of the two scheudler strategies under Triamaran, TargetLoadPacking and LoadVariationRiskBalancing - the plugin uses a [load watcher](https://github.com/paypal/load-watcher) that retrieves five-minute history metrics (from metric providers like Promethues) every minute and caches the analysis results and feeding those inputs into the Trimaran scheudling algoirtim, all in all allowing us to bridge this gap between the allocated resources (requests) to what's really happneing on the node (real-time utulization).

### TargetLoadPacking Plugin

Let's take a look under the hood of the plugin's algotrithim and end result of a schduled pod.

Suppose we have the follwing output that indicates the current CPU utlization. Memory here has been redacted to simplify the demonstration. 

```
$ oc adm top node -l node-role.kubernetes.io/worker=""
NAME       CPU(cores)    CPU%   
worker-1   2000m         25%   
worker-2   4000m         50%     
worker-3   800m          10%    
```

Here we have three worker nodes,, with 8 cores each and utlizaiton at 2, 4 and 0.8 cores sprecticaly (together with the value expressed as a %)

#### Algorithim

1. Determine the utilization of the current node. We'll call this 'A'.
2. Calculate the current pod's total CPU requests and overhead. We'll call this 'B'.
3. Add A and B together to get the expected utlizaiton (U). 

In our example, again for simplicitiy let us assume our pod(s) to be scheduled has 0 CPU requests and overhead.

Encapsualting the alogrithim into pseudocode, our goal is to output a Score for each node.

```
IF U <= X%
  score = (100 - X)U/X + X
ELSE IF X% < U <= 100%
  score = X(100 - U)/(100 - X)
ELSE IF U > 100%
  score = 0
```

Let us make the target utlization (X) equal to 40%. We can take actual utilzaiotn of the nodes from the output of `oc adm top nodes` above. 

Calculating the score of each node in that case:

```
worker-1 → (100 - 40)*25/40 + 40 = 77.5
worker-2 → 40 * (100 - 50)/(100 - 40) = 33.3
worker-3 → (100 - 40)*10/40 + 40 = 55
```

Obsevring the piecefunction function as a plot captured from the plugin's [README](https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/kep/61-Trimaran-real-load-aware-scheduling/README.md#targetloadpacking-plugin), we can see that the nodes are favored and penalisez linearly as they apporach and surpass the target utlization, respectively (in this case, 50%).

![](/img/2022-08-secondary-scheduler/trimaran-graph.png)

### Install the Operator

Hopefully by now, we have enough insight into the mechanics of our chosen prfile and we can now go ahead and isntall the Secondary Operator on our OpenShift cluster and see this load aware scheduling in action. 

1. Create openshift-secondary-scheduler-operator namespace:

```
$ oc create ns openshift-secondary-scheduler-operator
```

2. Proceed to the console's sidebar, Operators -> OperatorHub, search for secondary scheduler operator and install the operator

![](/img/2022-08-secondary-scheduler/secondary-scheduler-operator-hub.png)

3. Create a ConfigMap secondary-scheduler-config for the Trimaran KubeSchedulerConfiguration under the `openshift-secondary-scheduler-operator` namespace. 

```yaml
$ cat <<EOF > config.yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
leaderElection:
  leaderElect: false
profiles:
  - schedulerName: secondary-scheduler
    plugins:
      score:
        disabled:
          - name: NodeResourcesBalancedAllocation
          - name: NodeResourcesLeastAllocated
        enabled:
          - name: TargetLoadPacking
    pluginConfig:
      - name: TargetLoadPacking
        args:
          defaultRequests:
            cpu: "1000m"
          defaultRequestsMultiplier: "1"
          targetUtilization: 40
          metricProvider:
            type: Prometheus
            address: ${PROM_URL}
            Token: ${PROM_TOKEN}
EOF
```

{{% notice info %}}
For descriptions of the `TargetLoadPacking` arguments refer to this [README](https://pkg.go.dev/sigs.k8s.io/scheduler-plugins/pkg/trimaran/targetloadpacking#section-readme)
{{% /notice %}}

4. Define a series of Promethues specific variables required by the ConfigMap, before instantiating the ConfigMap

```yaml
export PROM_HOST=`oc get routes prometheus-k8s -n openshift-monitoring -ojson |jq ".status.ingress"|jq ".[0].host"|sed 's/"//g'` && \
PROM_URL="https://${PROM_HOST}" && \
TOKEN_NAME=`oc get secret -n openshift-monitoring|awk '{print $1}'|grep prometheus-k8s-token -m 1` && \
PROM_TOKEN=`oc describe secret $TOKEN_NAME -n openshift-monitoring|grep "token:"|cut -d: -f2|sed 's/^ *//g'`
```

```
$ oc create -n openshift-secondary-scheduler-operator configmap secondary-scheduler-config --from-file=config.yaml
```

5. And then from here, all we need to do is point the `scheduelrConfig` to our secondary-scehduelr-config (ConfigMap) upon creating our SecondaryScheudler CR (leaving defaults for the remainder). We can do this via the console or alternatively via the below:

```yaml
$ cat <<EOF | oc apply -f -
kind: SecondaryScheduler
apiVersion: operator.openshift.io/v1
metadata:
  name: cluster
  namespace: openshift-secondary-scheduler-operator
spec:
  managementState: Managed
  schedulerConfig: secondary-scheduler-config
EOF
```

6. Let's create a Pod (in the `openshift-secondary-scheduler-operator` namespace is fine) that desginates our secondary scheduler for allocation.

```yaml
$ cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: custom-scheduler-example
  labels:
    name: custom-scheduler-example
spec:
  schedulerName: secondary-scheduler 
  containers:
  - name: pod-with-second-annotation-container
    image: docker.io/ocpqe/hello-pod
EOF
```



## Wrap Up

This was just a fun dabble demonstrating mTLS communication from outside Istio. Certainly there's benefit in drawing from _some_ of the security architecture in here that could be implemented in a scale-out enterprise environment - it's a balance of _risks_ and _needs_; and **where** and **how** we terminate SSL is certainly one of those considerations. 

It should be noted that as of OpenShift 4.9, mTLS authentication can be enabled in the Ingress Controller, so there are other, arguably simpler ways if we want _just_ want to cherry-pick certain security features that were only previously on Istio's bumper sticker, at least in the OpenShift space :smile:

I hope you got some value from the above walkthrough and most importantly, hopefully forwards any discussions you might be having in your team on how you treat your Kubernetes/Service Mesh SSL. 