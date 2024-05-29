# What is reusable workflow

A reusable workflow allows us to define a workflow that can be called from another workflow. We can think of it as a function. For example, referencing the deploy workflow we created in the previous chapter, if we define it as a reusable workflow, we won't need to define the workflow in each repository that requires it. Instead, we can call the reusable workflow with some given inputs.

```yaml
...
jobs:
  deploy:
    uses: SamirMarin/github-by-example-reusable-workflows/.github/workflows/reusable-deployment.yaml@deployment-v0
    with:
      name: example-service
    secrets: inherit
```

Instead of defining the deploy workflow in each repository, as we did in Chapter 2 for the workout-management-service and user-management-service, we can define it once and call it from each repository that needs it.

## Creating a reusable workflow

Let's jump right into what it takes to define a reusable workflow to make the concept clearer.

First, all reusable workflows must be defined with:

```yaml
on:
  workflow_call:
```

### inputs

Since reusable workflows are like functions, they can take inputs. Inputs give us control over how to differentiate between the different workflows that call the reusable workflow.

```yaml
on:
  workflow_call:
    inputs:
      name:
        required: true
        type: string
        description: "The service name"
```

We can also pass secrets into a reusable workflow. We must place these under secrets.

```yaml
on:
  workflow_call:
    inputs:
      name:
        required: true
        type: string
        description: "The service name"
    secrets:
      docker-secret:
        required: true
```

At this point, we can start defining the workflow as we would define any regular workflow, with the difference that we can use the inputs/secrets passed by the calling workflow.

```yaml
name: Print Info

on:
  workflow_call:
    inputs:
      name:
        required: true
        type: string
        description: "The service name"
    secrets:
      docker-secret:
        required: true

jobs:
  print-passed-info:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Print Name
        run: |
          echo The service is ${{ inputs.name }}
      - name: Print Info
        run: |
          echo ${{ secrets.docker-secret }} this value does not show because it is a secret.
```

## Where do reusable workflows live?
Reusable workflows can live anywhere. For example, you can define them in the same repository that calls them. However, since the goal is to reuse these workflows across multiple repositories, we will define them in a separate repository.

Let's create a new repository called github-by-example-reusable-workflows. In the next section, we will start defining the workflows we created in Chapter 2 as reusable workflows.


