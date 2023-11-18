# Workflow
In this chapter, we will dive deep in Github action workflows by running through a multi job deployment workflow

The workflow will run tests, build a docker image, push the image to the Github registry, and trigger a deployment

## Services we will deploy
We will deploy two simple golang http services. The workout-management-service and the user-management-service. The implementation details of the services is not important from a workflow perspective, so you may skip s2-the-services if you are not interested in the implementation details of the service.

## The test workflow job
This workflow job will be responsible for running the unit tests that ensure the service is working as expected.

## The build-and-push workflow job
This workflow job will be responsible for building the docker image and pushing it to the Github registry.

## The deploy workflow job
This workflow job will be responsible for triggering the deployment of the service.
