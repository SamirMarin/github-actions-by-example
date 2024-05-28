# The test reusable workflow
The test reusable workflow will function like the test workflow we wrote in Chapter 2, Section 3, with the difference that this workflow will be reusable and can be called from another repository.

## Creating the reusable workflow
We will reference the logic we used in [the-test-resuable-workflow](https://github.com/SamirMarin/github-actions-by-example/blob/main/chap2-deployment-workflow/s3-the-test-workflow-job.md) and refactor it to make the workflow reusable.

### Setting up the reusable workflow files
We will start by creating a reusable-test.yaml file under the .github/workflows/ directory in the [github-actions-by-example-reusable-worflows](https://github.com/SamirMarin/github-actions-by-example-reusable-worflows) repo.

```
mkdir -p .github/workflows/
touch .github/workflows/test.yaml
```

We will then copy the test workflow job from one of service repositories, lets use the [user-management-service](https://github.com/SamirMarin/user-management-service/blob/main/.github/workflows/test-build-deploy.yaml) repo, and refactor to make reusable. 

Next, we will copy the test workflow job from one of the service repositories. Let's use the [test job](https://github.com/SamirMarin/user-management-service/blob/8ea4779ec3beb9368f99953aaf3b7fb02c09ef54/.github/workflows/test-build-deploy.yaml#L13-L36) from user-management-service test-build-and-deploy workflow and refactor it to make it reusable.

```yaml
```