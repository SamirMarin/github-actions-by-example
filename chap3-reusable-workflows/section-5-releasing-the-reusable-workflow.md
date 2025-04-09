# Section 5 - Releasing the reusable workflow

Now that we've defined a reusable `test-build-deploy` workflow, the next step is to create a release workflow. This allows us to version the reusable workflow, ensuring that each workflow referencing it has a stable version to rely on.

To achieve this, we will set up a release workflow similar to the example in Chapter 1, Section 3: Building a Workflow. However, this workflow will be adapted to handle multiple releases within a single repository, allowing us to host and release multiple reusable workflows.

## Creating the Release Workflow

The release workflow will be similar to what we did in Chapter 1, Section 3. However, instead of handling a single release, we will set up a system that releases each reusable workflow in the repository independently.

Currently, the [github-actions-by-example-reusable-workflows](https://github.com/SamirMarin/github-actions-by-example-reusable-workflows) repository only hosts the `test-build-deploy` workflow, but we want to ensure the release workflow can handle future workflows as well. This will prevent the need to create separate release workflows for each reusable workflow, which would be inefficient.

In traditional programming, repetitive actions are handled with loops, and GitHub Actions provides a similar capability using the [matrix strategy](https://docs.github.com/en/enterprise-server@3.14/actions/writing-workflows/choosing-what-your-workflow-does/running-variations-of-jobs-in-a-workflow). By using a matrix, we can automate the release process for each reusable workflow using a single workflow file.

The release process will consist of two jobs:

* **Job 1: `workflows-to-release`** – This job builds the matrix values (a list) of the reusable workflows ready for release.
* **Job 2: `release`** – This job performs the release of the reusable workflows that have new changes.

### Workflows to Release: The Matrix Job

First, let's create `release.yaml` under the `.github/workflows/` directory in the `github-`[`actions-by-example-reusable-workflows`](https://github.com/SamirMarin/github-actions-by-example-reusable-workflows) repository.

Since this is a release workflow, we will configure triggers so the workflow runs on `push` and `pull_request` events targeting the `main` branch.

The matrix job will monitor for any changes to reusable workflows. If a reusable workflow file is modified, we will extract the name of the file and generate matrix values (a list of workflow names) to use in the matrix strategy of the release job.

When creating matrix values, you are essentially generating an array that can be iterated over. For our use case, this array will contain the names of the reusable workflows ready for release. These values will be passed into the matrix strategy of the release job.

To distinguish reusable workflows from other files, we'll ensure that all reusable workflows are prefixed with `reusable-`, allowing us to filter only the relevant files.

Here’s how the job will work:

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
        uses: tj-actions/changed-files@v46

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
          
          # Trim any leading/trailing spaces or newlines from the final result
          trimmed_changed=$(echo "$changed" | jq -c .)

          echo "Changed Workflows: $trimmed_changed"
          echo "values=$trimmed_changed" >> $GITHUB_OUTPUT
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

This job will be very similar to the one in Chapter 1, Section 3. However, there are two main differences:

1. The strategy will be a matrix, allowing us to release each workflow independently.
2. We will also create a major version tag (`v1`) for each release. This will allow users to track the major version and avoid frequent updates when there are only minor or patch changes. For example, if we release version `v1.0.0`, we will also create a `v1` tag. If we later release a patch version `v1.0.1`, we will override the `v1` tag with this version.

Here is the release job:

```yaml
...
release:
    runs-on: ubuntu-latest
    if: ${{ needs.workflows-to-release.outputs.names != '[]' }}
    needs: [workflows-to-release]
    strategy:
      fail-fast: false
      matrix: 
        workflows: ${{ fromJson(needs.workflows-to-release.outputs.names) }}
    steps:
      - uses: actions/checkout@v4

      - name: Get bump version from PR labels
        id: bump_label
        uses: SamirMarin/get-labels-action@v0
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          label_key: bump
          label_value_order: "patch,minor,major,ignore"
          default_label_value: patch

      - name: Bump version and push tag
        id: tag_version
        uses: mathieudutour/github-tag-action@v6.2
        with:
          fetch_all_tags: true
          github_token: ${{ secrets.GITHUB_TOKEN }}
          default_bump: ${{ steps.bump_label.outputs.label_value }}
          tag_prefix: ${{ matrix.workflows }}-v
          dry_run: ${{ github.event_name == 'pull_request' }}

      - name: Create major version tag value
        id: major_tag_version
        run: |
          major_version=$(echo ${{ steps.tag_version.outputs.new_tag }} | cut -d "." -f 1)
          echo "value=${major_version}" >> $GITHUB_OUTPUT

      - name: Override or push major tag
        if: ${{ github.event_name != 'pull_request' }}
        uses: rickstaa/action-create-tag@v1
        with:
          tag: ${{ steps.major_tag_version.outputs.value }}
          force_push_tag: true

      - name: Comment on PR
        if: ${{ github.event_name == 'pull_request' }}
        uses: peter-evans/create-or-update-comment@v3
        with:
          issue-number: ${{ github.event.pull_request.number }}
          body: |
            **The current major Release:** 🚀 ${{ steps.major_tag_version.outputs.value }}
            **Next Release:** 🚀 ${{ steps.tag_version.outputs.new_tag }}

      - name: Create or update major GitHub release
        uses: ncipollo/release-action@v1
        if: ${{ github.event_name != 'pull_request' }}
        with:
          tag: ${{ steps.major_tag_version.outputs.value }}
          name: Major Release ${{ steps.major_tag_version.outputs.value }}
          body: ${{ steps.tag_version.outputs.changelog }}
          allowUpdates: true
          replacesArtifacts: true

      - name: Create a GitHub release
        uses: ncipollo/release-action@v1
        if: ${{ github.event_name != 'pull_request' }}
        with:
          tag: ${{ steps.tag_version.outputs.new_tag }}
          name: Release ${{ steps.tag_version.outputs.new_tag }}
          body: ${{ steps.tag_version.outputs.changelog }}
```

Lastly, since we are commenting on pull requests in the "Comment on PR" step, pushing tags, and creating releases, we need to ensure that the workflow has the appropriate write permissions for both pull requests and repository contents. This will allow the workflow to comment on PRs, push tags, and create releases. We can set these permissions at the top of the workflow file as follows:

```yaml
permissions:
  contents: write
  pull-requests: write
```

#### Explanation of Steps

* **Matrix strategy**: Allows the release workflow to run for each reusable workflow, passing in a list of workflows to release.
* **Get bump version from PR labels**: Extracts the version bump (e.g., patch, minor, major) based on labels, defaulting to a patch if no label is found.
* **Bump version and push tag**: Increments the version and pushes the tag.
* **Create major version tag**: Creates a major version tag (e.g., `v1`).
* **Override or push major tag**: Updates or creates the major tag.
* **Comment on PR**: Comments on the pull request with the current and next release versions.
* **Create or update major GitHub release**: Creates or updates a major GitHub release.
* **Create a GitHub release**: Creates the GitHub release for the specific version.

## Using the reusable workflow

Now that we’ve successfully released our reusable workflow, let’s put it to use in our services. To demonstrate this, we’ll update the test-build-deploy.yaml workflow to reference the reusable workflow instead of duplicating its logic. As an example, we’ll update the [test-build-deploy workflow](https://github.com/SamirMarin/user-management-service/blob/main/.github/workflows/test-build-deploy.yaml) for the user-management-service.

To do this, we’ll remove everything under the jobs section in the test-build-deploy.yaml workflow and reference the reusable workflow instead. Here’s how you can do it:

```yaml
name: test-build-deploy with reusable

on:
  push:
    branches:
      - main

  pull_request:
    branches:
      - main

jobs:
  test-build-deploy:
    uses: samirmarin/github-actions-by-example-reusable-workflows/.github/workflows/reusable-test-build-deploy.yaml@test-build-deploy-v0
```

This setup will call the reusable workflow and execute all its jobs. In this case, we don’t need to pass any parameters because we’re fine with the default values. However, if you wanted to override a default parameter, you could do it by using the `with` keyword. For example:

```yaml
...
uses: samirmarin/github-actions-by-example-reusable-workflows/.github/workflows/reusable-test-build-deploy.yaml@test-buil
with:
  image-name: user-mgmt
```

Here, the image-name parameter is overridden with the value user-mgmt, instead of using the default `${{ github.event.repository.name }}.`

After committing this change, the workflow will run, and you’ll see that all three jobs defined in the reusable workflow are executed. This greatly simplifies the logic, as we no longer need to define separate workflows for each service. Instead of copy-pasting similar workflows across services, we can centralize the logic in one place and reuse it across multiple projects.

You can follow the same steps to update the [workout-management-service](https://github.com/SamirMarin/workout-management-service) workflow to reference the reusable workflow in the same manner.
