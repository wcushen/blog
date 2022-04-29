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

Most in the IT industry now would be well acquainted with the meteoric rise of microservices that are paving the way for organisations to respond quicker to changing business needs and capitalise on the elasticity of the cloud. Going from monolithic to microservice is not without it's apparent challenges; more moving parts equates to more overhead, that is if managed improperly. 

The array of technologies in the microservices ecosystem are designed to help engineers and architects improve the security, observability and traffic control of an organisation's applications.

On the heels of last year's Log4J and Log4Shell vulnerabilities*, irrespective of vertical, a zero-trust security framework is becoming a model organisations **want** but most are really in their infancy when it comes to adoption at 21% according this [poll](https://venturebeat.com/2022/02/15/report-only-21-of-enterprises-have-adopted-zero-trust-architecture/) from Optiv Security. 

mTLS presents an option on the table for us to work towards zero trust. 

### What is mTLS?

Usually when one visits a browser with HTTPS prefixed ot the URL ; the browser will attempt to validate the server's certificate to make sure they're really who they say they are. 

mTLS upholds this Zero Trust paradigm, extending typical service-side SSL by _also_ requiring the client to present its certificates for validation as part of the TLS handshake. 

## About the example

For those that are familiar with Service Mesh mTLS capabilities, I bet in most circumstances your exposure has been in the form of mTLS between microservices running **inside** the mesh. In this post we'll look at example of an external server communicating with an application running inside an OpenShift Service Mesh 2.2 enabled namespace. This external server will act as a reverse proxy to facilitate the ingress routing to our mesh. 

_Why stand up an external Load Balancer when we can run a perfectly adequate in ingress gateway at the mesh edge?_

With an L7 external load balancer outside the mesh we're attempting to depict a common enterprise scenario with a GTM/LTM fronting various virtual and containerised applications and offloading features such as DDoS mitigation and traffic filtering to a dedicated appliance. Additionally, we may be tailoring our solution for mutli-cluster and mutli-regional load balancing, the latter particularly relevant to the cloud. 

Moreover, security benchmarks such as the Payment Card Industry - Data Security Standard (PCI - DSS) explicitly requires credit card information to be stored in internal networks segregated from your DMZ. 

The example in this article represents are very stripped down, rudimentary setup of an NGINX Reverse Proxy fronting an **httpbin** workload running in an instance of OpenShift Service Mesh 2.x; a birthchild of the Istio project. The intention is to highlight a prominent security feature in mTLS that is becoming increasingly sought after, although not unique to, in microservices and service mesh technologies alike. 

*Although Log4J isn't a straight up SSL vulnerability as say Heartbleed some years ago, it certainly refocussed InfoSec teams to consider the robustness of their risk mitigation strategies.


### Step 1: Installing Service Mesh 

This post will focus less on spinning up OpenShift Service Mesh 2.x and more on the mTLS between client (the reverse proxy) and the server (our sample httpbin application). The latter will be accessed via its own INgress Gateway. 

Whilst we'll defer the installation steps to the offiical guide, at the highest of levels, standing up Red Hat OpenShift Servce Mesh (RHOSSM) entails installing four Operators:

- ElasticSearch Operator (by Red Hat)
- Jaeger Operator (by Red Hat)
- Kiali Operator (by Red Hat)
- OpenShift Service Mesh Operator (by Red Hat)

### Step 2: Bringing up the Servie Mesh control plane

So go ahead and install the control plane as per the [documented steps](https://docs.openshift.com/container-platform/4.10/service_mesh/v2x/ossm-create-smcp.html#ossm-control-plane-deploy-cli_ossm-create-smcp), inclduing deplying the "basic" `ServiceMeshControlPlane` Red Hat have provided in the `istio-system` namespace that you will create as a precursive step.

Let's create the namespace where our Istio control plane will live. 

```yaml
$ oc new-project istio-system
```

```yaml
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

```yaml
$ oc new-project httpbin
```

Next, either via the console or CLI we will deploy the **default** `ServiceMeshMemberRoll` object in the RHOSSM control plane namespace of `istio-system`

`ServiceMeshMemberRoll` is the resource we configure to permit namespaces entry to RHOSSM and under the jurisdiction of the `ServiceMeshContorlPlane`.

```yaml
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

```yaml
$  oc get smmr -n istio-system default
NAME      READY   STATUS       AGE
default   1/1     Configured   10s
```

### Step 4: Deploy our applicaiton 

This sample applcaiotn runs a single-replica httpbin as an Istio service taken from this documentation (https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/). When deploying an application, you must opt-in to injection by configuring the annotation `sidecar.istio.io/inject=true` setting up the dpeloyment with an Envoy proxy that is the isiot compeont reposbile for inbound and outboud commecutoin to workload it is tied to. More informatoin can be found here on all the pieces of the puzzle that is Service Mesh and its architecture: https://docs.openshift.com/container-platform/4.10/service_mesh/v2x/ossm-architecture.html

{{% notice warning %}}
To make this pod run in OpenShift, we need to  allow it use the SCC of `anyuid` which we'll bind to the `default` service account of the `httpbin` namespce
{{% /notice %}}

```yaml
$ oc project httpbin
$ oc apply -f https://raw.githubusercontent.com/istio/istio/release-1.13/samples/httpbin/httpbin.yaml
$ oc patch deployment httpbin --patch "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"sidecar.istio.io\/inject\":\"true\"}}}}}"
```

**Or** if we'd like to do this at a global level affecting all worklaods in the namespace then we can do this via `oc label`

```yaml
$ oc label namespace httpbin istio-injection=enabled --overwrite
```

```yaml
$ oc adm policy add-scc-to-user anyuid -z httpbin && \
oc patch deployment/httpbin --patch \
   "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"last-restart\":\"`date +'%s'`\"}}}}}"
```
 
{{% notice tip %}}
We can head over to the Kiali dashboard to vlaidate that the sidecar is present in our httpbin deplyoment.
{{% /notice %}} 

Grab the Kiali route:

```yaml
$ oc get route kiali -n istio-system
NAME    HOST/PORT                                                             PATH   SERVICES   PORT    TERMINATION          WILDCARD
kiali   kiali-istio-system.apps.cluster-xlp6m.xlp6m.sandbox1808.opentlc.com          kiali      <all>   reencrypt/Redirect   None
```

Go to **Workloads***, and view our deployed applciation under the namespace `httpbin`

{{< rawhtml >}}
<p>
If we observe a green tick under Health or no <img style='display:inline;' src='/img/2022-04-mtls-service-mesh-nginx/missing-sidecar.png'/>warning in Details then we're all clear from a sidecar perspective.
</p>
{{< /rawhtml >}}

![](/img/2022-04-mtls-service-mesh-nginx/httpbin-kiali-health.png)

#### Step 5: Generate server certificates and keys for our applcaition

At this point, our applciaiton should be active and reigstered as an application in the mesh. Next we'll use `openssl` to generate our certifcates and keys. In a demo context, we'll resort to creating our own CA and create a self-signed certifcate for our company Example Corp. 

Frist let's create of Root CA and Prvate Key


```yaml
$ mkdir /var/tmp/{certs,private}
$ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=Example Corp./CN=example.com' -keyout /var/tmp/private/rootCAKey.pem -out /var/tmp/certs/rootCACert.pem
```

Next, let's generate a Cericate Signning Request which we will then sign.

```yaml
$ SUBDOMAIN=$(oc whoami --show-console  | awk -F'console.' '{print $3}')
$ CN=httpbin.$SUBDOMAIN
$ openssl req -out /var/tmp/certs/httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout /var/tmp/private/httpbin.example.com.key -subj "/CN=${CN}/O=httpbin organization"
$ openssl x509 -req -sha256 -days 365 -CA /var/tmp/certs/rootCACert.pem -CAkey /var/tmp/private/rootCAKey.pem -set_serial 0 -in /var/tmp/certs/httpbin.example.com.csr -out /var/tmp/certs/httpbin.example.com.crt
```

Finally, we'll store this in TLS secret to later refer to in our Gateway dpelyoment

```yaml
$ oc create secret generic httpbin-credential --from-file=tls.crt=/var/tmp/certs/httpbin.example.com.crt --from-file=tls.key=/var/tmp/private/httpbin.example.com.key --from-file=ca.crt=/var/tmp/certs/rootCACert.pem -n istio-system
```

### Step 6: Deploying the INgress Gateway

Gateways in Isitio are applied to the standalone Envoy proxies at the edge of mesh. It's here where we'll assert our TLS configuration and the routing rules in a `VirtualService` which will be bound to the Gateway. 

Again, we'll make use of the example in the Istio docuemnt - only with one iprotant distinction. Instead of the TLS mode set to `SIMPLE`, we want to enforce `MUTUAL`. 

https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/#configure-a-tls-ingress-gateway-for-a-single-host

we can go ahead and deploy our `Gateway` with the alteration  

```yaml
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

We can verify the route's creation in the `istio-system` namesapce


```yaml
$ oc get route -n istio-system -l maistra.io/gateway-name=httpbin
NAME                               HOST/PORT                                                  PATH   SERVICES               PORT    TERMINATION   WILDCARD
httpbin-httpbin-dcfcfc0729048e03   httpbin.apps.cluster-xlp6m.xlp6m.sandbox1808.opentlc.com          istio-ingressgateway   https   passthrough   None
```

From here, we'll create our `VirtualService` which you can see specifies specific paths to route to the backend service. 

```yaml
$ cat <<EOF | oc apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
  namespace: httpbin
spec:
  hosts:
  - "httpbin.${SUBDOMAIN}"
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

And validate its deployment

```yaml
$ oc get virtualservice -n httpbin
NAME      GATEWAYS      HOSTS                                                          AGE
httpbin   ["httpbin"]   ["httpbin.apps.cluster-gvh54.gvh54.sandbox1250.opentlc.com"]   20s
```


Performing the curl to simulate as if we were just any old client (i.e. only in possession of the Root CA), we are met with a `certificate required` error which is to be expected.

```yaml
$ curl  --cacert /var/tmp/certs/rootCACert.pem https://httpbin.apps.cluster-j7cwg.j7cwg.sandbox1228.opentlc.com/status/418 -I
curl: (56) OpenSSL SSL_read: error:1409445C:SSL routines:ssl3_read_bytes:tlsv13 alert certificate required, errno 0
```

Chanigng our TLS mode to `SIMPLE` on the Gateway would give us an approprate response as client certificat valition wouldn't be required. 

```yaml
$ curl --cacert /var/tmp/certs/rootCACert.pem https://httpbin.${SUBDOMAIN}/status/418

    -=[ teapot ]=-

       _...._
     .'  _ _ `.
    | ."` ^ `". _,
    \_;`"---"`|//
      |       ;/
      \_     _/
        `"""`

```

### Step 7: Client the client's certificate and keys

We essentially repeat the process to create the client’s key and certificate, and perform the self-signature with the CA created earlier.

Recall the **client** in our example will be the NGINX Reverse Proxy that we will create in a later step. 

Let's create this certificate as wildcard with a Subject Alternative Name (SAN) to cater for our frontend route that we'll hit as a concluding step.  


```yaml
cat <<EOF >>req.cfg
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[req_distinguished_name]

[v3_req]
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = *.example.com
DNS.2 = *.frontend.example.com
```

And now we're clear to create our self-signed certificate

```yaml
$ openssl req -out /var/tmp/certs/example.com.csr -newkey rsa:2048 -nodes -keyout /var/tmp/private/example.com.key -subj "/CN=reverseproxy.example.com/O=Example Org" -config req.cfg

$ openssl x509 -req -sha256 -days 365 -CA /var/tmp/certs/rootCACert.pem -CAkey /var/tmp/private/rootCAKey.pem  -CAcreateserial -in /var/tmp/certs/example.com.csr -out /var/tmp/certs/example.com.crt -extensions v3_req -extfile req.cfg
```


We should be at a point now where we run the same curl with the addition of the nginx certicate and keys and get a valid response back from the server.

```yaml
$ [lab-user@bastion ~]$ curl --key /var/tmp/private/example.com.key --cert /var/tmp/certs/example.com.crt --cacert /var/tmp/certs/rootCACert.pem https://httpbin.${SUBDOMAIN}/status/418

    -=[ teapot ]=-

       _...._
     .'  _ _ `.
    | ."` ^ `". _,
    \_;`"---"`|//
      |       ;/
      \_     _/
        `"""`
```
We're now good to tranpose these certificates and keys over to our nginx config in an upcomhin step.
 

### Step 7: Deploy our NGINX Reverse Proxy

It's time now to deploy our reverse proxy tha acts an intermdieary server between our client and httpbin backend residing in the Service Mesh.

I'll be running NGINXon my bastion server and depending on your running operating system, the installation medthod could vary. For RHEL 8, it's as simple as:

```yaml
$ sudo yum install nginx -y
```

We'll copy over all our created certficates and keys to more friendly driecotries for NGINX:

```yaml
sudo mkdir -p /etc/nginx/ssl/{certs,private} && sudo cp /var/tmp/certs/* /etc/nginx/ssl/certs && sudo cp /var/tmp/private/* /etc/nginx/ssl/private
```
{{% notice warning %}}
For the purposes of this demo we've hijacked port 80 from the  default `/etc/nginx/nginx.conf` configuration. So left unchanged you will run into conflict issues when you attempt to restart the `nginx` service.
{{% /notice %}}

Below is the compelte exceprt of our reverse proxy config. Basically, we're setting NGINX to listen for all traffic on port 80 and 443. The former we wnt ot rewrite to the latter which will be expressed via a 301 return code. 

The `proxy_pass` command directs all traffic on 443 to our **httpbin** Ingress Gateway on the edge of the mesh with the approprtiate client certs that we validated in the previous `curl`.

```yaml
$ cat /etc/nginx/conf.d/proxy.conf 
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name  _;

    listen 443 ssl; 

    # RSA certificate
    ssl_certificate /etc/nginx/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/private/example.com.key;

    ssl_client_certificate /etc/nginx/ssl/certs/rootCACert.pem;
    ssl_verify_client	  optional;

    # Redirect non-https traffic to https
    if ($scheme != "https") {
        return 301 https://$host$request_uri;
    }
        location /status/418 {
            proxy_pass                    https://httpbin.${SUBDOMAIN};
            proxy_ssl_certificate         /etc/nginx/ssl/certs/example.com.crt;
            proxy_ssl_certificate_key     /etc/nginx/ssl/private/example.com.key;
            proxy_ssl_protocols           TLSv1 TLSv1.1 TLSv1.2;
            proxy_ssl_ciphers             HIGH:!aNULL:!MD5;
            proxy_ssl_trusted_certificate /etc/nginx/ssl/certs/rootCACert.pem;

            proxy_ssl_verify        on;
       
            proxy_ssl_server_name on;
          
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
}

```

We've included the `example.com.crt` and `example.com.key` as both the client **AND** server certificate/key pair in this instance. This certificate is presented, when asked, to connections attempting to contact the reverse proxy server in addiotn to represneting the **ssl** certificate that will be present to the backend server; identified by `proxy_pass` which is our exposed httpbin application from the mesh.  

`ssl_client_certifcates` switch that enables/disables the reverse proxy's server's certificate authentication behavior. Here we've set it to `optional`. This is easier to manage in an intenral corpoate environment where you could have more or more endorsed root Certifcates, concaentaed into a single file. Outside this though, the overhead of managing an accepted list of CAs of publicy-accesible browsers would be burdensome, to put it lightly. 

### Step 8: Test the OpenShift Service Mesh with mTLS enabled

We're going to set up local DNS to allow `httpbin.frontend.example.com` to resolve to our localhost NGINX server.

And with any luck we _should_ end up with...

```yaml
$ curl --cacert /var/tmp/certs/rootCACert.pem  https://httpbin.frontend.example.com/status/418 

    -=[ teapot ]=-

       _...._
     .'  _ _ `.
    | ."` ^ `". _,
    \_;`"---"`|//
      |       ;/
      \_     _/
        `"""`
```

:tada: :tada:

## Wrap Up

Tihs was just a fun dabble demonstrating mTLS communcatication from outside Istio. Certinaly there's benefit in drawing from some of the security architeicture in here that coupld be mpletmened in a scale-out enterprise environment - it's a balacne of _risks_ and _needs_; and where and how we temrinate SSL is certinaly one of those considerations. 

It should be noted that as of OpenShift 4.9, mTLS authenticaiton can be enabled in the Ingress Controller, so there are other, argulably simpler ways if we want _just_ want to cherry-pick certain security features that were only previously on Istio's bumper sticker.

I hope you got some value from the above walkthrough and most importantly, hopefully forwards any discussion you might be having in your team on how you treat your Kubernetes/Service Mesh SSL. 