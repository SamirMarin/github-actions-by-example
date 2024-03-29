# The deploy workflow job
In this section, we address the deployment workflow job, a crucial part of our CI/CD pipeline designed for deploying our service. Triggered upon the successful completion of the build-and-push job, this job takes the reins to roll out the latest version of our service.

## Understanding Deployment in Context
Before we explore the deploy workflow, let's contextualize what deployment entails within the scope of this book. Deployment, while a broad topic with numerous facets, will be simplified here to focus on the essence of what it means to deploy a service.

At its core, a deployment process involves updating a service to a new version, typically by changing the service's running environment to utilize the latest application image. This is orchestrated through configuration files, which can be viewed as a blueprint detailing how the service should be deployed. These files dictate various parameters such as the service image to use, environment variables, secrets, and more.

### Deployment configuration file
For illustrative purposes, we will utilize a Kubernetes deployment configuration file. It's important to note that an understading of Kubernetes is not required for this discussion. Our objective is to illustrate how such a configuration file represents the deployment instructions:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workout-management-service
  labels:
    app: workout-management-service
spec:
    replicas: 3
    selector:
        matchLabels:
        app: workout-management-service
    template:
        metadata:
        labels:
            app: workout-management-service
        spec:
        containers:
        - name: workout-management-service
          image: ghcr.io/SamirMarin/workout-management-service:<tag>
          ports:
          - containerPort: 1323
```

This file outlines the deployment setup for our service, most importantly it defines the <image:tag> to use. The essence of our deploy workflow job, therefore, lies in updating this deployment configuration file to reference the latest Docker image tag. This update process signifies the deployment of our newest service version. By automating the update to point to the image tag generated by our recent build-and-push job, we achieve an effective and streamlined deployment.

We'll house this file within our repository, specifically in a deploy/ directory. For the sake of simplicity in our explanation, we'll bypass the actual deployment execution using this file. Instead, let's conceptualize that merely updating this file to reference the latest image tag equates to deploying our service. Imagine a Continuous Delivery (CD) tool in play, vigilantly tracking changes to this file on our repository's default branch. Upon detecting any modifications, this hypothetical CD tool would automatically roll out our service, thereby enacting the deployment based on our updated configuration.

lets place this file under the deploy/ directory in our repository:

```bash
mkdir -p deploy
touch deploy/deployment.yaml
```
Let commit this file to main branch.

## Creating the deploy workflow job
Let's utilize the same file we used for the test workflow job, but lets rename the file:

```yaml
mv .github/workflows/test-build-and-push.yaml .github/workflows/test-build-deploy.yaml
```

Let also rename the build-and-push job to 'build' to keep naming simple. We will configure the deploy to job to execute only after the build job has successfully completed.

### Purpose of the Deploy Job
The deploy job's primary function is to ensure our deployment configuration file (deploy/deployment.yaml) accurately points to the Docker image tag generated during the build process.

To facilitate this update, we'll leverage the fjogeleit/yaml-update-action community action. This powerful tool not only updates our deployment.yaml with the latest Docker tag but also automates the creation of a pull request against the main branch to implement this change.

```yaml
  name: test build deploy

  on:
    push:
      branches:
        - main

    pull_request:
      branches:
        - main

  jobs:
    test:
      runs-on: ubuntu-latest
      steps:
        - name: checkout
          ...
          
    build:
      runs-on: ubuntu-latest
      steps:
        - name: checkout
          ...
    
    deploy:
      needs: [build]
      runs-on: ubuntu-latest
      steps:
        - name: checkout
          uses: actions/checkout@v4
        - name: Update deployment.yaml
          uses: fjogeleit/yaml-update-action@v0.14.0
          with:
            valueFile: 'deploy/deployment.yaml'
            propertyPath: 'spec.template.spec.containers[0].image'
            value: ghcr.io/${{ github.event.repository.owner.login }}/${{ github.event.repository.name }}:${{ github.sha }}
            commitChange: ${{ github.event_name != 'pull_request' }}
            targetBranch: main
            masterBranchName: main # needed when default branch is not master
            createPR: ${{ github.event_name != 'pull_request' }}
            branch: 'deploy'
            token: ${{ secrets.GITHUB_TOKEN }}
            message: 'Update Image Version to: ${{ github.sha }}'
```

Overview of Each Step in the Build-and-Push Job:
- Checkout Step: Fetch the repository code into the action runner using actions/checkout@v4.
- Update deployment.yaml: Here, the fjogeleit/yaml-update-action is tasked with modifying the deployment.yaml to reference the newest Docker image tag. It then proceeds to create a pull request with this update, promoting seamless integration and deployment of the latest service version.

Before this action can successfully create a PR you will need to allow actions to create PRs in the repository settings. To do this go to setting->Actions->General->Workflow permissions and select 'Allow GitHub Actions to create and approve pull requests'.