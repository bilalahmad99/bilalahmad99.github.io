---
layout: default
---

## Local k8s development setup with Skaffold

When you want to deploy or test your application on kubernetes setup before committing, which is always a good idea if your production environment is running in Kubernetes you would like to test the code in as similar environment to production as possible, here are thing you would require to do:

* Make a change to code
* Build docker/container image
* Push to image repository
* Deploy to Kubernetes

[Skaffold](https://skaffold.dev/) is an open source tool developed by Google that does all of the above and more in an automated easier way for you. Here is an example of it.

#### Pre-requisite
* k8s installed in your local machine; you can use minikube or docker-desktop or microk8s whatever is your preference.
* kubectl config set and updated to point to your local k8s setup
* docker installed

#### Install Skaffold
On macOS you can just do:

```bash
brew install skaffold
```
on other OS you can install using the docs [here](https://skaffold.dev/docs/install/)

#### Add skaffold yaml
On the root directory of your repository, create skaffold.yaml file

```yaml
apiVersion: skaffold/v1
kind: Config
build:
  artifacts:
  - image: example-repo
    context: .
    sync:
      manual:
        - src: "**"
          dest: "/src"
deploy:
  kubectl:
    manifests:
      - k8s-manifests/*.yaml
```

Running  `skaffold init`  at the root of your project directory will walk you through a wizard and create a skaffold.yaml also with build and deploy config and then later you can update the yaml manually to add file sync and other stuff.

#### Add the k8s manifests
Now add the manifests like your deployment and service yamls inside the folder k8s-manifests (mentioned above in the yaml).
Here is the manifests file in my case as an example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: node
spec:
  ports:
  - port: 3000
  type: LoadBalancer
  selector:
    app: node
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node
spec:
  selector:
    matchLabels:
      app: node
  template:
    metadata:
      labels:
        app: node
    spec:
      containers:
      - name: node
        image: node-example
        ports:
        - containerPort: 3000
```

#### Run skaffold dev

`skaffold dev` command will take Dockerfile at the root of your repository to create the image, you can also configure it to take from any other directory or take any other file like cloudbuild etc. [Here](https://skaffold.dev/docs/references/yaml/) is a useful refernce to all thing you can configure inside the skaffold.yaml file.


#### Other cool things

You can do debugging with Skaffold also with
``` bash
skaffold debug
```
Right now languages supported are:
* Go
* NodeJS
* Java and JVM languages
* Python
* .NET Core

You can set healthcheck also which will wait for deployment to stabilize. It can be used in command.
```bash
skaffold run
```

You can do rendering of templates which will perform builds on all artifacts in your project and instead of sending these through the deployment process, print out the final deployment artifacts.
```bash
skaffold render
```


You can do log tailing with
```bash
skaffold run --tail
```

#### Skaffold supports

building artificats with
* docker
* gcloud build
* kaniko
* Bazel

deploying artifacts with
* kubectl
* helm
* kustomize