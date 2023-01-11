---
layout:     post
title:      "ChatGPT, the Kube Administrator"
subtitle:   ""
description: "ChatGPT, the Kube Administrator"
excerpt: "Secondary Scheduler for OpenShift"
date:       2023-01-07
author:         "Will Cushen"
image: "/img/2023-01-chatgpt-kube-deploy/chatgpt.jpeg"
published: true
tags:
    - Kubernetes
    - AI
#categories: [ Tech ]
URL: "/2023-01-chatgpt-kube-deploy"
---

## ChatGPT 

For two years running, the year's most talked about tech story in the midst of milestones like Black Friday and the Christmas winddown that herald the winding down of the yeart. In 2021, it was lo4, Log4j is a Java-based logging utility,  that usually spell trend (log4j in 2021) In the midst of the frantic that where events like Black Friday and a general wind down toward Christmas, one tech trend still managed to cathc the eyes of most and that was CHatGPT. Many are calling it the most promtient innovation in  social engineering since Google's search engines. 

Intorcued by OpenAI on NOvember 30 last year, there's no shortage of beliweildermeant in the tool's abaility to provide human-quality answers to quite literally antyhing that's thrown at it. 

CHatGPT is a large language model (LLM) based on GPT-3.5, the third generation Generative Pre-trained Transformer, that aoccrding to [Standford UNiversity](https://hai.stanford.edu/news/how-large-language-models-will-transform-science-society-and-ai) "has 175 billion parameters and was trained on 570 gigabytes of text. For comparison, its predecessor, GPT-2, was over 100 times smaller at 1.5 billion parameters."

Again, as with the entering of any new frontier in AI, ChatGPT really has poeple (including me!) restate the age old quesiton, "will this thing take my job?".

Certainly, its postion to supersede human aptitude is already in question, inlcuding what it means for the future of AI in education. A [school in New York](https://www.abc.net.au/news/2023-01-08/artificial-intelligence-chatgpt-chatbot-explained/101835670) has recently banned the tool  banning it out fo the fear that it could write A+ essays on the AMerican VCivil War. 

But conversely, there's many out there that are taking a glass half-full approach and spekaing to ways in which ChatGPT can aid certain vocations, such as software delopement as explained [here](https://medium.com/geekculture/7-ways-chatgpt-can-help-you-to-become-a-better-developer-2c490d7a387d)

### Kubernetes Day Job

In this blog, I too am going to see how much CHatGPt can help (replace) me in my day gig mananging Kubernestes clusters. 

We'll throw it some Kube 101 queires and then go on to see what it can provide on more complex topics such as service mesh.

#### Demo

{{< vimeo 787798457 >}}

A couple of things to note here: 

1. I ran the same script of four promtps apprimately 4-5 times and there was some variaition in the reposne for most of them, aside maybe from scaling up the deplyoment to two replicas. 
2. It didn't capture it in the recorded video demo, but on other occasions when I asked it to deploy to an actual cluster, it reaffrimed its constraints stating, "I'm sorry, but I am a text-based AI and do not have the ability to interact with external systems." So as expected, it cannot actually deploy a workload for you. 
3. WIth respect to Service Mehs, I chopped and changed in my testing between referring to Istio and OpenShift Servcie Mesh and only on one occasion did I recevie direciton about installing a Gteway and VirtualService, necessary fro getting traffic into the mesh from externally.


{{% notice info %}}
OpenShift Service Mesh is actually based on an opiotniosted distbituon of Istio called Maistra and it differs to generic Istio by which the latter inject the required Envoy sidecar into all deployments within labeled projects whereas the Maistra version relies on presence of the `sidecar.istio.io/inject` annotation AND the project being listed in the `ServiceMeshMemberRoll`.
{{% /notice %}}


#### Following ChatGPT's Istio orders 

So with all that said, I combined the best-of-breed reposnes from my numerous askings to ChatGPT and executed the YAML it provied me on the cluster.


{{% notice info %}}
Given my line of quesitoning, I'm going to give it benefit of the doubt and assume it believed that I had already installed OpenShift Sevrice Mesh and enabled a namespace for to be managed by the Service Mesh Control Plane. 
{{% /notice %}}


As you saw in the video, CHatGPT rendered a Deployment anifest that I took and applied to the cluster. 

From there I exposed the deployment. 

```yaml
$ oc expose deployment/nginx --port=80
service/nginx exposed
```

{{% notice warning %}}
Remember that you may have to do some SCC tickery here
{{% /notice %}}


...And then went on to create the `Gateway` and `VirtualService` (that it told me to create after some continual nudging on my behalf!)

```yaml
oc apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: my-mesh-enabled-namespace
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
EOF
```


```yaml
oc apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: nginx-vs
  namespace: my-mesh-enabled-namespace
spec:
  hosts:
  - "*"
  gateways:
  - nginx-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: nginx
        port:
          number: 80
EOF
```

I'm prepared to give CHatGPT a pass mark despite the fact that it did not instruct me to retirve the GATEWAY_URL to access NGINX's welcome page

```yaml
$ export GATEWAY_URL=$(oc -n istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}')
```

![](/img/2023-01-chatgpt-kube-deploy/nginx-welcome.png)

## Wrap Up

To keep in theme with this blog post (as well as finding an opportunity to lazily outsource), I thought I'd hand over the reigns of this conclusion to CHatGPT and see what they had to say 


Provide me with a conclusion to a blog post that discusses the use of ChatGPT to develop Kubernetes manifests and provide guidance on how to deploy to a Kubernetes cluster. 

_"In conclusion, ChatGPT can be a useful tool for generating Kubernetes manifests and providing guidance on how to deploy to a Kubernetes cluster. It is able to quickly generate manifest files and provide recommendations based on user input, making it a time-saving and convenient tool for those working with Kubernetes. However, it is important to note that ChatGPT is not a replacement for in-depth knowledge of Kubernetes and should be used as a supplement to, rather than a substitute for, manual configuration and understanding of the system. Overall, ChatGPT can be a valuable tool for those looking to streamline their Kubernetes workflow and deploy applications efficiently."_

I like this answer for two reasons. First, it perfectly sums ups the fucntion  ChatGPT serves for software devleopers and infrastrucute engineers, a 
time-saving tool that makes it easier for folks to get their job done and focus on more important (a la innovative) tasks. Second, it's reaffriming the role of humans to not take it's output as gospel and always (always) thoroughly review and test any manifests or deployment instructions generated by the tool before implementing them in a production environment - TL;DR, our jobs are still safe tomorrow. 

ChatGPT is ana amzing playgorund to tinker in and I hope you dive right in  to see what it can do for you!