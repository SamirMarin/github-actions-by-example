# The test workflow job
The test workflow job is an essential component of our CI/CD pipeline, designed to preserve the integrity and reliability of our services by automatically executing a suite of unit and integration tests. These tests are activated upon every Pull Request (PR) creation or merge into the main branch.

## Creating the workflow job
Given the structural resemblance between our services, namely the workout-management-service and the user-management-service, the test workflow job for each will be quite similar. Therefore, we'll focus on the implementation details for one service, understanding that the process for the other would be almost identical.

The objective of the test workflow job is to automatically execute `go test -v ./...` every time a PR is created or merged into the service repositories.

### Setting up the Workflow File
We start by creating a test.yaml workflow file under the .github/workflows/ directory in the workout-management-service repository:

```bash
mkdir -p .github/workflows/
touch .github/workflows/test.yaml
```

We'll then define a basic skeleton for our workflow, initially setting it to trigger on PRs and pushes to the main branch, and performing a simple echo command:

```yaml
name: testing

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
      - name: test
        run: echo "testing"
```
Push this initial setup and create a PR against the main branch to validate the workflow syntax. The actions console should display "testing" for the test job.

### Implementing the Test Workflow
To transition from a skeleton to a functioning test workflow, we need to execute a series of setup steps before our action job can successfully run the tests:

Checkout Step: Fetch the repository code into the action runner using the actions/checkout@v4.
Setup-Go: Install Go in the action runner using the actions/setup-go@v5.
Docker Compose: Launch the local DynamoDB instance needed for the tests using docker compose up.
Create Tables: Ensure the required tables are created in the local DynamoDB instance by executing the create-table.sh script from scripts/dynamodb.

- Checkout Step: Fetch the repository code into the action runner using [actions/checkout@v4](https://github.com/actions/checkout)
- Setup-Go: Install Go in the action runner using the [actions/setup-go@v5](https://github.com/actions/setup-go)
- Docker Compose: Launch the local DynamoDB instance needed for the tests using `docker compose up`
- Create Tables: Ensure the required tables are created in the local DynamoDB instance by executing [scripts/dynamodb/create-table.sh](https://github.com/SamirMarin/workout-management-service/blob/main/scripts/dynamodb/create-table.sh)
In order to ensure we don't run the create command prior to the DB being ready we will update the create-table.sh script to include a wait for DynamoDB Local to become ready step.
Add the following to the create-table.sh script above the create table command:
```bash
#!/bin/bash
echo "Waiting for DynamoDB Local to be ready..."
# Wait for DynamoDB Local to become ready
until aws dynamodb list-tables --endpoint-url http://localhost:8000 --region us-west-2 > /dev/null 2>&1; do
    echo "DynamoDB Local is not ready yet..."
    sleep 5
done
echo "DynamoDB Local is ready."
```

Once these steps are complete, we can run `go test -v ./...` to execute our tests.

Putting it all together into the test workflow definition:

```yaml
name: test

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
        uses: actions/checkout@v4
      - name: set-up go
        uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - name: docker compose
        run: |
          docker compose up -d
        env:
          DOCKER_COMPOSE_COMMAND_OPTS: "-inMemory"
      - name: test
        run: |
          ./scripts/dynamodb/create-table.sh
          go test -v ./...
        env:
          AWS_ACCESS_KEY_ID: test
          AWS_SECRET_ACCESS_KEY: test
          AWS_DEFAULT_REGION: 'us-west-2'
          DYNAMODB_LOCAL_ENDPOINT: "http://localhost:8000"
```

Commit and push this to your branch, and then validate its success through the GitHub Actions console, ensuring the go test command executes as expected.

### Workflow Steps Explained
- Checkout Step: Retrieves the latest code from the repository into our GitHub runner.
- Setup-Go: Ensures that Go is installed in the GitHub runner, using the version specified in go.mod.
- Docker Compose: Launches a local DynamoDB instance required for our tests. It's set to run in memory mode during the tests to ensure a clean, isolated environment and to avoid file system dependencies in the action runner.
- Test: This step ensures that the required tables are set up in the local DynamoDB instance by running the create-table.sh script. It then runs our unit tests using `go test -v ./....`