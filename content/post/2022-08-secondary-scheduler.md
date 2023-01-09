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


create CM for the KubeSchedulerConfiguration (the config file has to stored under config.yaml). E.g.:





```yaml
$ oc project httpbin && \
oc apply -f https://raw.githubusercontent.com/istio/istio/release-1.13/samples/httpbin/httpbin.yaml && \
oc patch deployment httpbin --patch "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"sidecar.istio.io\/inject\":\"true\"}}}}}"
```

**_OR_** if we'd like to do this at a **global** level affecting all workloads in the namespace then we can do this via `oc label`

```yaml
$ oc label namespace httpbin istio-injection=enabled --overwrite
```

```yaml
$ oc adm policy add-scc-to-user anyuid -z httpbin && \
oc rollout restart deployment/httpbin
```
 
{{% notice tip %}}
We can head over to the Kiali Dashboard to validate that the sidecar is present in our `httpbin` deployment.
{{% /notice %}} 

Grab the Kiali route:

```yaml
$ oc get route kiali -n istio-system
NAME    HOST/PORT                                       PATH   SERVICES   PORT    TERMINATION          WILDCARD
kiali   kiali-istio-system.apps.cluster-sandbox-1.com          kiali      <all>   reencrypt/Redirect   None
```

Go to **Workloads**, and view our deployed application under the namespace `httpbin`.

{{< rawhtml >}}
<p>
If we observe no <img style='display:inline;' src='/img/2022-05-service-mesh-reverse-proxy-mtls/missing-sidecar.png'/>warning then we're all clear from a sidecar perspective.
</p>
{{< /rawhtml >}}

![](/img/2022-05-service-mesh-reverse-proxy-mtls/httpbin-kiali-health.png)

### Step 5: Generating server certificates and keys for our application

At this point, our application should be active and registered as an application in the mesh. Next we'll use `openssl` to generate our certificates and keys. For the context of this demo, we'll resort to creating our own CA and create a self-signed certificate for our company Example Corp. 

First let's create our **Root CA** and **Private Key**.


```yaml
$ mkdir /var/tmp/{certs,private} && \
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=Example Corp./CN=example.com' -keyout /var/tmp/private/rootCAKey.pem -out /var/tmp/certs/rootCACert.pem
```

Next, let's generate a **Certificate Signing Request (CSR)** which we will then sign.

```yaml
$ SUBDOMAIN=$(oc whoami --show-console  | awk -F'console.' '{print $3}') && \
CN=httpbin.$SUBDOMAIN && \
openssl req -out /var/tmp/certs/httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout /var/tmp/private/httpbin.example.com.key -subj "/CN=${CN}/O=httpbin organization" && \
openssl x509 -req -sha256 -days 365 -CA /var/tmp/certs/rootCACert.pem -CAkey /var/tmp/private/rootCAKey.pem -set_serial 0 -in /var/tmp/certs/httpbin.example.com.csr -out /var/tmp/certs/httpbin.example.com.crt
```

Finally, we'll store this in an OpenShift Secret to later refer to in our Gateway deployment.

```yaml
$ oc create secret generic httpbin-credential --from-file=tls.crt=/var/tmp/certs/httpbin.example.com.crt --from-file=tls.key=/var/tmp/private/httpbin.example.com.key --from-file=ca.crt=/var/tmp/certs/rootCACert.pem -n istio-system
```

### Step 6: Deploying the Ingress Gateway

Gateways in Istio are applied to the standalone Envoy proxies at the edge of the mesh. It's here where we'll assert our TLS configuration and the routing rules defined within a `VirtualService` which will be bound to the Gateway. 

Again, we'll make use of the example in the Istio [documentation](https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/#configure-a-tls-ingress-gateway-for-a-single-host) - only with one _important distinction_. Instead of the TLS mode set to `SIMPLE`, we want to enforce `MUTUAL`. 

We can go ahead and deploy our `Gateway` with the alteration. 

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

In OpenShift Service Mesh, every time we deploy a `Gateway`, an OpenShift route is automatically created. Updates and deletes will also be reflected. We can however, disable this automation altogether if we want to. 

We can verify the route's creation in the `istio-system` namespace.


```yaml
$ oc get route -n istio-system -l maistra.io/gateway-name=httpbin
NAME                               HOST/PORT                            PATH   SERVICES               PORT    TERMINATION   WILDCARD
httpbin-httpbin-dcfcfc0729048e03   httpbin.apps.cluster-sandbox-1.com          istio-ingressgateway   https   passthrough   None
```

From here, we'll create our `VirtualService` which you can see embeds certain paths to route to the backend service. 

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

Let's validate its deployment:

```yaml
$ oc get virtualservice -n httpbin
NAME      GATEWAYS      HOSTS                                    AGE
httpbin   ["httpbin"]   ["httpbin.apps.cluster-sandbox-1.com"]   20s
```

Performing the `curl` to simulate as if we were just any old client in possession of the **Root CA**, we are met with a `certificate required` error which is to be expected.

```yaml
$ curl  --cacert /var/tmp/certs/rootCACert.pem https://httpbin.apps.cluster-sandbox-1.com/status/418 -I
curl: (56) OpenSSL SSL_read: error:1409445C:SSL routines:ssl3_read_bytes:tlsv13 alert certificate required, errno 0
```

Changing our TLS mode to `SIMPLE` on the Gateway would return a response as client certificate validation wouldn't be required. 

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

### Step 7: Creating the client's keys and certificates

We essentially repeat the process to create the client’s key and certificate, and perform the self-signature with the CA created earlier.

Recall that the **client** in our example will in fact be the NGINX Reverse Proxy that we will stand up in a moment. 

Let's create this certificate as a wildcard with a **Subject Alternative Name (SAN)** to cater for our frontend route that we'll access as a concluding step.  


```yaml
cat <<EOF > req.cfg
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
EOF
```

And now we're clear to create our self-signed certificate.

```yaml
$ openssl req -out /var/tmp/certs/example.com.csr -newkey rsa:2048 -nodes -keyout /var/tmp/private/example.com.key -subj "/CN=reverseproxy.example.com/O=Example Org" -config req.cfg && \
openssl x509 -req -sha256 -days 365 -CA /var/tmp/certs/rootCACert.pem -CAkey /var/tmp/private/rootCAKey.pem  -CAcreateserial -in /var/tmp/certs/example.com.csr -out /var/tmp/certs/example.com.crt -extensions v3_req -extfile req.cfg
```

We should be at the point now where we run the same `curl` with the addition of the NGINX certificate and keys and get a valid response back from the server.

```yaml
$ curl --key /var/tmp/private/example.com.key --cert /var/tmp/certs/example.com.crt --cacert /var/tmp/certs/rootCACert.pem https://httpbin.${SUBDOMAIN}/status/418

    -=[ teapot ]=-

       _...._
     .'  _ _ `.
    | ."` ^ `". _,
    \_;`"---"`|//
      |       ;/
      \_     _/
        `"""`
```
With that success, we're now good to transpose these certificates and keys over to our NGINX config in an upcoming step.

### Step 8: Deploying our NGINX Reverse Proxy

It's time now to deploy our reverse proxy that will act as an intermediary server between our client and `httpbin` backend residing in the Service Mesh.

This example will run NGINX on a bastion server and depending on your running operating system, the installation method could vary. For RHEL 8, it's as simple as:

```yaml
$ sudo yum install nginx -y
```

We'll copy over all our created certificates and keys to _more friendly_ directories for NGINX:

```yaml
sudo mkdir -p /etc/nginx/ssl/{certs,private} && \ 
sudo cp /var/tmp/certs/* /etc/nginx/ssl/certs && \ 
sudo cp /var/tmp/private/* /etc/nginx/ssl/private
```

{{% notice warning %}}
For the purposes of this demo we've hijacked port 80 from the default `/etc/nginx/nginx.conf` configuration. So left unchanged you will run into conflict issues when you attempt to restart the `nginx` service.
{{% /notice %}}

Below is the complete excerpt of our reverse proxy config. Basically, we're setting NGINX up to listen for all traffic on port 80 and 443. The former we want to rewrite to the latter which will be expressed via a 301 return code. 

The `proxy_pass` command directs all traffic on 443 to our `httpbin` Ingress Gateway on the edge of the mesh with the appropriate client certs that we validated in the previous `curl`.

```yaml
$ sudo cat <<EOF > /etc/nginx/conf.d/proxy.conf 
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
            proxy_pass                    https://httpbin.apps.cluster-sandbox-1.com;
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
EOF
```

We've included the `example.com.crt` and `example.com.key` as both the client **AND** server certificate/key pair in this instance. This certificate is presented, when asked, to connections attempting to contact the reverse proxy server _in addition_ to representing the SSL certificate that will be presented to the backend server; identified by `proxy_pass` which is our exposed `httpbin` application from the mesh.  

`ssl_client_certificates` is a switch that enables/disables the reverse proxy server's certificate authentication behavior. Here we've set it to `optional`. It really depends on how you are serving your application, but given ours pertains to a simple HTTP Request & Response Service (`httpbin`) - a `curl` command or browser will be our front door. Authenticating for client certificates in this context may perhaps be easier to manage in an internal corporate environment where you could have more or more trusted root certificates, concatenated into a single file. Outside this though, the overhead of managing an accepted list of CAs through various vendors' trust programs could prove burdensome.

### Step 9: Testing our `httpbin` endpoint with mTLS enabled

We're going to set up local DNS to set up `httpbin.frontend.example.com` to resolve to our localhost NGINX server.

Enable and restart the `nginx` service.

```yaml
sudo systemctl enable nginx && \
sudo systemctl restart nginx
```

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

This was just a fun dabble demonstrating mTLS communication from outside Istio. Certainly there's benefit in drawing from _some_ of the security architecture in here that could be implemented in a scale-out enterprise environment - it's a balance of _risks_ and _needs_; and **where** and **how** we terminate SSL is certainly one of those considerations. 

It should be noted that as of OpenShift 4.9, mTLS authentication can be enabled in the Ingress Controller, so there are other, arguably simpler ways if we want _just_ want to cherry-pick certain security features that were only previously on Istio's bumper sticker, at least in the OpenShift space :smile:

I hope you got some value from the above walkthrough and most importantly, hopefully forwards any discussions you might be having in your team on how you treat your Kubernetes/Service Mesh SSL. 