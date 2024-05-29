# The test reusable workflow
The test reusable workflow will function like the test workflow we wrote in Chapter 2, Section 3, with the difference that this workflow will be reusable and can be called from another repository.

## Creating the reusable workflow
We will reference the logic we used in [the-test-reusable-workflow](https://github.com/SamirMarin/github-actions-by-example/blob/main/chap2-deployment-workflow/s3-the-test-workflow-job.md) and refactor it to make the workflow reusable.

### Setting up the reusable workflow files
We will start by creating a reusable-test.yaml file under the .github/workflows/ directory in the [github-actions-by-example-reusable-worflows](https://github.com/SamirMarin/github-actions-by-example-reusable-worflows) repo.

```
mkdir -p .github/workflows/
touch .github/workflows/test.yaml
```

Next, we will copy the test workflow job from one of the service repositories. Let's use the [test job](https://github.com/SamirMarin/user-management-service/blob/8ea4779ec3beb9368f99953aaf3b7fb02c09ef54/.github/workflows/test-build-deploy.yaml#L13-L36) from user-management-service test-build-and-deploy workflow and refactor it to make it reusable.

```yaml
name: Test

on:
  workflow_call:
    inputs:
      docker-compose-command:
        required: false
        type: string
        description: "Docker compose command"
        default: |
          docker compose up -d
      docker-compose-command-opts:
        required: false
        type: string
        description: "Docker compose command options"
        default: "-inMemory"
      test-script:
        required: false
        type: string
        description: "Test script"
        default: |
          ./scripts/dynamodb/create-table.sh
          go test -v ./...
      aws-region:
        required: false
        type: string
        description: "AWS region"
        default: "us-west-2"
      dynamodb-local-endpoint:
        required: false
        type: string
        description: "DynamoDB local endpoint"
        default: "http://localhost:8000"
      test-aws-access-key-id:
        required: false
        type: string
        description: "AWS access key ID (for local testing, should not be a real access key ID)"
        default: "test"
      test-aws-secret-access-key:
        required: false
        type: string
        description: "AWS secret access key (for local testing, should not be a real secret access key)"
        default: "test"
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - name: Docker Compose
        run: |
          ${{ inputs.docker-compose-command }}
        env:
          DOCKER_COMPOSE_COMMAND_OPTS: ${{ inputs.docker-compose-command-opts }}
      - name: Test
        run: |
          ${{ inputs.test-script }}
        env:
          AWS_ACCESS_KEY_ID: ${{ inputs.test-aws-access-key-id }}
          AWS_SECRET_ACCESS_KEY: ${{ inputs.test-aws-secret-access-key }}
          AWS_DEFAULT_REGION: ${{ inputs.aws-region }}
          DYNAMODB_LOCAL_ENDPOINT: ${{ inputs.dynamodb-local-endpoint }}
```

Commit and push the changes to the reusable workflows repository. In our case, we will push the changes to the [github-actions-by-example-reusable-workflows](https://github.com/SamirMarin/github-actions-by-example-reusable-workflows) repository.

### Refactoring the test workflow job explanation
To refactor the test workflow job to be a reusable workflow, we need to make the following changes:
- Update the trigger to `workflow_call`.
- Define the inputs that the calling workflow can pass to the reusable workflow.
- - The idea is to make the reusable workflow as flexible as possible, but there is a trade-off. Given that the reusable workflow may only be used for a given type of service, its flexibility can be tied to this. For example, we are assuming all our services are in Go, so we default to installing Go in the workflow.
- In our case, we added the following inputs and provided default values for all these inputs, giving the caller the flexibility to override the defaults only if needed:
- - docker-compose-command: the Docker Compose command to run
- - docker-compose-command-opts: the Docker Compose command options
- - test-script: the test script to run
- - aws-region: the AWS region
- - dynamodb-local-endpoint: the DynamoDB local endpoint
- - test-aws-access-key-id: the AWS access key ID
- - test-aws-secret-access-key: the AWS secret access key

### Using the reusable workflow
To use the reusable workflow, we need to call it from another workflow. Let's update our services to use the reusable workflow:

Example: Update the [user-management-service](https://github.com/SamirMarin/user-management-service/blob/8ea4779ec3beb9368f99953aaf3b7fb02c09ef54/.github/workflows/test-build-deploy.yaml#L13-L36) to use the test reusable workflow:

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
    uses: SamirMarin/github-by-example-reusable-workflows/.github/workflows/reusable-test.yaml@main
    secrets: inherit
...
```

In our case, we will use all the default inputs, so we won't pass any inputs to the reusable workflow. However, we could pass inputs if we wanted to override the defaults by specifying the inputs in the `with` section.

```yaml
jobs:
  test:
    uses: SamirMarin/github-by-example-reusable-workflows/.github/workflows/reusable-test.yaml@main
    with:
      test-aws-access-key-id: local
      test-aws-secret-access-key: local
    secrets: inherit

```