# Section 1 - Composite Custom Actions

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

The inputs make this action versatile, allowing it to handle various label formats, not just bump:\<value>. If no label is found, the action defaults to `input.default-label-value` as the label value. Here’s what each input does:

* label-key: The key of the label to read (e.g., \<key>:\<value>).
* default-label-value: The default value to output if no label is found.
* github-token: A token required to access the PR, especially for non-pull request events (e.g., push events).

The output, label-value, allows the value of the label to be passed to subsequent steps in the workflow, making it accessible for further decisions. In the case of releases the value is passed to decide what kind of bump to use in the version update for the release.

Next we define the action logic.

