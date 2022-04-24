---
layout:     post
title:      "Service Mesh mTLS with NGINX Reverse Proxying"
subtitle:   ""
description: "Service Mesh"
excerpt: "Service Mesh"
date:       2022-04-18
author:         "Will Cushen"
image: "/img/2018-04-11-service-mesh-vs-api-gateway/background.jpg"
published: true
tags:
    - Microservice
    - Service Mesh
    - Kubernetes
categories: [ Tech ]
URL: "/2022-servicemesh"
---

## Serivice Mesh and Microservices

Mircoservices does what it says on the 'tin' - Their metric rise   are part of the wider DevOps evolution that allow organisation to respond qcuiker to chanigng  buiness needs and capatalise on the elasticity of the cloud. Going from moonotlic to microsevrice is not without it's apparent challenges;  more moving parts equates to more overhead

The array of Service Mesh technologities in the field are design to help enginers and archiects improve the  security, obsevrality and traffic control of an organisation's microservices.

On the heels of last year's Log4J and Log4Shell vulenabirlieis*; irrspiective of industrt, a zero-trust access model should be accepted as standard practice


### What is mTLS?

Usually whenone visits a browser with HTTPS prefixed ot the URL ; the broswer will attempt to validate the server's certificate to make sure they're really who they say they are. 

mTLS uphols this Zero Trust paradigm by extending typical service-side SSL, also requiring the client to present its certiicates for validation as part of the TLS handshake. 

## About the example

For those that are fmailar with Sevrice Mesh mTLS capabiltities, in most cirucmstances you'r eepxosure has most likely been in the form of mTLS between microservices running inside the mesh. In this post we'll look at example of an external server communciating with an application running inside an OpenShift Service Mesh 2.2 enabled namespace. 

Why an extenral Load Balcner when we can run a perfectly adeuqate in ingress gateway at the edge mesh?

With an L7 extenral lad balcnace outisde the mesh we're attmepting to depict a comon orgnisational scenario with a GTM/LTM fronting various virtual and contaiinerized applications and offloading features such as DDoS defnce and traffic fitleing to a deicated appliance. Addiotnally, we may be tailoring our soltion for mutli-cluster and mutli-regional load balancing, the latter partiucalrly relevant to the cloud. 

This example in this article represents are very srtipped down, rudimentay setup of an NGINX Revser Proxy fornting our httpbin workload running in an insatance of OpenShift Service Mesh 2.x; a birthchile of the Istio project. Nonetheless, the intention is to more highlight a prominent seucirty feature in mTLS that is becoming increaisngly sought after, although not unique to, in microservcies and service mesh tehcnologies alike. 

*Log4J isn't as aplciable as an SSL vulnerability as say Heartbleed but it certainly refocussed InfoSec teams to finger tight on security. 


### Step 1: Installing Service Mesh 

This post will focus less on spinning up OpenShift Service Mesh 2.x and more on the mTLS between client (the reverse proxy) and the server (our sample httpbin applicaiton). The latter will be accessed via its own INgress Gateway. 

Whilst we'll defer the installation steps to the offiical guide, at the highest of levels, standing up Red Hat OpenShift Servce Mesh (RHOSSM) entails installing four Operators:

- ElasticSearch Operator (by Red Hat)
- Jaeger Operator (by Red Hat)
- Kiali Operator (by Red Hat)
- OpenShift Service Mesh Operator (by Red Hat)

### Step 2: Bringing up the Servie Mesh control plane

So go ahead and install the control plane as per the documneted process, inclduing deplying the "basic" `ServiceMeshControlPlane` Red Hat have provided in the `istio-system` namespace that you will create as a precursive step.

Let's create the namespace where our Istio control plane will live. 

```
$ oc new-project istio-system
```

https://docs.openshift.com/container-platform/4.10/service_mesh/v2x/ossm-create-smcp.html#ossm-control-plane-deploy-cli_ossm-create-smcp

```
$ cat <<EOF | oc apply -f -
apiVersion: maistra.io/v2
kind: ServiceMeshControlPlane
metadata:
  name: basic
  namespace: istio-system
spec:
  version: v2.0
  tracing:
    type: Jaeger
    sampling: 10000
  addons:
    jaeger:
      name: jaeger
      install:
        storage:
          type: Memory
    kiali:
      enabled: true
      name: kiali
    grafana:
      enabled: true
EOF
```

### Step 3: Create our application namespace

We'll move onto crating the namespace where our applciaiton will reside.

```
$ oc new-project httpbin
```

Next, either via the console or CLI we will deploy the **default** `ServiceMeshMemberRoll` object in the RHOSSM control plane namespace of `istio-system`

`ServiceMeshMemberRoll` is the resource we configure to permit namespaces entry to RHOSSM and under the jurisdiction of the `ServiceMeshContorlPlane`.

```
$ cat <<EOF | oc apply -f -
apiVersion: maistra.io/v1
kind: ServiceMeshMemberRoll
metadata:
  name: default
  namespace: istio-system
spec:
  members:
    - httpbin
EOF    
```

We should see the status of the Member Roll as **CONFIGURED** 

```
$  oc get smmr -n istio-system default
NAME      READY   STATUS       AGE
default   1/1     Configured   10s
```

### Step 4: Deploy our applicaiton 

This sample applcaiotn runs a single-replica httpbin as an Istio service taken from this documentation (https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/). When deploying an application, you must opt-in to injection by configuring the annotation `sidecar.istio.io/inject=true` setting up the dpeloyment with an Envoy proxy that is the isiot compeont reposbile for inbound and outboud commecutoin to workload it is tied to. More informatoin can be found here on all the pieces of the puzzle that is Service Mesh and its architecture: https://docs.openshift.com/container-platform/4.10/service_mesh/v2x/ossm-architecture.html


```
$ oc project httpbin
Already on project "httpbin" on server "https://api.cluster-j7cwg.j7cwg.sandbox1228.opentlc.com:6443".
```
```
$ oc apply -f https://raw.githubusercontent.com/istio/istio/release-1.13/samples/httpbin/httpbin.yaml
```

```
$ oc patch deployment httpbin --patch "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"sidecar.istio.io\/inject\":\"true\"}}}}}"
```

**Or** if we'd like to do this at a global level affecting all worklaods in the namespace then we can do this via `oc label`

```
$ oc label namespace httpbin istio-injection=enabled --overwrite
```

We can head over to the Kiali dashboard to vlaidate that the sidecar is present in our httpbin deplyoment. 

Grab the Kiali route:

```
$ oc get route -n istio-system
NAME                                       HOST/PORT                                                                            PATH   SERVICES               PORT         TERMINATION          WILDCARD
...
kiali                                      kiali-istio-system.apps.cluster-j7cwg.j7cwg.sandbox1228.opentlc.com                         kiali                  <all>        reencrypt/Redirect   None
```

Go to Workloads, and view our deployed applciation under the namespace `httpbin`

If we observe a green tick under Health or no Missing Sidecar  warning in Details then we're all clear from a sidecar perspective.


#### Step 5: Generate server certificates and keys for our applcaition

At this point, our applciaiton should be active and reigstered as an application in the mesh. Next we'll use `openssl` to generate our certifcates and keys. In a demo context, we'll resort to creating our own CA and create a self-signed certifcate for our company Sample Corp. 

Frist let's create of Root CA and Prvate Key


```
$ sudo openssl genrsa -out /etc/ssl/certs/rootCAKey.pem 2048
$ sudo openssl req -x509 -sha256 -new -nodes -subj '/O=Example Corp./CN=example.com' -key /etc/ssl/certs/rootCAKey.pem -days 3650 -out /etc/ssl/certs/rootCACert.pem
```

Next, let's generate a Cericate Signning Request which we will then sign.

```
$ SUBDOMAIN=$(oc whoami --show-console  | awk -F'console.' '{print $3}')
$ CN=httpbin.$SUBDOMAIN
$ sudo openssl req -out /etc/ssl/certs/httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout /etc/ssl/certs/httpbin.example.com.key -subj "/CN=${CN}/O=IT Department"
$ sudo openssl x509 -req -days 365 -CA /etc/ssl/certs/rootCACert.pem -CAkey /etc/ssl/certs/rootCAKey.pem -set_serial 0 -in /etc/ssl/certs/httpbin.example.com.csr -out /etc/ssl/certs/httpbin.example.com.crt
```

Finally, we'll store this in TLS secret to later refer to in our Gateway dpelyoment

```
$ oc create secret tls httpbin-crendential --cert=path/to/tls.cert --key=path/to/tls.key -n istio-system 
```

### Step 6: Deploying the INgress Gateway

Gateways in Isitio are applied to the standalone Envoy proxies at the edge of mesh. It's here where we'll assert our TLS configuration and the routing rules in a `VirtualService` which will be bound to the Gateway. 

Again, we'll make use of the example in the Istio docuemnt - only with one iprotant distinction. Instead of the TLS mode of `SIMPLE`, we want to enforce `MUTUAL` 

https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/#configure-a-tls-ingress-gateway-for-a-single-host

we can go ahead and deploy our `Gateway` with the alteration  

```
$ cat <<EOF | oc apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin
  namespace: httpbin
spec:
  selector:
    istio: ingressgateway # use istio default ingress gateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: MUTUAL
      credentialName: httpbin-credential # must be the same as secret
    hosts:
    - httpbin.${SUBDOMAIN}
EOF
```

In OpenShift Service Mesh, everytime we hit dpleoy on a `Gateway`, an OpenShift route is automatically created. Updates and deletes will also be reflected. We can however, disable this aumotaiotn altogether if want to. 

We can verify the route's creation in the `istio-syste` namesapce


```
$ oc get route -n istio-system
NAME                                       HOST/PORT                                                                            PATH   SERVICES               PORT         TERMINATION          WILDCARD
...
httpbin-httpbin-gateway-103ecd597519df74   httpbin.apps.cluster-j7cwg.j7cwg.sandbox1228.opentlc.com                                    istio-ingressgateway   https        passthrough          None
```

From here, we'll create our `VirtualService` which you can see specifies specific paths to route to the backend service. 

```
$ cat <<EOF | oc apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.${SUBDOMAIN}
  gateways:
  - httpbin
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
```

Performing the curl to simulate as if we were just any old client (i.e. only in possession of the Root CA), we are met with a `certificate required` error which is to be expected.

```
$ curl  --cacert example.com.crt https://httpbin.apps.cluster-j7cwg.j7cwg.sandbox1228.opentlc.com/status/418 -I
curl: (56) OpenSSL SSL_read: error:1409445C:SSL routines:ssl3_read_bytes:tlsv13 alert certificate required, errno 0
```

Chanigng our TLS mode to `SIMPLE` on the Gateway would give us an approprate response as client certificat valition wouldn't be required. 

```
$ curl  --cacert example.com.crt https://httpbin.apps.cluster-j7cwg.j7cwg.sandbox1228.opentlc.com/status/418 -I
HTTP/2 418 
server: istio-envoy
```

### Step 7: Client the client's certificate and keys

We essentially repeat the process to create the client’s key and certificate, and perform the self-signature with the CA created earlier.

Recall the **client** in our example will be the NGINX Reverse Proxy that we will create in a later step. 

```
$ openssl req -out nginx.sample.com.csr -newkey rsa:2048 -nodes -keyout nginx.sample.com.key -subj "/CN=nginx.sample.com/O=client organization"
$ openssl x509 -req -sha256 -days 365 -CA sample.com.crt -CAkey sample.com.key -set_serial 1 -in nginx.sample.com.csr -out nginx.sample.com.crt
```

We should be at a point now where we run the same curl with the addition of the nginx certicate and keys and get a valid response back from the server.

```
$ curl --key  /etc/nginx/ssl/reverseproxy.key --cert  /etc/nginx/ssl/reverseproxy.crt --cacert example.com.crt https://httpbin.apps.cluster-j7cwg.j7cwg.sandbox1228.opentlc.com/status/418

    -=[ teapot ]=-

       _...._
     .'  _ _ `.
    | ."` ^ `". _,
    \_;`"---"`|//
      |       ;/
      \_     _/
        `"""`
```
We're now good to tranpose this data over to our nginx config in an upcomhin step.
 

### Step 7: Deploy our NGINX Reverse Proxy

It's time now to deploy our reverse proxy tha acts an intermdieary server between our client and httpbin backend residing in the Service Mesh.

I'll be running NGINXon my bastion server and depending on your running operating system, the installation medthod could vary. For RHEL 8, it's as simple as:

```
$ yum install nginx
```

```
[lab-user@bastion ~]$ cat /etc/nginx/conf.d/proxy.conf
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name  app.example.com;

    listen 443 ssl; 

    # RSA certificate
    ssl_certificate /etc/ssl/certs/app.example.com.crt;
    ssl_certificate_key /etc/ssl/certs/app.example.com.key;

    ssl_client_certificate /etc/nginx/ssl/rootCA.pem;
    ssl_verify_client	  optional;

    # Redirect non-https traffic to https
    if ($scheme != "https") {
        return 301 https://$host$request_uri;
    }
        location /status/418 {
            proxy_pass                    https://httpbin.apps.cluster-j7cwg.j7cwg.sandbox1228.opentlc.com;
            proxy_ssl_certificate         /etc/nginx/client.example.com.crt;
            proxy_ssl_certificate_key     /etc/nginx/client.example.com.key;
            proxy_ssl_protocols           TLSv1 TLSv1.1 TLSv1.2;
            proxy_ssl_ciphers             HIGH:!aNULL:!MD5;
            proxy_ssl_trusted_certificate /etc/ssl/certs/example.com.crt;

            proxy_ssl_verify        on;
       
            proxy_ssl_server_name on;
          
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
}
```

NOTE: For the urposes of this demo we've hijacked port 80 from the  default /tc/nginx/nginx.conf configuration. So left unchanged you will run into conflict issues when you attempt to restart the `nginx` service.

We've included the `reverseproxy.crt` and `reverseproxy.key` as both the client **AND** server certificate.key pair in this instance. This certificate is presented, when asked, to connections attempting to contact this virtual proxy server in addiotn to represneting the ssl certificate that will be present to the backend server; identified by `proxy_pass` which is our exposed httpbin application from the mesh.  

`ssl_client_certifcates` switch that enables/disables the reverse proxy's server's certificate authentication behavior. Here we've set it to `optional`. In an internla corporate environment may this mTLS between the browser and the reverse proxy in most circumstances would be deemed overkill.  most cetinaly covering all bases and 

### Step 8: Test the OpenShift Service Mesh with mTLS enabled



