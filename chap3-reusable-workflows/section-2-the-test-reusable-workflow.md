# Section 2 - The test reusable workflow

The test reusable workflow will function like the test workflow we wrote in Chapter 2, Section 3, with the difference that this workflow will be reusable and can be called from another repository.

## Creating the reusable workflow

We will reference the logic we used in Chapter2, Section 3 - The test workflow and refactor it to make the workflow reusable.

### Setting up the reusable workflow files

We'll start by creating a reusable-test.yaml file under the .github/workflows/ directory in the [github-actions-by-example-reusable-worflows](https://github.com/SamirMarin/github-actions-by-example-reusable-workflows) repo.

```bash
mkdir -p .github/workflows/
touch .github/workflows/reusable-test.yaml
```

### Refactoring the test workflow

Next, we'll copy the test workflow job from one of our service repositories. In this example, we take the [`test` job](https://github.com/SamirMarin/user-management-service/blob/8ea4779ec3beb9368f99953aaf3b7fb02c09ef54/.github/workflows/test-build-deploy.yaml#L13-L36) from the `user-management-service`'s `test-build-and-deploy` workflow and refactor it to make it reusable.

To refactor the test job into a reusable workflow, we make the following changes:

1. Update the trigger to `workflow_call`.
2. Define the inputs that the calling workflow can pass to the reusable workflow.

The goal is to make the reusable workflow as flexible as possible, though there is a trade-off. Since this reusable workflow may only be used for a specific type of service, we can limit its flexibility accordingly. For instance, we're assuming all our services are written in Go, so by default, we include Go installation steps in the test workflow job.

We'll add the following inputs and provided default values, allowing the caller to override these defaults only if needed:

* docker-compose-command: the Docker Compose command to run
  * defaults to: `docker compose up -d`
* docker-compose-command-opts: the Docker Compose command options
  * defaults to: `-inMemory`
* test-script: the test script to run
  *   defaults to:&#x20;

      ```
      ./scripts/dynamodb/create-table.sh
      go test -v ./...
      ```
* aws-region: the AWS region:
  * defaults to: `us-west-2`
* dynamodb-local-endpoint: the DynamoDB local endpoint:
  * defaults to: [http://localhost:8000](http://localhost:8000)
* test-aws-access-key-id: the AWS access key ID
  * defaults to: `test`
* test-aws-secret-access-key: the AWS secret access key
  * defaults to: `test`

We'll utilize all the inputs defined in the workflow. For example, instead of hardcoding commands like `docker compose up -d` directly in the workflow, we use the input value `${{ inputs.docker-compose-command }}`, which allows the calling workflow to override the default command if needed.

Here is the refactored test job in the reusable workflow:

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

Finally, we commit and push the changes to the reusable workflows repository. In our case, we’ll push the updates to the [`github-actions-by-example-reusable-workflows`](https://github.com/SamirMarin/github-actions-by-example-reusable-workflows) repository.
