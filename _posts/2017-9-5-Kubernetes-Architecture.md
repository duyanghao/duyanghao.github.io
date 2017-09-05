---
layout: post
title: Introduction to Kubernetes Architecture
date: 2017-9-5 19:10:31
category: 技术
tags: Kubernetes
excerpt: Introduction to Kubernetes Architecture……
---

## Kubernetes

Containerisation has brought a lot of flexibility for developers in terms of managing the deployment of the applications. However, the more granular the application is, the more components it consists of and hence requires some sort of management for those.

One still needs to take care of scheduling the deployment of a certain number of containers to a specific node, managing networking between the containers, following the resource allocation, moving them around as they grow and much more.

Nearly all applications nowadays need to have answers for things like

* Replication of components
* Auto-scaling
* Load balancing
* Rolling updates
* Logging across components
* Monitoring and health checking
* Service discovery
* Authentication

Google has given a combined solution for that which is Kubernetes, or how it’s shortly called – K8s.

In this article, we will look into the moving parts of Kubernetes – what are the key elements, what are they responsible for and what is the typical usage of them. We will then have them all installed using the docker container provided as a playground by K8s team, and review the components deployed.

## Glossary

Before we dive into setting up the components, you should get comfortable with some Kubernetes glossary.

### Pod

Kubernetes targets the management of elastic applications that consist of multiple microservices communicating with each other. Often those microservices are tightly coupled forming a group of containers that would typically, in a non-containerized setup run together on one server. This group, the smallest unit that can be scheduled to be deployed through K8s is called a pod.

This group of containers would share storage, Linux namespaces, cgroups, IP addresses. These are co-located, hence share resources and are always scheduled together.

Pods are not intended to live long. They are created, destroyed and re-created on demand, based on the state of the server and the service itself.

### Service

As pods have a short lifetime, there is not guarantee about the IP address they are served on. This could make the communication of microservices hard. Imagine a typical Frontend communication with Backend services.

Hence K8s has introduced the concept of a service, which is an abstraction on top of a number of pods, typically requiring to run a proxy on top, for other services to communicate with it via a Virtual IP address. 
This is where you can configure load balancing for your numerous pods and expose them via a service.

## Kubernetes components

A K8s setup consists of several parts, some of them optional, some mandatory for the whole system to function.

This is a high-level diagram of the architecture

![](/public/img/k8s/k8s_architecture.png)

Let’s have a look into each of the component’s responsibilities.

### Master node

The master node is responsible for the management of Kubernetes cluster. This is the entry point of all administrative tasks. The master node is the one taking care of orchestrating the worker nodes, where the actual services are running.

Let's dive into each of the components of the master node.

#### <font color="#8B0000">API server</font>

The API server is the entry points for all the REST commands used to control the cluster. It processes the REST requests, validates them, and executes the bound business logic. The result state has to be persisted somewhere, and that brings us to the next component of the master node.

#### <font color="#8B0000">etcd storage</font>


