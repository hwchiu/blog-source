---
title: Slides - Kubernetes with private docker registry
keywords: 'Linux,Ubuntu,Kubernetes,Docker,Registry'
date: 2018-05-30 16:56:00
tags:
	- Kubernetes
	- Linux
	- Docker
	- Registry
	- Slides
	- CNTUG
description:
---

### Information
- Time: 2018/05/26
- Meetup: [Cloud Native Taiwan User Group #5](https://cntug.kktix.cc/events/sdn-cntug-5)

### Abstraction
This slides introduce a basic concept of how docker pulls images from docker hub and private registry first and then assumes a simple scenario which do the CI/CD for applications in the kubernetes cluster.

In that scenario, you will use a kubernetes pod to build your application into a docker image and push that to the private registry and then pull that image to run as another kubernetes Pod in kubernetes cluster.

The slides shows every problems you will meet, including how do you deploy your private registry, the network connectivity between the k8s Node/k8s Pod and private registry and a trusted SSL certificate.

In the last section, the author provides a simple solution for that scenario and use a simple graph to explain how to works.

<!--more-->

### Slides
<iframe src="//www.slideshare.net/slideshow/embed_code/key/2eeiuiva1AABek" width="595" height="485" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//www.slideshare.net/hongweiqiu/integration-kubernetes-with-docker-private-registry" title="Integration kubernetes with docker private registry" target="_blank">Integration kubernetes with docker private registry</a> </strong> from <strong><a href="//www.slideshare.net/hongweiqiu" target="_blank">宏瑋 邱(Hung-Wei Chiu)</a></strong> </div>
