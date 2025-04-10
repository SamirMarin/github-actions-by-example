# Section 2 - Composite Custom Actions

Composite custom actions allow you to define a series of steps that can be run as a single workflow job step. Similar to reusable workflows, composite actions let you group multiple steps together, but with one key difference: while reusable workflows encapsulate an entire workflow and cannot be called from a single step, composite actions wrap a sequence of steps into a single step that can be called within a workflow.

Additionally, when a reusable workflow runs, the full workflow is displayed in the GitHub Actions UI. In contrast, composite actions are treated as a single step, making them more streamlined and focused on specific tasks.

To make this concept clearer, let’s dive into an example. We’ll define the get-labels action as a composite action.

## Get-Labels custom action

Composite actions are defined in an action.yaml file. For this example, we will define our composite action in the [GitHub Actions by Example Reusable Workflows repository](https://github.com/SamirMarin/github-actions-by-example-reusable-workflows). While custom actions are usually defined in their own repository, for simplicity and to demonstrate how we can encapsulate multiple actions and workflows in a single repository, we will define it here rather than creating a new repo.

Let’s start by creating the directories and the action.yaml file:

```bash
mkdir -p .github/actions/get-labels-composite
touch .github/actions/get-labels-composite/action.yaml
```

### Purpose of the Action

This action reads a label from a pull request (PR) to determine the type of release we want to create. For example:

* bump:patch (e.i v0.0.1 --> v0.0.2)
* bump:minor (e.i v0.0.1 --> v0.1.0)
* bump:major (e.i v0.0.1 --> v1.0.0)

In these examples, the action will search for a label in the form [key:value](key:value) and output the label’s value.

### Defining the Action

#### Inputs and Outputs

Now, let’s start defining the composite action in the action.yaml file by specifying its name, description, inputs, and outputs:

```yaml
name: 'get labels'
description: 'Outputs a bump type based on the bump:<value> label'
inputs:
  label-key:
    description: 'The key of keyed label. i.e key:value'
    required: true
  default-label-value:
    description: 'The default label value if no label is found'
    required: false
    default: 'patch'
  github-token:
    description: 'Token to access github repo, needed for push events'
    required: true
outputs:
  label-value:
    description: "the value of the keyed label. i.e key:value"
    value: ${{ steps.label.outputs.value }}
```

The inputs make this action versatile, allowing it to handle various label formats, not just `bump:<value>`. If no label is found, the action defaults to `input.default-label-value` as the label value. Here’s what each input does:

* label-key: The key of the label to read (e.g., \<key>:\<value>).
* default-label-value: The default value to output if no label is found.
* github-token: A token required to access the PR, specifically for non-pull request events (e.g., push events).

The output, label-value, allows the value of the label to be passed to subsequent steps in the workflow, making it accessible for further decisions. In the case of releases the value is passed to decide what kind of bump to use in the version update for the release.

#### Action Logic

The composite action logic is composed of several steps. Since the purpose of the action is to retrieve pull request (PR) labels, we divide the logic into two separate steps: one for pull request events and another for push events.

* For PR events, the labels are already available in the github.event parameter since they are part of the pull request itself.
* For push events, however, labels are not directly available because a push event is not associated with a PR. Therefore, we need to make an API call to GitHub to retrieve the PR based on the commit hash and extract the associated labels.

The third step is responsible for setting the output label value by determining which previous step successfully found the label and setting it accordingly.

Here’s how the logic comes together:

```yaml
...
runs:
  using: "composite"
  steps:
    - id: push-event-label
      if: ${{ github.event_name != 'pull_request' }}
      run: |
        LABEL_VALUE=${{ inputs.default-label-value }}
        REPOSITORIES=$(curl \
          -H "Accept: application/vnd.github+json" \
          -H "Authorization: Bearer ${{ inputs.github-token }}" \
          https://api.github.com/repos/${{ github.event.repository.owner.login }}/${{ github.event.repository.name }}/commits/${GITHUB_SHA}/pulls)
        LENGTH=$(echo "$REPOSITORIES" | jq length)
        LABEL_KEY=${{ inputs.label-key }}
        if [[  "$LENGTH" == "1" ]]; then
          LABELS=$(echo "$REPOSITORIES" | jq '.[0].labels | .[].name')
          if echo "${LABELS}" | grep "$LABEL_KEY:major" ; then
            LABEL_VALUE="major"
          elif echo "${LABELS}" | grep "$LABEL_KEY:minor" ; then
            LABEL_VALUE="minor"
          elif echo "${LABELS}" | grep "$LABEL_KEY:patch" ; then
            LABEL_VALUE="patch"
          elif echo "${LABELS}" | grep "$LABEL_KEY:ignore" ; then
            LABEL_VALUE="ignore"
          fi
        fi
        echo "The push event label value is: $LABEL_VALUE"
        echo "value=${LABEL_VALUE}" >> $GITHUB_OUTPUT
      shell: bash
    - id: pull-request-event-label
      if: ${{ github.event_name == 'pull_request' }}
      run: |
        LABEL_VALUE=${{ inputs.default-label-value }}
        if ${{ contains(github.event.pull_request.labels.*.name, format('{0}:major', inputs.label-key)) }}; then
          LABEL_VALUE="major"
        elif ${{ contains(github.event.pull_request.labels.*.name, format('{0}:minor', inputs.label-key)) }}; then
          LABEL_VALUE="minor"
        elif ${{ contains(github.event.pull_request.labels.*.name, format('{0}:patch', inputs.label-key)) }}; then
          LABEL_VALUE="patch"
        elif ${{ contains(github.event.pull_request.labels.*.name, format('{0}:ignore', inputs.label-key)) }}; then
          LABEL_VALUE="ignore"
        fi
        echo "The pull request event label value is: $LABEL_VALUE"
        echo "value=${LABEL_VALUE}" >> $GITHUB_OUTPUT
      shell: bash
    - id: label
      run: |
        LABEL_VALUE=${{ inputs.default-label-value }}
        PULL_REQUEST_LABEL=${{ steps.pull-request-event-label.outputs.value }}
        PUSH_EVENT_LABEL=${{ steps.push-event-label.outputs.value }}
        echo "The pull request label value is: $PULL_REQUEST_LABEL"
        echo "The push event label value is: $PUSH_EVENT_LABEL"
        if [ -n "$PULL_REQUEST_LABEL" ]; then
          LABEL_VALUE=$PULL_REQUEST_LABEL
        elif [ -n "$PUSH_EVENT_LABEL" ]; then
          LABEL_VALUE=$PUSH_EVENT_LABEL
        fi
        echo "The label value is: $LABEL_VALUE"
        echo "value=${LABEL_VALUE}" >> $GITHUB_OUTPUT
      shell: bash
```

Summary of Steps:

1. Step 1 - Get the Push Event Label (only runs for push events)
   1. This step makes a curl request to the GitHub API to retrieve the PR associated with the commit. If multiple PRs are associated, the value remains empty. This can be updated to be more sophisticated, but in most cases, a single PR is associated with the commit, so this approach works well.
2. Step 2 - Get the Pull Request Event Label (only runs for pull request events)
   1. This step uses the github.event parameter to find the label. Since it is a pull request event, the labels are available in github.event.pull\_request.labels. If no label is found, the value defaults to empty.
3. Step 3 - Set the Output Label Value
   1. This step determines the final output label value. It checks if either Step 1 or Step 2 has set a value, and if so, uses that value. If neither step sets a value, it defaults to the default\_label\_value.

You can find the full source code for the composite action here:

[Get-Labels Composite Action Source](https://github.com/SamirMarin/github-actions-by-example-reusable-workflows/tree/main/.github/actions/get-labels-composite).

## Releasing and using the Composite Action

As we discussed in Chapter 3, Section 5: Releasing the Reusable Workflow, we added logic to release custom actions. This logic essentially looks for any custom actions defined under the .github/actions/ directory and makes a release for them in the [github-actions-by-example-resuable-workflows](https://github.com/SamirMarin/github-actions-by-example-reusable-workflows) repository.

To use the action, you can refer to it in a workflow step as follows:

```
- name: get-label
  id: label
  uses: SamirMarin/github-actions-by-example-reusable-workflows/.github/actions/get-labels-composite@get-labels-composite-v0
  with:
    label-key: bump
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

Alternatively, if the action lives in its own repository, you could release it in a similar fashion to how we did demonstrated in Release workflow example in Chapter 1, Section 3.
