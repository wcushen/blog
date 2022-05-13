---
layout:     post
title:      "Deep Dive on Skupper"
subtitle:   ""
description: "Bridging multicloud workloads across Kubernetes clusters with the new interconnect on the block"
excerpt: "An introductory post on Skupper/Application Edge Interconnect"
date:       2022-04-04
author:         "Will Cushen"
image: "/img/2022-04-skupper-deep-dive/bridge-wide.jpg"
published: true
tags:
    - Microservice
    - Kubernetes
#categories: [ Tech ]
URL: "/2022-04-skupper-deep-dive"
---

## What is Skupper

The ubiquity of hybrid/multi-cloud architecture in enterprise today means that engineering teams are met with challenging design decisions on how to manage secure connections not only between regionally distributed workloads but different hosting stacks (i.e. containerised vs. virtualised). Skupper aims to simplify this task, marketing itself as a layer 7 service interconnect that allows namespaced-workloads across Kubernetes clusters to connect as if there were local. 

The topopoly is relatively straightforward with the lightweight Apache Qpid router brokering the traffic between namespaces. 

![Skupper Diagram](/img/2022-04-skupper-deep-dive/skupper-diagram.png)

### Removing Silos for Hybrid Cloud

Understanding the need for Skupper means taking stock of the growing hybrid-cloud boom as many organisations contend with the need to connect on-premise, private and public cloud applications. The inter-connect possibilities available through Skupper however, are not limited to, a mesh of Kubernetes clusters. As I alluded to above; virtualised, even mainframe-based workloads can be included in Skupper's fabric.

In this post, we'll highlight the _flagship_ use case of encrypting workload communication (over mTLS) between *two* Kubernetes (OpenShift 4.10) clusters.

## About the deployment

We will create a front and back-end microservice, deploying each in a separate cluster to show the real value prop of Skupper. The example used can be found in these Kubernetes docs [here](https://kubernetes.io/docs/tasks/access-application-cluster/connecting-frontend-backend/) where we'll apply some minor tweaking to cater for some 'Skupper on OpenShift' nuances which we'll outline below. 

### Step 1: Installing the Skupper CLI

Seeing we have an OpenShift cluster in the picture here, we do have the community-based Skupper Operator at our disposal. There are slight distinctions I've found (when compared to the CLI method shown below), such as the provisioning of a `skupper-site-controller` which allows us to follow a declarative method to install Skupper. We can store our YAML files this way if we have GitOps practices we need to adhere to. 

In the interest of this post to get folks up and running, we'll stick with what the Skupper documentation refers to as the **_'primary entrypoint'_** for installing Skupper and do so via its CLI. 

So let's go ahead and do just that. We will need to grab the latest [version](https://github.com/skupperproject/skupper-cli/releases) of the Skupper CLI. 

And then finally add it to our path and make it executable.

```yaml
$ curl https://skupper.io/install.sh | sh && \
sudo mv .local/bin/skupper /usr/local/bin 
```

### Step 2: Creating our namespaces

We have created a `north` and a `south` namespace which we will be deploying our two applications into. 

We can see the output of `skupper status` proves the absence of anything Skupper at this point.

```yaml
$ oc project
Using project "north" on server "https://api.cluster-one.example.com:6443".

$ skupper status
Skupper is not enabled for namespace 'north'
``` 

```yaml
$ oc project
Using project "south" on server "https://api.cluster-two.example.com:6443".

$ skupper status
Skupper is not enabled for namespace 'south'
```

### Step 3: Installing the Skupper Router and Controller

Next we will roll out the Skupper componentry being the Router and Controller in each namespace, responsible for creating our Virtual Application Network (VAN) and watching for service annotations (`internal.skupper.io/controlled: "true"`), respectively.  

{{% notice tip %}}
For HA, we can also up the replica count on the router via `--router` to specify two or greater.
{{% /notice %}}

```yaml
$ skupper init --site-name north
Skupper is now installed in namespace 'north'.  Use 'skupper status' to get more information.

$ skupper status
Skupper is enabled for namespace "north" in interior mode. It is connected to 1 other site. It has 1 exposed service.
The site console url is:  https://skupper-north.apps.cluster-one.example.com
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'

$ oc get pod
NAME                                          READY   STATUS    RESTARTS   AGE
skupper-router-555dcb8f4d-5g24z               2/2     Running   0          6s
skupper-service-controller-547d68b9ff-24bfq   1/1     Running   0          4s
```

```yaml
$ skupper init --site-name south
Skupper is now installed in namespace 'south'.  Use 'skupper status' to get more information.

$ skupper status
Skupper is enabled for namespace "south" in interior mode. It is connected to 1 other site. It has 1 exposed service.
The site console url is:  https://skupper-south.apps.cluster-two.example.com
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'

$ oc get pod
NAME                                          READY   STATUS    RESTARTS   AGE
skupper-router-7f6d6dfb5f-ct7nd               2/2     Running   0          7s
skupper-service-controller-85f76cfdb8-jlgh6   1/1     Running   0          5s
```

### Step 4: Connecting our namespaces

Let's move onto generating a link token on the namespace that is serving as the **Listener** (`north`) in this instance.

```yaml
$ skupper token create $HOME/secret.yaml
Token written to /home/lab-user/secret.yaml 
```

This renders a YAML file for the **Connecting** (`south`) namespace to write as a `secret`.

Once transferred over via `scp` or other means, we can then create that secret to establish the connection between the two clusters.

```yaml
$ skupper link create $HOME/secret.yaml
Site configured to link to https://claims-north.apps.cluster-one.example.com:443/dfd9ec0d-af2d-11ec-b421-0a126f52272a (name=link1)
Check the status of the link using 'skupper link status'.
```

It's important to know that once setup, in a Skupper mesh of **more** than two clusters; traffic will be completely _bi-directional_, meaning that if the originating namespace (where the token was created initially) goes down; the network between those remaining participating clusters will continue to be active.  

```yaml
$ skupper link status
Link link1 is active
```

### Step 5: Deploying our two detached applications

Now with our multi-cluster Skupper network in place, it's time to spin up the front and back-end services that will comprise our microservice using the YAML definitions verbatim from the documentation link above. 

Coming back to those _'tweaks'_ we said we would make. Effectively, we need to: 

{{% notice info %}}
**1.** Allow the `default` Service Accounts in _both_ namespaces to use the privileged port of **80** via `oc adm policy add-scc-to-user anyuid -z default`
**2.** Update the name of the backend deployment to `hello`, since this will be the internal DNS name by which the frontend sends requests to the backend worker Pods (set inside `nginx.conf`). Unfortunately we can't as of yet modify the Service name of a Deployment when we execute a `skupper expose`
{{% /notice %}}

Let's create the backend first in the `south` namespace.

```yaml
$ wget -O- -q https://k8s.io/examples/service/access/backend-deployment.yaml | sed  "s/name: backend/name: hello/g" | oc apply -f -
deployment.apps/hello created
```

Next the frontend deployment in the `north` namespace.

```yaml
$ oc apply -f https://k8s.io/examples/service/access/frontend-deployment.yaml
deployment.apps/frontend created
```

Now create and expose the `frontend` service.

At this point, we can now initiate the frontend service and expose as a typical OpenShift Route for external access. Alternatively, we could expose via `--type LoadBalancer` and access via an `externalIP`.

```yaml
$ oc expose deployment frontend --port 80 && oc expose service frontend
```

### Step 6: Observing that the application does not work

Remembering that it won't yet have a connected backend, nor will the backend have a public ingress. 

```yaml
$ curl $(oc get route frontend -o jsonpath='http://{.spec.host}') -I 
curl: (7) Failed to connect to frontend-north.apps.cluster-one.example.com port 80: Connection timed out
```

Clearly, our `frontend` can't find the `hello` backend.

```yaml
$ oc get pod
NAME                                          READY   STATUS    RESTARTS      AGE
frontend-65cd658457-jbtvs                     0/1     Error     2 (15s ago)   19s
skupper-router-c8c6b7bb4-86wrc                2/2     Running   0             2m42s
skupper-service-controller-55769965bd-ncskj   1/1     Running   0             2m41s

$ oc logs frontend-65cd658457-jbtvs
2022/04/07 04:08:02 [emerg] 1#1: host not found in upstream "hello" in /etc/nginx/conf.d/frontend.conf:2
nginx: [emerg] host not found in upstream "hello" in /etc/nginx/conf.d/frontend.conf:2
```

### Step 7: Exposing the backend

The `skupper expose deployment` command will invoke the creation of a standard K8s Service with the addition of the necessary selectors so that traffic is brokered through the Skupper router. 

```yaml
$ skupper expose deployment/hello --port 80
deployment hello exposed as hello
```

Observe the updated `skupper-services` ConfigMap and the created Service.

```yaml
$ oc get cm skupper-services -o jsonpath='{.data}'
{"hello":"{\"address\":\"hello\",\"protocol\":\"tcp\",\"ports\":[80],\"targets\":[{\"name\":\"hello\",\"selector\":\"app=hello,tier=backend,track=stable\",\"targetPorts\":{\"80\":80}}]}"}

$ oc describe service hello
...
Annotations:       internal.skupper.io/controlled: true
Selector:          application=skupper-router,skupper.io/component=router
```

Restart the `frontend` deployment.

```yaml
$ oc rollout restart deployment/frontend
deployment.apps/frontend restarted
```

### Step 8: Validating the frontend now responds

Now we're at a point where we can expect a response back from the frontend with the backend _now exposed via Skupper_.

You should see the message generated by the backend:

```yaml
$ curl $(oc get route frontend -o jsonpath='http://{.spec.host}') 
{"message":"Hello"}
```

## Conclusion

Skupper as you can see, has been architected so each application administrator **is in control** of the their own inter-cluster connectivity as opposed to a `cluster-admin` delegation having to manage everything. Some may argue the drawback as a result, is a decentralised mesh that could become problematic at scale when it comes to management and observability from a single point. _Ultimately_, it really depends who you intend to give the keys to in your environment. 

Finally, if we wanted to observe a graphical view of the sites and exposed services, the default-enabled Skupper Console provides us with some useful network topology data which we didn't look at in this demo. Skupper's development can be tracked on the official site or directly from the code on the [GitHub project](https://github.com/skupperproject/skupper). Sought after features such as extending to other layer 7 protocols beyond gRPC and HTTP or making use of custom certificates are some of many active developments (at the time of writing) being worked on in the Skupper roadmap.

I hope you enjoyed this introductory post on Skupper!