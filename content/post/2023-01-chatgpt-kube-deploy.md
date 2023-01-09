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

## Kubernetes Default Scheduling

For two years running, the year's most talked about tech story in the midst of milestones like Black Friday and the Christmas winddown that herald the winding down of the yeart. In 2021, it was lo4, Log4j is a Java-based logging utility,  that usually spell trend (log4j in 2021) In the midst of the frantic that where events like Black Friday and a general wind down toward Christmas, one tech trend still managed to cathc the eyes of most and that was CHatGPT. Many are calling it the most promtient innovation in  social engineering since Google's search engines. 

Intorcued by OpenAI on NOvember 30 last year, there's no shortage of beliweildermeant in the tool's abaility to provide human-quality answers to quite literally antyhing that's thrown at it. 

CHatGPT is a large language model based on GPT-3.5, the third generation Generative Pre-trained Transformer, that aoccrding to [Standford UNiversity](https://hai.stanford.edu/news/how-large-language-models-will-transform-science-society-and-ai) "has 175 billion parameters and was trained on 570 gigabytes of text. For comparison, its predecessor, GPT-2, was over 100 times smaller at 1.5 billion parameters."


## Wrap Up

This was just a fun dabble demonstrating mTLS communication from outside Istio. Certainly there's benefit in drawing from _some_ of the security architecture in here that could be implemented in a scale-out enterprise environment - it's a balance of _risks_ and _needs_; and **where** and **how** we terminate SSL is certainly one of those considerations. 

It should be noted that as of OpenShift 4.9, mTLS authentication can be enabled in the Ingress Controller, so there are other, arguably simpler ways if we want _just_ want to cherry-pick certain security features that were only previously on Istio's bumper sticker, at least in the OpenShift space :smile:

I hope you got some value from the above walkthrough and most importantly, hopefully forwards any discussions you might be having in your team on how you treat your Kubernetes/Service Mesh SSL. 