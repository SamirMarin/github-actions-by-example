# The test workflow
The test workflow is designed to maintain the integrity and reliability of our service by automatically running a suite of unit/integration tests. These tests are triggered whenever a Pull Request (PR) is created or merged. This ensures that any new code contributions or changes to the existing codebase do not disrupt the functionality of the service.

## Creating the workflow
Each of our service the workout-management-service and the user-management-service will run a test workflow, given the similarities of each workflow we will only cover the implemenatoins deatils in one service given the next service will more or less be a copy paste.

The idea of the test workflow will be to execute `go test -v ./...` automatically everytime someone create a PR/merge a PR the service repos.

Lets first start by creating a test.yaml workflow file under .github/workflows/ directory in the workout-management-service repo.

```bash
mkdir .github/workflows/test.yaml
```

Let's start by setting up a basic skeloton of the workflow, to start we will set a workflow that triggers on PR and pushes to main and simply print to console "testing"

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

To validate the workflow syntax is correct lets push this and create a PR against main, in the actions console we should see "testing" being printed for the test job

Now that we have a basic skeloton or our testing actions lets create a job that actual runs our unit test, to do this we need our actions to simpy run:

```bash 
go test -v .\...
```

Before our action job can sucessfully run the test we will need to run few set-up steps:

- Get the repo code into the action runner using the [actions/checkout@v4](https://github.com/actions/checkout)
- Install go into our action running using the [actions/setup-go@v5](https://github.com/actions/setup-go)
- Run docker compose to get the dynamodb db up and running using regular bash command `docker compose up`
- Ensure the tables required for the unit test are created in the dynamodb by running the create-table script under [scripts/dynamodb/create-table.sh](https://github.com/SamirMarin/workout-management-service/blob/main/scripts/dynamodb/create-table.sh)

once we have run these steps we should then be able to successfully run `go -v test ./...`

putting it all together into the test workflow:

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
      - name: Run docker compose
        run: docker compose up -d
      - name: Run dynamodb set-up scripts
        run: ./scripts/dynamodb/create-table.sh
        env:
          AWS_ACCESS_KEY_ID: test
          AWS_SECRET_ACCESS_KEY: test
      - name: test
        run: |
          go test -v ./...
        env:
          AWS_ACCESS_KEY_ID: test
          AWS_SECRET_ACCESS_KEY: test
```

lets now commit this and push up to our branch, we can validate it worked by referencing the github actions consule.