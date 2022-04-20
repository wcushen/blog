---
layout:     post
title:      "Service Mesh mTLS with NGINX Reverse Proxying"
subtitle:   ""
description: "Skupper Rocks"
excerpt: "Skupper rocks again"
date:       2022-04-18
author:         "Will Cushen"
image: "/img/2018-04-11-service-mesh-vs-api-gateway/background.jpg"
published: true
tags:
    - Microservice
    - Service Mesh
    - Kubernetes
categories: [ Tech ]
URL: "/2022-skupper"
---

## Serivice Mesh and Microservices

Mircoservices does what it says on the 'tin' - Their metric rise   are part of the wider DevOps evolution that allow organisation to respond qcuiker to chanigng  buiness needs and capatalise on the elasticity of the cloud. Going from moonotlic to microsevrice is not without it's apparent challenges;  more moving parts equates to more overhead

The array of Service Mesh technologities in the field are design to help enginers and archiects improve the  security, obsevrality and traffic control of an organisation's microservices.

On the heels of last year's Log4J and Log4Shell vulenabirlieis;irrspiective of industrt, a zero-trust access model should be accepted as standard practice


### What is mTLS?

Usually whenone visits a browser with HTTPS prefixed ot the URL ; the broswer will attempt to validate the server's certificate to make sure they're really who they say they are. 

mTLS uphols this Zero Trust paradigm by extending typical service-side SSL by requiring the client to present its certiicates for validation as part of the TLS handshake. 

## About the example

For those that are fmailar with Sevrice Mesh mTLS capabiltities, in most cirucmstances you'r eepxosure has most likely been in the form of mTLS between microservices running inside the mesh. In this post we'll look at example of an external server communciating with an application running inside an OpenShift Service Mesh 2.2 enabled namespace. 

Why an extenral Load Balcner when we can run a perfectly adeuqate in ingress gateway at the edge mesh?

With an L7 extenral lad balcnace outisde the mesh we're attmepting to depict a comon orgnisational scenario with a GTM/LTM fronting various virtual and contaiinerized applications and offloading features such as DDoS defnce and traffic fitleing to a deicated appliance. Addiotnally, we may be tailoring our soltion for mutli-cluster and mutli-regional load balancing, the latter partiucalrly relevant in the cloud. 

This example in this article represents are very srtipped down, rudimentay setup of an NGINX Revser Proxy fornting our httpbin workload running in an insatance of OpenShift Service Mesh 2.x; a birthchile of the Istio project. Nonetheless, the intention is to more highlight are increaisnlgy sought after seucirty feature in mTLS that although not unique to, for organisations adopting microservcies and service mesh tehcnologies alike. 


### Step 1: Installing Skupper CLI

First things first, we will need to grab the latest version of the Skupper CLI from https://github.com/skupperproject/skupper-cli/releases. Let's add it to our path and make it executable.

```
$ curl https://skupper.io/install.sh | sh

$ sudo mv .local/bin/skupper /usr/local/bin 
```

### Step 2: Create our namespaces

Log into your OCP clusters with `oc login`. We have created a `north` and a `south` namespace, which we will be dpeploying our two applications into. 

We can see the output of `skupper status` proves the absence of anything Skupper at this point.

```
$ oc project
Using project "north" on server "https://api.cluster-one.example.com:6443".

$ skupper status
Skupper is enabled for namespace "north" in interior mode. It is connected to 1 other site. It has 1 exposed service.
The site console url is:  https://skupper-north.apps.cluster-one.example.com
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'
``` 

```
$ oc project
Using project "south" on server "https://api.cluster-two.example.com:6443".

$ skupper status
Skupper is enabled for namespace "south" in interior mode. It is connected to 1 other site. It has 1 exposed service.
The site console url is:  https://skupper-south.apps.cluster-two.example.com
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'
```

### Step 3: Installing Skupper Router and Controller

Next we will roll out the Skupper componentry being the Router and Controller in each namespace, responsible for creating our Virtual Application Network (VAN) and watching for service annotations (`internal.skupper.io/controlled: "true"`), respectively.  

For redundancy, we can also up the replica count on the router via `--router` to specify two or greater.

```
$ skupper init --site-name north
Skupper is now installed in namespace 'north'.  Use 'skupper status' to get more information.

$ oc get pod
NAME                                          READY   STATUS    RESTARTS   AGE
skupper-router-555dcb8f4d-5g24z               2/2     Running   0          6s
skupper-service-controller-547d68b9ff-24bfq   1/1     Running   0          4s
```

```
$ skupper init --site-name south
Skupper is now installed in namespace 'south'.  Use 'skupper status' to get more information.

$ oc get pod
NAME                                          READY   STATUS    RESTARTS   AGE
skupper-router-7f6d6dfb5f-ct7nd               2/2     Running   0          7s
skupper-service-controller-85f76cfdb8-jlgh6   1/1     Running   0          5s
```

### Step 4: Connecting our namespaces

Let's move onto generating a link token on the namespace that is serving as the **Listener** (`north`) in this instance.

```
$ skupper token create $HOME/secret.yaml
Token written to /home/lab-user/secret.yaml 
```

This renders a YAML file for the **Connecting** namespace on the `south` cluster to maintain as a `secret`.

Once transferred over via `scp` or other means, we can then create that link to instantiate the linkage between the two clusters.

```
$ skupper link create $HOME/secret.yaml
Site configured to link to https://claims-north.apps.cluster-one.example.com:443/dfd9ec0d-af2d-11ec-b421-0a126f52272a (name=link1)
Check the status of the link using 'skupper link status'.
```

It's important to know that once setup, in a Skupper mesh of **more** than two clusters; traffic will be completely bi-directional meaning that if the originating namespace (where the token was created initially) goes down; the network between those remaining participating clusters continues to be active.  

```
$ skupper link status
Link link1 is active
```

### Step 5: Deploy our two discrete applications

Now with our multi-cluster Skupper network in place, it's time to spin up the front and back-end services that will comprise our microservice using the YAML defintions verbatim from the documentation link above. 

NOTE: Going back to those 'tweaks' we said we would make; we need to:

1. Allow the `default` Service Accounts in both namespaces to use the privileged port of **80** via `oc adm policy add-scc-to-user privileged -z default`

[lab-user@bastion ~]$ oc logs frontend-65cd658457-76d7g
2022/04/07 03:35:09 [warn] 1#1: the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /etc/nginx/nginx.conf:2
nginx: [warn] the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /etc/nginx/nginx.conf:2
2022/04/07 03:35:09 [emerg] 1#1: mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)
nginx: [emerg] mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)

2. Update the name of the backend deployment to `hello`, since this will be the internal DNS name by which the frontend sends requests to the backend worker Pods (set inside `nginx.conf`). Unfortunately we can't as of yet modify the Service name of a Deployment when we execute a `skupper expose`

Let's create the backend first in the `south` namespace.

```
$ wget -O- -q https://k8s.io/examples/service/access/backend-deployment.yaml | sed  "s/name: backend/name: hello/g" | oc apply -f -
deployment.apps/hello created
```

Next the frontend deployment in the `north` namespace.

```
$ oc apply -f https://k8s.io/examples/service/access/frontend-deployment.yaml
deployment.apps/frontend created
```

Now create and expose the frontend service.

At this point, we can now initiate the frontend service and expose as a typical OpenShift Router for external access. Alternatively, we could expose via `--type LoadBalancer` and acces via an `externalIP`.

```
$ oc expose deployment frontend --port 80 && oc expose service frontend
```

### Step 6: Observe that the application does not work

Remembering that it won't yet have a connected backend, nor will the backend have a public ingress. 

```
$ curl $(oc get route frontend -o jsonpath='http://{.spec.host}') -I 
curl: (7) Failed to connect to frontend-north.apps.cluster-one.example.com port 80: Connection timed out
```

Clearly, our frontend can't find the backend

```
$ oc get pod
NAME                                          READY   STATUS    RESTARTS      AGE
frontend-65cd658457-jbtvs                     0/1     Error     2 (15s ago)   19s
skupper-router-c8c6b7bb4-86wrc                2/2     Running   0             2m42s
skupper-service-controller-55769965bd-ncskj   1/1     Running   0             2m41s

$ oc logs frontend-65cd658457-jbtvs
2022/04/07 04:08:02 [emerg] 1#1: host not found in upstream "hello" in /etc/nginx/conf.d/frontend.conf:2
nginx: [emerg] host not found in upstream "hello" in /etc/nginx/conf.d/frontend.conf:2
```

### Step 7: Expose the backend

The `skupper expose deployment` command will invoke the creation of a standard K8s Service with the addition of the necesary selectors so that traffic is brokered through the Skupper router. 

```
$ skupper expose deployment/hello --port 80
deployment hello exposed as hello
```

Observe the Service created.

```
$ oc describe service hello
...
Annotations:       internal.skupper.io/controlled: true
Selector:          application=skupper-router,skupper.io/component=router
```

Restart the frontend deployment

```
$ oc rollout restart deployment/frontend
deployment.apps/frontend restarted
```

### Step 8: Validate the frontend now responds

Now we're at a point where we can expect something back from the frontend with the backend now exposed via Skupper.

You should see the message generated by the backend:

```
$ curl $(oc get route frontend -o jsonpath='http://{.spec.host}') 
{"message":"Hello"}
```

## Conclusion

Skupper as you can see, has been architected so each application administrator is in control of the their own inter-cluster connectivity as opposed to a single `cluster-admin`. Some may argue the drawback as a result, is a decentalised mesh that could become problematic at scale when it comes to management and observability from a single point. Ultimately, it really depends in your environment who you intend to give the keys to. 

Finally, if we wanted to observe a graphical view of the sites and exposed services, the default-enabled Skupper Console provides us with some useful network topology data which we didn't look at in this demo. Skupper's development can be tracked on the official site or directly from the code on the GitHub project. 