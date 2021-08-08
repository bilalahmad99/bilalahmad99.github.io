---
layout: default
---

## &#128075;

Hey there!

Welcome to my blog. Writing usually helps me structure my thoughts and reflect on them, hence this.

Some topics might interest you also. Do check out the archive section where i have tried to label articles with the related subjects that are discussed in it.

##### Mandatory Disclosure:
Opinions are my own and not the views of my employer or previous employers.

&nbsp;

&nbsp;

## Latest thoughts

### Everyone needs to vent sometimes

I recently noticed in my organisation increase in a bit of annoyance or bitterness. For example some passive-agressive Q&As during all-hands or other meetings, a feeling of distress and uneasiness during regular standups or 1x1s. I noticed a similar behaviour in myself also. So one morning after an uneasy standup meeting I thought to myself, lets figure out the root cause of this persistent feeling all around... [Continue Reading](./pages/everyone_needs_to_vent.html)

&nbsp;

### Adding CI workflow using Github Actions

Github Actions is one of the best things to come out of Github. If you dont know about it you can read [here](https://docs.github.com/en/actions). Basically it makes your CI/CD extremeely easy to configure and manage. You can set up workflows to tell the github to build, test, package, release or deploy code in any repository in your preferred order. And you can have multiple workflows for a single repository also.

Here is an example how to set it up... [Continue Reading](./pages/github_actions.html)


&nbsp;

### Gcloud Secret Manager for accessing environment variables in application

So finally Gcloud launched its own secret manager as a service. I got to know from Seth Vargo's [tweet](https://twitter.com/sethvargo/status/1220035296018018310), who btw, if you don't follow, has given some great talks around DevOps and generally on technology.
Before this, gcloud did not have anything around this to compete with [AWS Secret manager](https://aws.amazon.com/secrets-manager/) and [Hashicorp Vault](https://www.vaultproject.io/). Ofcourse you could use both of these and integrate with your gcloud resources. But after this, the integration is seamless... [Continue Reading](./pages/gcloud_secrets.html)

&nbsp;

### Local k8s setup with Skaffold for development environment

When you want to deploy or test your application on kubernetes setup before committing, which is always a good idea if your production environment is running in Kubernetes you would like to test the code in as similar environment to production as possible, here are thing you would require to do:

* Make a change to code
* Build docker/container image
* Push to image repository
* Deploy to Kubernetes

[Skaffold](https://skaffold.dev/) is an open source tool developed by Google that does all of the above and more in an automated easier way for you. Here is an example of it... [Continue Reading](./pages/skaffold.html)

&nbsp;

### DevSecOps: Security breach Incident protocol

Security breaches in the tech world are increasing at an alarming rate. If it hasn't happened for you yet, be prepared for it because as the saying goes *Hope for the best and prepare for the worst*... [Continue Reading](./pages/incident_protocol.html)

&nbsp;

### AWS ELB IP dynamically resolve issue on Nginx

We were using Nginx (free version) as a reverse proxy in front of AWS Elastic Load Balancer to proxy all requests to the servers behind our load balancer. Now with ELB, you are supposed to use the CNAME record of it since the IPs of ELBs can and do keep changing. Usually what happens on AWS side is if your ELB starts to get traffic increase, AWS changes the machine to add a heavy beefy machine behind the scene which can cause IPs to change and their can be other reasons also like maintenance etc.
Anyways Nginx resolves the address defined in the proxy_pass directive just once when it starts up or reloaded. For the rest of the times it keeps using the local cache. This caused the problem for us as after some time or days the IP changed and we started getting `Error: upstream timed out`. Here is how we solved it ... [Continue Reading](./pages/nginx_aws_elb_ip.html)

&nbsp;

### Setting up Hashicorp Vault to store and manage secrets

Vault is an extensive tool with both opensource and paid version to allow you to secure, store and tightly control access to tokens, passwords, certificates, encryption keys for protecting secrets and other sensitive data using a UI, CLI, or HTTP API... [Continue Reading](./pages/vault.html)

&nbsp;

### Everyone will tell you how to give feedback, nobody tells you how to receive it

The concept of 1:1 meetings are very common these days in tech organizations. As we grow in our career it gets increasingly common to give and receive feedback in professional environment to colleagues.

You will see quite a few articles and guidelines around for managers for how to effectively give feedback to your sub-ordinates so that it has a greater impact. Even organizations sometimes do a good job in training the managers around this area to improve giving feedback to the team members they are managing. However nobody tells you how to receive the feedback, atleast nobody told me about it... [Continue Reading](./pages/feedback.html)
