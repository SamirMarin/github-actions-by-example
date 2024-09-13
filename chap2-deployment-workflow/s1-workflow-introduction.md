# Section 1 - Workflow
In this chapter, we will delve deeply into GitHub Action workflows by examining a multi-job deployment workflow. This workflow encompasses several key processes: running tests to validate the service, building a Docker image, uploading this image to the GitHub registry, and ultimately initiating the deployment process.

## Services we will deploy
We will deploy two straightforward Golang HTTP services: the workout-management-service and the user-management-service. The implementation details of these services are not critical from a workflow standpoint, so feel free to skip Section 2 ('s2-the-services') if you are not interested in the implementation specifics.

## The test workflow job
This job within the workflow is tasked with executing the unit tests to verify that the service is functioning as expected.

## The build-and-push workflow job
This job focuses on constructing the Docker image and pushing it to the GitHub registry.

## The deploy workflow job
The responsibility of this job is to trigger the deployment of the service.