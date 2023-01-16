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

## What is ChatGPT? 

For two years running, the year's most talked about tech story came at a time where most are caught up the end of year wind down, with Black Friday and the festive season serving as the ultimate distractions for most. In 2021, it was the [log4j scare](https://www.infosecurity-magazine.com/news-features/log4j-vulnerability-12-months-on/), the exposed vulnerability of a Java-based logging utility, and again 2022 gave birth to what many are calling the most progressive leap forward in natural language processing since Google's search engine. 

Introduced to the world by OpenAI on November 30 last year, there's been no shortage of bewilderment out there in the tool's ability to provide human-quality answers to quite literally anything that's thrown at it. 

ChatGPT is a _large language model_ (LLM) based on GPT-3.5, the third generation Generative Pre-trained Transformer, that according to [Stanford University](https://hai.stanford.edu/news/how-large-language-models-will-transform-science-society-and-ai) **"has 175 billion parameters and was trained on 570 gigabytes of text. For comparison, its predecessor, GPT-2, was over 100 times smaller at 1.5 billion parameters."**

As we seemingly enter yet another new frontier in AI, the emergence of ChatGPT has again invoked the age-old question, _"will this thing take my job?"_.

Certainly, its position to supersede human aptitude is already in question, including what it means for the future of AI in education as an example. A [school in New York](https://www.abc.net.au/news/2023-01-08/artificial-intelligence-chatgpt-chatbot-explained/101835670) has recently banned the tool, prohibiting it out of the fear that it could write A+ essays on the American Civil War and Australian uni's are [considering a retreat](https://www.theguardian.com/australia-news/2023/jan/10/universities-to-return-to-pen-and-paper-exams-after-students-caught-using-ai-to-write-essays) to 'pen and paper' exams.

But conversely, there's many out there that are taking a _glass half-full_ approach and speaking to ways in which ChatGPT can aid certain vocations, such as software development as explained [here](https://medium.com/geekculture/7-ways-chatgpt-can-help-you-to-become-a-better-developer-2c490d7a387d).

Unsurprisingly, the advancement of ChatGPT is moving at a scary pace. At the time of writing, supposedly OpenAI is on the verge (Q1 of 2023) of releasing **GPT-4**. The rumour mill's in overdrive, but reports are hinting at a bump up to [1 trillion](https://thechainsaw.com/business/chatgpt-4-vs-chatgpt-3-latest-development/) model-fed parameters. Mind-boggling. 

### Handing AI the keys

In this blog, I too am going to put ChatGPT to the test and see how much it can help (replace) me in my day gig managing Kubernetes clusters. 

We'll throw it some _Kube 101_ queries and then go on to gauge if it can assist on more complex topics such as service mesh (Istio).

Watch the below video for the interrogation :smiling_imp:

{{< vimeo 787798457 >}}

A couple of things to note here: 

1. I ran the same script of four prompts approximately a dozen times and there was some variation in the responses for most of them, aside maybe from scaling up the deployment to two replicas (i.e. something very straightforward). 
2. It wasn't captured in the recorded video demo, but on other occasions when I asked ChatGPT to deploy to an actual cluster, it reaffirmed its constraints stating, _"I'm sorry, but I am a text-based AI and do not have the ability to interact with external systems."_ So as expected, it cannot **literally** deploy a workload for you. 
3. With respect to Service Mesh, I chopped and changed in my testing between referring to _Istio_ and _OpenShift Service Mesh_ and only on **one** occasion did I receive direction about installing a Gateway and VirtualService, necessary for getting traffic into the mesh from externally.

{{% notice info %}}
OpenShift Service Mesh is actually based on an opinionated distribution of Istio called **Maistra** and it differs to generic Istio by which the latter inject the required Envoy sidecar into all deployments within labeled projects whereas the Maistra version relies on presence of the `sidecar.istio.io/inject` annotation AND the project being listed in the `ServiceMeshMemberRoll`.
{{% /notice %}}


#### Deploying an Istio microservice - _according to ChatGPT_

So with all that said, I combined the _best-of-breed_ responses from my numerous askings to ChatGPT and executed the YAML it provided me on the cluster.

{{% notice info %}}
Given my line of questioning, I'm going to give it benefit of the doubt and assume it believed that I had already installed OpenShift Service Mesh and enabled a namespace to be managed by the Service Mesh Control Plane. 
{{% /notice %}}

As you saw in the video, ChatGPT rendered a Deployment manifest that I took and applied to the cluster. 

From there I exposed the deployment. 

```yaml
$ oc expose deployment/nginx --port=80
service/nginx exposed
```

{{% notice warning %}}
We bound a highly privileged Security Context Constraint (SCC) to the default `ServiceAccount` in order to make this particular deployment run in OpenShift. This should **NEVER** be used in production, and any namespace that this has been done to **SHOULD NOT** be used to run Linux pod.
{{% /notice %}}


...And then I went on to create the `Gateway` and `VirtualService` (that it told me to create after some continual nudging on my behalf!)

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

I'm prepared to give ChatGPT a pass mark despite the fact that it did not instruct me to retrieve the `GATEWAY_URL` to access NGINX's welcome page. For [others](https://www.linkedin.com/posts/ioannis-kalyvas-86467034_coding-istio-kubernetes-activity-7019327867058606080-3l4m?utm_source=share&utm_medium=member_desktop) though, the jury is still out regarding its competence around Istio. 

```yaml
$ export GATEWAY_URL=$(oc -n istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}')
```

![](/img/2023-01-chatgpt-kube-deploy/nginx-welcome.png)

## Wrap Up

To keep in theme with this blog post (as well as finding an opportunity to lazily outsource), I thought I'd hand over the reigns of this conclusion to ChatGPT and see what it could come up with. 

And so, I fed it the following prompt:

**Provide me with a conclusion to a blog post that discusses the use of ChatGPT to develop Kubernetes manifests and provide guidance on how to deploy to a Kubernetes cluster.**

_"In conclusion, ChatGPT can be a useful tool for generating Kubernetes manifests and providing guidance on how to deploy to a Kubernetes cluster. It is able to quickly generate manifest files and provide recommendations based on user input, making it a time-saving and convenient tool for those working with Kubernetes. However, it is important to note that ChatGPT is not a replacement for in-depth knowledge of Kubernetes and should be used as a supplement to, rather than a substitute for, manual configuration and understanding of the system. Overall, ChatGPT can be a valuable tool for those looking to streamline their Kubernetes workflow and deploy applications efficiently."_

I like this answer for two reasons. First, it perfectly sums up the function ChatGPT serves for software developers and infrastructure engineers - a 
time-saving tool that makes it easier for folks to get their job done and focus on more important (à la innovative) tasks. Second, it's reaffirming the role of humans to not take its output as gospel and always, **always** thoroughly review and test any manifests or deployment instructions generated by the tool before implementing them in a _production environment_ - in short, our jobs are still safe tomorrow. 

Put simply however, ChatGPT is an amazing playground to tinker in and I hope you dive right in to see what it can do for you!