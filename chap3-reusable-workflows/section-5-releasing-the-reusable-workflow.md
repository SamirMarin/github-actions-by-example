# Section 5 - Releasing the reusable workflow

Now that we have defined a reusable `test-build-deploy` workflow, we will create a workflow to release this reusable workflow. This will allow us to version the `test-build-deploy` reusable workflow, ensuring that each workflow using it has a stable version to reference.

To accomplish this, we will create a release workflow similar to the example shown in Chapter 1, Section 3: Building a Workflow. The difference is that we will adapt this workflow to handle multiple releases per repository. This approach enables us to host multiple reusable workflows within the same repository.

## Creating the Release Workflow

The process for creating the release workflow will be quite similar to what we did in Chapter 1, Section 3. However, instead of creating a single release, we will set up a system that creates a release for each reusable workflow in the repository.

Currently, the [`github-actions-by-example-reusable-workflows`](https://github.com/SamirMarin/github-actions-by-example-reusable-workflows) repository only hosts one reusable workflow, the `test-build-deploy` workflow. However, we want to configure the release workflow in a way that allows it to handle any new reusable workflows added to the repository, ensuring they are released independently.

One approach would be to add a separate release workflow for each new reusable workflow. However, this is inefficient and cumbersome. A better solution would be to handle the release of multiple reusable workflows within a single workflow file.

In traditional programming, when we need to repeat an action, we typically use a `for` loop. GitHub Actions provides a similar capability through the [**matrix strategy**](https://docs.github.com/en/enterprise-server@3.14/actions/writing-workflows/choosing-what-your-workflow-does/running-variations-of-jobs-in-a-workflow). By utilizing a matrix, we can automate the release process for each reusable workflow in the repository using a single release workflow.

The release process will occur in two jobs:

* **Job 1: `workflows-to-release`** – This job builds the matrix values of the reusable workflows that are ready for release.
* **Job 2: `release`** – This job releases the reusable workflows that have new changes.

### Workflows to Release: The Matrix Job

First, let's create `release.yaml` under the `.github/workflows/` directory in the `github-`[`actions-by-example-reusable-workflows`](https://github.com/SamirMarin/github-actions-by-example-reusable-workflows) repository.

Since this is a release workflow, we will configure triggers to ensure the workflow runs on `push` and `pull_request` events targeting the `main` branch.

The matrix job will monitor for any changes to reusable workflows. If a reusable workflow file is modified, we will extract the name of the file and generate matrix values—a list of these names. These values will then be used in the matrix strategy of the release job.

When creating matrix values in GitHub Actions, you are essentially generating a list (or array) that can be iterated over and utilized within a job. For our use case, we will create matrix values consisting of a list of strings—those strings being the names of the reusable workflows that are ready for release. These values will later be passed into a matrix strategy in the release job.

Since the reusable workflow repository may also contain non-reusable workflow files, we will ensure that all reusable workflows are prefixed with `reusable-`. This way, we can easily filter out non-reusable workflows and focus only on those that need to be released.

**Approach for the Job:**

1. Identify all the files that have changed.
2. If a changed file is prefixed with `reusable-`, extract the file name.
3. Place the name into the matrix value list.

Now, let's put it all together in the `release.yaml` workflow file:

```yaml
name: release

on:
  push:
    branches:
      - main
    paths:
      - ".github/workflows/reusable-**"
    tags:
      - '*'

  pull_request:
    paths:
      - ".github/workflows/reusable-**"
    types:
      - opened
      - synchronize
      - reopened
      - labeled
      - unlabeled
    branches:
      - main

jobs:
  workflows-to-release:
    runs-on: ubuntu-latest
    outputs:
      names: ${{ steps.changed-workflows.outputs.values }}

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get changed files
        id: changed-files
        uses: tj-actions/changed-files@v45

      - name: Get changed workflows ounder /workflows
        id: changed-workflows
        run: |
          changed_workflows=$(echo "${{ steps.changed-files.outputs.all_modified_files }}" \
          | tr " " "\n" |  sed '/^\.github\/workflows\/reusable-/!d' \
          | sed 's/^\.github\/workflows\/reusable-//g;s/\.yaml//g' \
          | sort -u | tr "\n" "," | sed 's/.$//')
          
          # Extract action directory names without considering individual files
          changed_actions=$(echo "${{ steps.changed-files.outputs.all_modified_files }}" \
          | tr " " "\n" |  sed '/^\.github\/actions\//!d' \
          | sed 's/^\.github\/actions\///g' | awk -F'/' '{print $1}' \
          | sort -u | tr "\n" "," | sed 's/.$//')
          
          changed=''
          if [ ! -z "$changed_workflows" ] && [ ! -z "$changed_actions" ]
          then
            changed="${changed_workflows}, ${changed_actions}"
          elif [ ! -z "$changed_workflows" ]
          then
            changed="${changed_workflows}"
          elif [ ! -z "$changed_actions"  ]
          then
            changed="${changed_actions}"
          fi
          echo "Changed Workflows: $changed"
          echo "values=${changed}" >> $GITHUB_OUTPUT
```

**Let's walk through each step:**

* **Step 1: `actions/checkout@v4`**\
  Checks out the repository code.
* **Step 2: Get changed files**\
  Uses the `tj-actions/changed-files` action to gather the list of files that were modified.
* **Step 3: Get changed workflows under `/workflows`**\
  Filters the list of changed files to identify workflows prefixed with `reusable-`. This step ensures we’re only tracking reusable workflows for release. You'll also notice filtering for custom actions (prefixed with `.github/actions/`)—this is in preparation for Chapter 4, where we will discuss custom actions and their releases.\
  We collect the names of all workflows or actions ready for release and store them in `$GITHUB_OUTPUT.values`

Lastly, we place the list of changed workflows (or actions) into  `job.outputs.names` so that it can be utilized in the matrix strategy for the release job.

### The Release job
