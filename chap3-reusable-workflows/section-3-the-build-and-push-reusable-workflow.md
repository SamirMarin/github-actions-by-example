# Section 3 - The build and push reusable workflow

Similarly to the reusable test workflow, the build and push workflow will closely resemble the one we created in Chapter 2, Section 4. The main difference is that this new workflow will be reusable, allowing it to be called from other repositories.

## Creating the reusable workflow

We’ll start by referencing the logic used in Chapter 2, Section 4 (The Build and Test Workflow) and refactor it to create a reusable workflow.

### Setting up the reusable workflow files

First, we will rename the `reusable-test.yaml` file located in the `.github/workflows/` directory of the [`github-actions-by-example-reusable-workflows`](https://github.com/SamirMarin/github-actions-by-example-reusable-workflows) repository to `reusable-test-build.yaml`.

### Refactoring the build and push workflow

Next, we’ll copy the build and push workflow job from one of our service repositories and place it below the test job in the newly renamed reusable workflow file. For this example, let’s copy the [build and push job](https://github.com/SamirMarin/user-management-service/blob/8ea4779ec3beb9368f99953aaf3b7fb02c09ef54/.github/workflows/test-build-deploy.yaml#L38-L68) from the `test-build-and-deploy` workflow of the `user-management-service` repository and refactor it to make it reusable.

To make the build and push job reusable, we’ll follow a similar process to what we did with the test job. Since the job is now part of an already reusable workflow file, we just need to add the necessary inputs for the build and push job and use these inputs within the job.

* Inputs required for the build and push job



