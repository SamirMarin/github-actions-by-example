# Section 3 - Building a workflow
Now that we have a basic understanding of GitHub Actions, let's build a common and useful workflow. This workflow will automatically create a GitHub release for us.

This workflow is beneficial for any project that relies on GitHub releases to distribute software. For example, if you're developing your own community GitHub action, you can use this workflow to automatically create a release for your action.

## Workflow
Our example will release a simple JavaScript GitHub action, the [get-labels-action](https://github.com/SamirMarin/get-labels-action), which provides an output depending on the labels of the pull request.

The specific details of the action aren't vital at this point. We will cover the implementation details of this action in the custom actions chapter.

You can find the complete source code for this example in the [get-labels-action release workflow](https://github.com/SamirMarin/get-labels-action/blob/main/.github/workflows/release.yaml).

The workflow we're building simply creates a GitHub release when a pull request merges into the main branch. This enables users to use the [get-labels-action](https://github.com/SamirMarin/get-labels-action) by referencing the release tag, like this:

example:
```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Get PR labels
        uses: SamirMarin/get-labels-action@v0
```

Here, any user of our action can utilize it by referencing the release tag `v0`.

Now that we have a basic understanding of what we're building, let's start by building the workflow triggers.

### Triggers
The first thing we need to do is define our workflow's triggers. We want our workflow to run when a pull request is created, a commit is pushed to a pull request, and when a pull request merges into the main branch.

We can define these triggers using the `on` keyword like this:

```yaml
on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main
```
By adding the `pull_request` trigger, our workflow will run when a pull request is created or updated. The `push` trigger ensures our workflow runs when we push a commit to the main branch. This is particularly useful when merging a pull request.

### Jobs
Having defined our triggers, we can start defining our jobs. We will define a single job called `release` that will run on the `ubuntu-latest` runner.

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
     ...
```

#### Steps
After defining our job, we can begin defining the steps that our job will run.

First, we'll use the [checkout action](https://github.com/actions/checkout) to checkout the repository files.
```yaml
    steps:
      - name: Checkout
        uses: actions/checkout@v3
```

Next, we'll use the [mathieudutour/github-tag-action](https://github.com/mathieudutour/github-tag-action) to increment our tag version and push the tag.

```yaml
    steps:
       ...
      - name: Bump version and push tag
        uses: mathieudutour/github-tag-action@v6.1
        id: tag_version
        with:
          default_bump: patch
          github_token: ${{ secrets.GITHUB_TOKEN }}
          dry_run: ${{ github.event_name == 'pull_request' }}
          tag_prefix: v
```

We use the `github_token` secret to authorize the action to push a tag to the repository. GitHub action workflows come with a `secrets.GITHUB_TOKEN` by default, which can access the repo where the action runs.

We use the `dry_run` option to ensure that the tag only gets pushed when the workflow is triggered by a `push` event. This prevents pushing a tag when creating or updating a pull request.

This action will then pull the latest tags from the repository. It will use the `tag_prefix` to match the latest tag and increment the version based on the `default_bump` option. In our case, we're using the `patch` bump, meaning the version will increase by a patch version (e.g., from v1.0.0 to v1.0.1).

Next, we'll use the [peter-evans/create-or-update-comment](https://github.com/peter-evans/create-or-update-comment) action to comment on the PR, indicating what the next release will be. While not necessary for the actual publishing of the release, it's a nice feature to add for user visibility on the version being released upon merge.

```yaml
    steps:
       ...
       - name: Comment on PR
         if: ${{ github.event_name == 'pull_request' && steps.tag_version.outputs.new_tag != '' }}
         uses: peter-evans/create-or-update-comment@v3
         with:
           issue-number: ${{ github.event.pull_request.number }}
           body: |
             **Next Release:** 🚀 ${{ steps.tag_version.outputs.new_tag }}
```

We use the action's `if` condition here to ensure that the action only runs when the workflow is triggered by a pull request. The comment provides information to the user when creating the pull request, ahead of merging.

We utilize the event object to get the pull request number, which is used to create the comment. We also use the `steps.tag_version.outputs.new_tag` to get the new tag version created by the `Bump version and push tag` to indicate the next release version.


>> One thing to note about this action is that it requires read and write permissions to workflows, you can set this by selecting read and write permissions under the repo settings->actions->general->workflow permissions.

Lastly, we'll add the [ncipollo/release-action](https://github.com/ncipollo/release-action), which will create a GitHub release for us. This is the final action required to complete the release workflow.

```yaml
    steps:
       ...
         
       - name: Create a GitHub release
         uses: ncipollo/release-action@v1
         if: ${{ github.event_name != 'pull_request' && steps.tag_version.outputs.new_tag != '' }}
         with:
           tag: ${{ steps.tag_version.outputs.new_tag }}
           name: Release ${{ steps.tag_version.outputs.new_tag }}
           body: ${{ steps.tag_version.outputs.changelog }}
```

We use the action's `if` condition here to ensure this only runs when the workflow is triggered by a `push` event. We also use the `steps.tag_version.outputs.new_tag` to get the new tag version created by the `Bump version and push tag` step and provide it as the tag for the GitHub release. We also utilize the `steps.tag_version.outputs.changelog` to get the changelog, and provide that as the body for the GitHub release that will be created.

### How to specify the type of release

Let's now revisit the `Bump version and push tag` step. We earlier hardcoded the input to `default_bump`, which isn't ideal if we want the ability to adjust our version by patch, minor, or major versions, depending on the type of release.

We need a method to specify the type of release we want. By default, any PR created should be a patch release and only when the user decides to create a minor or major release should they be able to do so.

One way to achieve this is by using pull request labels. If we had a mechanism to read labels, we could add a pull request label, for example, `bump:major`.

The [SamirMarin/get-labels-action](https://github.com/SamirMarin/get-labels-action) allows us to read the labels of a pull request and use them in our workflow.

We can add a step before the `Bump version and push tag` step that reads the labels and sets the `default_bump` input of the `Bump version and push tag` step to the value of the `bump` label.

```yaml
    steps:
       ...
       - name: Get labels
         uses: SamirMarin/get-labels-action@v1.0.0
         id: get_labels
         with:
           github_token: ${{ secrets.GITHUB_TOKEN }}
           label_key: get_labels
           label_value_order: "patch,minor,major,ignore"
           default_label_value: patch
           
       - name: Bump version and push tag
         uses: mathieudutour/github-tag-action@v6.1
         id: tag_version
         with:
           default_bump: ${{ steps.get_labels.outputs.labels.bump }}
           github_token: ${{ secrets.GITHUB_TOKEN }}
           dry_run: ${{ github.event_name == 'pull_request' }}
           tag_prefix: v
```
The `secrets.GITHUB_TOKEN` is set to permit the action to read the labels from the PR. The `label_key` indicates the key of the label that determines the bump type; in our case, it's `bump`. Therefore, if we apply a label with `bump:major`, it will know that the bump type is `major`. The `label_value_order` specifies the order of the label values, telling the action the precedence when multiple labels are added to the PR. The `default_label_value` defines the default value of the label if no value is specified.

The `bump` output for the `Get labels step` is used to specify the `default_bump` input for the `Bump version and push tag` step. This means if we add a label to the PR with `bump:major`, the `Bump version and push tag` step will run with `default_bump` set to major. As a result, the version will be bumped by a `major` version. For instance, from v1.0.0 to v2.0.0.

### The complete Workflow
Below is the comprehensive workflow encompassing all the steps we've discussed so far. I encourage you to test it and explore how it operates. To implement this in your repository, create a `.github/workflows/release.yaml` file. For testing purposes, you could even integrate this into an empty repository.

```yaml
name: Release

on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Get labels
        uses: SamirMarin/get-labels-action@v1.0.0
        id: get_labels
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          label_key: get_labels
          label_value_order: "patch,minor,major,ignore"
          default_label_value: patch

      - name: Bump version and push tag
        uses: mathieudutour/github-tag-action@v6.1
        id: tag_version
        with:
          default_bump: ${{ steps.get_labels.outputs.labels.bump }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          dry_run: ${{ github.event_name == 'pull_request' }}
          tag_prefix: v
          
      - name: Comment on PR
        if: ${{ github.event_name == 'pull_request' && steps.tag_version.outputs.new_tag != '' }}
        uses: peter-evans/create-or-update-comment@v3
        with:
          issue-number: ${{ github.event.pull_request.number }}
          body: |
            **Next Release:** 🚀 ${{ steps.tag_version.outputs.new_tag }}
            
      - name: Create a GitHub release
        uses: ncipollo/release-action@v1
        if: ${{ github.event_name != 'pull_request' && steps.tag_version.outputs.new_tag != '' }}
        with:
          tag: ${{ steps.tag_version.outputs.new_tag }}
          name: Release ${{ steps.tag_version.outputs.new_tag }}
          body: ${{ steps.tag_version.outputs.changelog }}
```

This workflow begins by checking out the repository. It then uses the get-labels-action to read the labels of a pull request and accordingly sets the bump version. Next, the Bump version and push tag step pushes the new version based on the bump type.

If the event is a pull request, the workflow comments on the PR about the next release. And finally, if the event isn't a pull request, a GitHub release is generated.