---
layout: default
---

## Adding CI workflow using Github Actions

Github Actions is one of the best things to come out of Github. If you dont know about it you can read [here](https://docs.github.com/en/actions). Basically it makes your CI/CD extremeely easy to configure and manage. You can set up workflows to tell the github to build, test, package, release or deploy code in any repository in your preferred order. And you can have multiple workflows for a single repository also.

Here is an example how to set it up:

#### Create workflow directory:
Create .github/workflows directory in the root of your repository.

#### Make the yaml file that will include the steps:
Create a yaml file inside the above folder that will define steps you want to run along with thier order and configuration. You can read the docs of what can be included [here](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions).

As an example here, I am making the workflow to package and run unit test cases in the yaml.

```yaml
name: Run unit test cases
on: [push, pull_request]
jobs:
build:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
    - uses: actions/checkout@v2
    - name: set up python
    - uses: actions/setup-python@v1
    - name: install requirements
    run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    - name: run tests
    run: |
        pytest
```

#### Commit and check your repo on github output of running tests (or your workflow)
![automated-workflow](../assets/img/annotated-workflow.png)


#### Other fancy things
* You can set a schedule to run the worflow jobs also rather than triggering on an event like pull request or push to a branch.
* You can choose two types of runners on which the job will bee executed; a github-hosted runner (ubuntu, windows, macOS and a buch of other options are available on github side) and a self-hosted runner like [here](https://docs.github.com/en/actions/hosting-your-own-runners/using-self-hosted-runners-in-a-workflow).
* Different actions can be used by the job or inside the job like in the example above im using checkout@v2 which checks out the repository and is basic. But there are other options available [here](https://github.com/actions). e.g. Docker container actions, javaScript actions, composite run steps actions etc


[back](../)