# Section 3 - The build and push reusable workflow

Similarly to the reusable test workflow, the build and push workflow will closely resemble the one we created in Chapter 2, Section 4. The main difference is that this new workflow job will be reusable, allowing it to be called from other repositories.

## Creating the reusable workflow job

We’ll start by referencing the logic used in Chapter 2, Section 4 (The build and test job) and refactor it to part of a reusable workflow.

### Setting up the reusable workflow files

First, we will rename the `reusable-test.yaml` file located in the `.github/workflows/` directory of the [`github-actions-by-example-reusable-workflows`](https://github.com/SamirMarin/github-actions-by-example-reusable-workflows) repository to `reusable-test-build.yaml`.

### Refactoring the build and push workflow

Next, we’ll refactor the build and push workflow by copying it from one of our service repositories and placing it below the `test` job in our newly renamed reusable workflow file. For this example, we'll use the [`build` and `push` job](https://github.com/SamirMarin/user-management-service/blob/8ea4779ec3beb9368f99953aaf3b7fb02c09ef54/.github/workflows/test-build-deploy.yaml#L38-L68) from the `test-build-and-deploy` workflow in the `user-management-service` repository and modify it to be reusable.

To make the `build` and `push` job reusable, we’ll follow a similar process as with the `test` job. Since it’s now part of a reusable workflow file, we only need to introduce the necessary inputs and utilize them within the job.

We’ll add the following inputs, providing default values for each:

* image-owner:  the repository owner (the ORG or username)
  * defaults to: `${{ github.event.repository.owner.login }}`
* image-name: the image name
  * defaults to: `${{ github.event.repository.name }}`
* build-tags: the build tags
  *   defaults to:&#x20;

      ```
            type=sha,prefix=,format=long
            type=ref,event=branch
            type=ref,event=pr
      ```
* build-push: whether to push the image after the build
  *   defaults to:&#x20;

      ```
      ${{ github.event_name != 'pull_request' }}
      ```

We’ll now place the `build` and `push` job into the reusable workflow, integrating the inputs we just defined:

```yaml
name: test-build
...
on:
  workflow_call:
    inputs:
...

      image-owner:
        required: false
        type: string
        description: "The repository owner (the ORG or username)"
        default: ${{ github.event.repository.owner.login }}
      image-name:
        required: false
        type: string
        description: "The image name"
        default: ${{ github.event.repository.name }}
      build-tags:
        required: false
        type: string
        description: "The build tags"
        default: |
          type=sha,prefix=,format=long
          type=ref,event=branch
          type=ref,event=pr
      build-push:
        required: false
        type: boolean
        description: "Whether to push the image after build"
        default: ${{ github.event_name != 'pull_request' }}
jobs:
  test:
  ...
  build:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - name: checkout
        uses: actions/checkout@v4
      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ inputs.image-owner }}/${{ inputs.image-name }}
          tags: ${{ inputs.build-tags }}
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Login to Github Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          push: ${{ inputs.build-push }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}


```

