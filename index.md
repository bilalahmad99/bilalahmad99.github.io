---
layout: default
---
<div class='index-head'>
<p> Hey there! <span class="emoji wave" aria-label="hand wave"></span>
I use this space to write down my thoughts and experiments (mostly tech related). In the archive section, I have tried to label everything with sub-topics or subjects that are discussed in it.
<br>
<b>Mandatory Disclosure:</b> Opinions are my own and not the views of my employer or previous employers.
</p>
</div>
---

&nbsp;

### 5 signs your workplace is toxic 

#### High levels of stress and pressure:
A toxic work environment can create a culture of fear and anxiety, leading to high levels of stress and pressure for employees.

[Continue Reading](./pages/toxic-workplace.html)

&nbsp;

### Lets talk about some drawbacks for Microservices and why it is not always the best architecture pattern for you

Microservices are a popular architectural pattern that allows organizations to build and deploy applications as a collection of small, independently deployable services. While there are many benefits to using microservices, there are also some drawbacks to consider. Here are a few:

[Continue Reading](./pages/microservices-drawback.html)

&nbsp;

### 5 common mistakes when setting up Istio

Istio is an open-source service mesh that can be used to manage and control the traffic within a microservices-based application. Setting up Istio can be complex, and it's important to avoid common mistakes in order to ensure that the service mesh is functioning correctly. Here are five common mistakes that organizations make when setting up Istio:

[Continue Reading](./pages/istio-mistakes.html)

&nbsp;

### Cloud Migrations

There are several types of cloud migrations that organizations can undertake, depending on their needs and resources. Some common types of cloud migrations include:

[Continue Reading](./pages/migration.html)

&nbsp;

### Continuous Deployment - Best Practices 

* Implement version control: Use a version control system such as Git to manage and track changes to your code.

* Automate the build process: Automate the process of building, testing, and deploying your code. This can be done using tools such as Jenkins or Travis CI.

[Continue Reading](./pages/cd.html)


&nbsp;

### Core Pillars to consider when designing the software architecture

#### Functionality:
The software architecture should provide the necessary functionality to meet the requirements of the system and its users.

#### Scalability:
The architecture should be designed to handle expected and unexpected increases in usage and data volume, making it easy to add more resources and capacity as needed.
[Continue Reading](./pages/pillars.html)

&nbsp;

### Service Level Agreements - Best Practices 

First lets talk about the what are SLA, SLO and SLIs.

[Continue Reading](./pages/sla.html)

&nbsp;


### Software Deployment

Software deployment is a combination of all the steps, processes, and activities that are required to make a software system or update available to its intended users.

[Continue Reading](./pages/deployment_types.html)

&nbsp;

### Adding Self-Hosted Github Runners on EKS with Ansible


Assuming the EKS cluster already exists.

Use the following tasks.yml for ansible to 

* Install cert manager helm chart
* Install actions runner system helm chart
* Create a dedicated namespace for this stuff
* Install helm chart for self hosted runner

[Continue Reading](./pages/github_runners.html)

&nbsp;

### What constitutes a Continuous Integration process 
* Static code analysis covering code quality and vulnerability checking,

* Software composition analysis to check code dependencies

* Unit testing to prove desired functionality at a low level

* Container inspection

[Continue Reading](./pages/testing_types.html)

&nbsp;

## Running Terraform through Ansible

From my experience so far, I have seen a lot of organizations using multiple automation tools in their DevOps toolchain. There is no single tool that can do everything required.

Ansible is quite useful for many organization for creating automated playbooks for different tasks e.g. configuration of Nginx, or Management for SQL users etc. But its not very widely used for provisioning for Infrastructure on Cloud providers. For this the most common used tool is Terraform.

[Continue Reading](./pages/running_terraform_through_ansible.html)

&nbsp;


### No wonder Pulumi is getting a lot of traction, Its Great!

For so long if you wanted a good Infrastructure-as-a-code tool that works for multiple cloud providers, you had one option: 'The Hashicorp Terraform'. It is based on HCL and it did a pretty good job in all fairness and still does. But it lacks a few things which Pulumi has solved. E.g. HCL is difficult and awkward language for infrastructure-as-a-code, It's not very developer friendly and it does not have a fancy UI. With Pulumi, you get the fancy UI and you can code in Python,Java or any developer friendly language and it also takes care of managing your state of infrastructure, you dont have to manage it yourself (although sometimes its a good idea to manage your state yourself and Pulumi allows it too).

[Continue Reading](./pages/pulumi.html)

&nbsp;

### Getting started with Cloud Run

Google cloud run is a fully managed compute platform for running containerized applications on the cloud.
The best serverless solution for containerized applications at this moment is Cloud Run in my opinion. Yes, its better then AWS Lambda (as of now) particularly because of its easiness, excellent containerized support and seamless integration with Kubernetes (GKE with Knative) and cheaper also

[Continue Reading](./pages/cloud_run.html)

&nbsp;

### Running gRPC-web on Istio

gRPC web is the JavaScript implementation of [gRPC](https://grpc.io) for browser clients.

There is no current browser that supports HTTP/2 gRPC spec currently essentially because raw HTTP/2 frames are inaccessible in browsers. You can read further on this [here](https://grpc.io/blog/state-of-grpc-web/#the-grpc-web-spec). Hence, gRPC-web clients connect to gRPC services via a special proxy; by default, gRPC-web uses [Envoy](https://www.envoyproxy.io). The purpose of Envoy is to translates the HTTP/1.1 calls produced by the client into HTTP/2 calls that can be handled by those services

[Continue Reading](./pages/grpc-web.html)

&nbsp;

### Gcloud SQL disaster recovery strategy

Firstly we don't have any on-premise database instances so it makes our life much easier. Few things to note before designing your disaster recovery approach is RTO (Recovery time objective) and RPO (Recovery point objective) metrics. Both of these can vary from service to service as well as business to business. RTO is the maximum acceptable length of time that your application can be offline, and RPO is s the maximum acceptable length of time during which data might be lost from your application due to a major incident. The values of these you set yourself as part of your SLAs (Service level agreements)

[Continue Reading](./pages/gcloud_disaster_recovery_data.html)

&nbsp;

### Everyone needs to vent sometimes

I recently noticed in my organization increase in a bit of annoyance or bitterness. For example some passive-aggressive Q&As during all-hands or other meetings, a feeling of distress and uneasiness during regular standups or 1x1s. I noticed a similar behaviour in myself also. So one morning after an uneasy standup meeting I tried to figure out the root cause of this (as an SRE will do)

[Continue Reading](./pages/everyone_needs_to_vent.html)

&nbsp;

### Adding CI workflow using Github Actions

Github Actions is one of the best things to come out of Github. If you don't know about it you can read [here](https://docs.github.com/en/actions). Basically it makes your CI/CD extremely easy to configure and manage. You can set up workflows to tell the github to build, test, package, release or deploy code in any repository in your preferred order. And you can have multiple workflows for a single repository also.

Here is an example how to set it up

[Continue Reading](./pages/github_actions.html)


&nbsp;

### Gcloud Secret Manager for accessing environment variables in application

So finally Gcloud launched its own secret manager as a service. I got to know from Seth Vargo's [tweet](https://twitter.com/sethvargo/status/1220035296018018310), who btw, if you don't follow, has given some great talks around DevOps and generally on technology.
Before this, gcloud did not have anything around this to compete with [AWS Secret manager](https://aws.amazon.com/secrets-manager/) and [Hashicorp Vault](https://www.vaultproject.io/). Ofcourse you could use both of these and integrate with your gcloud resources. But after this, the integration is seamless

[Continue Reading](./pages/gcloud_secrets.html)

&nbsp;

### Local k8s setup with Skaffold for development environment

When you want to deploy or test your application on Kubernetes setup before committing, which is always a good idea if your production environment is running in Kubernetes you would like to test the code in as similar environment to production as possible, here are thing you would require to do:

* Make a change to code
* Build docker/container image
* Push to image repository
* Deploy to Kubernetes

[Skaffold](https://skaffold.dev/) is an open source tool developed by Google that does all of the above and more in an automated easier way for you. Here is an example of it

[Continue Reading](./pages/skaffold.html)

&nbsp;

### DevSecOps: Security breach Incident protocol

Security breaches in the tech world are increasing at an alarming rate. If it hasn't happened for you yet, be prepared for it because as the saying goes *Hope for the best and prepare for the worst*... [Continue Reading](./pages/incident_protocol.html)

&nbsp;

### AWS ELB IP dynamically resolve issue on Nginx

We were using Nginx (free version) as a reverse proxy in front of AWS Elastic Load Balancer to proxy all requests to the servers behind our load balancer. Now with ELB, you are supposed to use the CNAME record of it since the IPs of ELBs can and do keep changing. Usually what happens on AWS side is if your ELB starts to get traffic increase, AWS changes the machine to add a heavy beefy machine behind the scene which can cause IPs to change and their can be other reasons also like maintenance etc.
Anyways Nginx resolves the address defined in the proxy_pass directive just once when it starts up or reloaded. For the rest of the times it keeps using the local cache. This caused the problem for us as after some time or days the IP changed and we started getting `Error: upstream timed out`. Here is how we solved it

[Continue Reading](./pages/nginx_aws_elb_ip.html)

&nbsp;

### Setting up Hashicorp Vault to manage secrets

Vault is an extensive tool with both opensource and paid version to allow you to secure, store and tightly control access to tokens, passwords, certificates, encryption keys for protecting secrets and other sensitive data using a UI, CLI, or HTTP API... [Continue Reading](./pages/vault.html)

&nbsp;

### Everyone will tell you how to give feedback, nobody tells you how to receive it

The concept of 1:1 meetings are very common in tech organizations. As we grow in our career it gets increasingly common to give and receive feedback in professional environment to colleagues.

You will see quite a few articles and guidelines around for managers for how to effectively give feedback to your sub-ordinates so that it has a greater impact. Even organizations sometimes do a good job in training the managers around this area to improve giving feedback to the team members they are managing. However nobody tells you how to receive the feedback, atleast nobody told me about it

[Continue Reading](./pages/feedback.html)
