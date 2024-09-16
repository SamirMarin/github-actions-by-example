# Section 4 - The deploy reusable workflow

Similarly to the reusable  test and the build and push job , the reusable deploy job will closely resemble the one we created in Chapter 2, Section 5. The main difference is that this new workflow job will be part of a reusable workflow, allowing it to be called from other repositories.

## Creating the reusable workflow job

We’ll start by referencing the logic used in Chapter 2, Section 5 (The deploy job) and refactor it to be part of a reusable workflow.

### Setting up the reusable workflow files

First, we will rename the `reusable-test-build.yaml` file located in the `.github/workflows/` directory of the [`github-actions-by-example-reusable-workflows`](https://github.com/SamirMarin/github-actions-by-example-reusable-workflows) repository to `reusable-test-build-deploy.yaml`

### Refactoring the build and push workflow

Next, we’ll refactor the deploy workflow job by copying it from one of our service repositories and placing it below the `build and push` job in our newly renamed reusable workflow file. For this example, we'll use the [deploy job](https://github.com/SamirMarin/user-management-service/blob/8ea4779ec3beb9368f99953aaf3b7fb02c09ef54/.github/workflows/test-build-deploy.yaml#L70-L88) from the `test-build-and-deploy` workflow in the `user-management-service` repository and modify it to be reusable.

To make the `deploy` job reusable, we’ll follow a similar process as with the `build and push` job. Since it’s now part of a reusable workflow file, we only need to introduce the necessary inputs and utilize them within the job.

We’ll add the following inputs, providing default values for each:

* deploy-commit: wether to commit changes for deploy&#x20;
  * defaults to:  `${{ github.event_name != 'pull_request' }}`
* deploy-create-pr: wethe to create the pr for deploy
  * defauts to: `${{ github.event_name != 'pull_request' }}`

We’ll now place the `deploy` job into the reusable workflow, integrating the inputs we just defined:

```yaml
name: test-build-deploy

on:
  workflow_call:
    inputs:
...
      deploy-commit:
        required: false
        type: boolean
        description: "Commit changes to deploy"
        default: ${{ github.event_name != 'pull_request' }}
      deploy-create-pr:
        required: false
        type: boolean
        description: "Create PR for deploy"
        default: ${{ github.event_name != 'pull_request' }}

jobs:
  test:
  ...
  build:
  ...
  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: checkout
        uses: actions/checkout@v4
      - name: Update deployment.yaml
        uses: fjogeleit/yaml-update-action@v1
        with:
          valueFile: 'deploy/deployment.yaml'
          propertyPath: 'spec.template.spec.containers[0].image'
          value: ghcr.io/${{ inputs.image-owner }}/${{ inputs.image-name }}:${{ github.sha }}
          commitChange: ${{ inputs.deploy-commit }}
          targetBranch: main
          masterBranchName: main
          createPR: ${{ inputs.deploy-create-pr }}
          branch: 'reusable/deploy'
          token: ${{ secrets.GITHUB_TOKEN }}
          message: 'Update Image Version to: ${{ github.sha }}'

  
```

