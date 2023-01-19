---
layout: default
---

## Adding Self-Hosted Github Runners on EKS with Ansible


Assuming the EKS cluster already exists.

Use the following tasks.yml for ansible to 

* Install cert manager helm chart
* Install actions runner system helm chart
* Create a dedicated namespace for this stuff
* Install helm chart for self hosted runner

```yaml
---
- name: Add Jetstack Helm Repository
  kubernetes.core.helm_repository:
    name: jetstack
    repo_url: "https://charts.jetstack.io"

- name: Install Helm chart for Cert Manager
  kubernetes.core.helm:
    name: cert-manager
    chart_ref: jetstack/cert-manager
    chart_version: "v1.11.0"
    create_namespace: true
    release_namespace: cert-manager
    release_values:
      installCRDs: true
    wait: true

- name: Add actions-runner-controller Helm Repository
  kubernetes.core.helm_repository:
    name: actions-runner-controller
    repo_url: "https://actions-runner-controller.github.io/actions-runner-controller"

- name: Install Helm chart for actions-runner-controller
  kubernetes.core.helm:
    name: actions-runner-controller
    chart_ref: actions-runner-controller/actions-runner-controller
    create_namespace: true
    release_namespace: actions-runner-system
    release_values:
      authSecret:
        create: true
        github_token: "github_token"
    wait: true

- name: Create a k8s namespace
  kubernetes.core.k8s:
    name: "self-hosted-runners"
    api_version: v1
    kind: Namespace
    state: present

- name: Install Helm chart for self-hosted-runner
  community.kubernetes.helm:
    chart_ref: "templates/helm/runner"
    release_name: "self-hosted-runner"
    release_namespace: "self-hosted-runners"
    release_values:
      application:
        minReplicas: "3"
        maxReplicas: "5"
        name: "custom-runner"
        repo: "github_runners_repository_name"
    wait: true

```

And the custom template for helm chart is as follows:

templates/helm/runner/Chart.yml
```yaml
apiVersion: v2
name: self-hosted-runners
description: A Helm chart for Github Self Hosted Runners

# A chart can be either an 'application' or a 'library' chart.
#
# Application charts are a collection of templates that can be packaged into versioned archives
# to be deployed.
#
# Library charts provide useful utilities or functions for the chart developer. They're included as
# a dependency of application charts to inject those utilities and functions into the rendering
# pipeline. Library charts do not define any templates and therefore cannot be deployed.
type: application

# This is the chart version. This version number should be incremented each time you make changes
# to the chart and its templates, including the app version.
# Versions are expected to follow Semantic Versioning (https://semver.org/)
version: 1.0.0

# This is the version number of the application being deployed. This version number should be
# incremented each time you make changes to the application. Versions are not expected to
# follow Semantic Versioning. They should reflect the version the application is using.
# It is recommended to use it with quotes.
appVersion: "latest"
```

templates/helm/runner/templates/deployment.yml
```yaml
{% raw %}
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: {{ .Values.application.name }}
spec:
  template:
    spec:
      repository: {{ .Values.application.repo }}
      labels:
        - {{ .Values.application.name }}
{% endraw %}
```

templates/helm/runner/templates/hpa.yml
```yaml
{% raw %}
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: runner-deployment-autoscaler
spec:
  scaleTargetRef:
    name: runner-deployment
  minReplicas: {{ .Values.application.minReplicas }}
  maxReplicas: {{ .Values.application.maxReplicas }}
  metrics:
  - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
    repositoryNames:
    - {{ .Values.application.repo }}
{% endraw %}
```


And thats it!
Now you can use this runner in your workflow yaml file using the `runs-on`

e.g
```yaml
jobs:
  build:
    # The type of runner that the job will run on
    runs-on: self-hosted
```

[back](../)
