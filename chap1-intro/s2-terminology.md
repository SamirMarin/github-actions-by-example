# Section 2 - Terminology and Core Concepts

Before diving into more complex GitHub Action workflows, let’s define some key terms and explore core concepts that are commonly used when building workflows.

## Workflow

A "workflow" refers to the automation process. Workflows are defined in YAML files and placed in the `.github/workflow/` directory in your repository.

Each workflow is made up of one or more "jobs", which are executed in parallel by default. Each job is then broken down into one or more "steps", which are executed sequentially by default.

In order for a workflow to run, it must be triggered by an "event". Events can range from things like a `push` to the repository or a `pull_request`.

## Triggers

"Triggers" are what initiate a workflow. They are defined in the `on` section of a workflow file.

For example, to set a trigger that starts a workflow when a pull request is created or updated, you would use the following syntax:

```yaml
on:
  pull_request:
    branches:
      - main
```

This trigger will start the workflow anytime a pull request is created or updated that's based on the `main` branch.

There are many different types of triggers, including `push`, `pull_request`, `workflow_dispatch`, `repository_dispatch`, and `schedule`, among others. In this course, we'll primarily focus on the `push` and `pull_request` triggers.

More details on different types of triggers can be found in the [GitHub Actions documentation](https://docs.github.com/en/actions/reference/events-that-trigger-workflows).

## Jobs

A "job" is a set of steps that are executed on the same runner. Jobs are run in parallel by default, but can be configured to run sequentially.

Here's a simple example of how to define a job:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Hello world
        run: echo "Hello world"
```

In this example, `build` is the unique name of the job. It will run on the `ubuntu-latest` runner and contains a single step that prints "Hello, world!" to the console.

### Runners

"Runners" are the virtual machines that execute the jobs. They are specified in the `runs-on` field of a job.

There are several types of Github-hosted runners available, including `ubuntu-latest`, `windows-latest`, and `macos-latest`, among others. You can also use self-hosted runners, which are machines that you manage and on which the GitHub Actions runner application is installed. We'll discuss [self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners) in more detail in a later chapter.

One important thing to note is Standard Github-hosted runners are [free to use in public repositories](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions)

### Steps

"Steps" are the individual tasks that make up a job. They are defined in the `steps` section of a job.

Each step has several properties. The most commonly used ones are `name`, which gives the step a name (useful for debugging), and `run`, which defines the command to be executed by the step. When using community actions, you will use the `uses` property to specify the action to be executed.

Here's an example of how to use the community [actions/checkout](https://github.com/actions/checkout) action to checkout the repository:

```yaml
steps:
  - name: Checkout repository
    uses: actions/checkout@v3
```

We will cover additional properties of steps as we go through more examples. For now, this should be enough to get you started with more complex examples.

## Expressions

GitHub Actions gives us the ability to use expressions to programmatically manage workflows and make them more dynamic. We saw examples of this in the previous section, whenever we opened double curly brackets like this: \`$\{{ \<expression> \}}\`.

Expressions are commonly used for:

* Accessing contextual information (e.g., github.event\_name).
* Setting environment variables.
* Applying conditions to control when steps, jobs, or workflows run.

Expressions are incredibly powerful because they allow us to combine functions, operators, variables, and contextual information to programmatically and dynamically handle workflows.

```yaml
- name: Check pull request
  if: ${{ github.event_name == 'pull_request' && github.event.pull_request.draft == false }}
  run: echo "This is a non-draft pull request."
```

In this example, the expression uses logical operators (&&), contextual information (github.event\_name), and a condition to ensure the step runs only if the workflow was triggered by a non-draft pull request.

Why Expressions Matter:

Expressions come with a variety of built-in functionality, including:

* Functions like contains(), startsWith(), or toJSON() for string and data manipulation.
* Operators like ==, !=, &&, ||, and arithmetic operators for comparisons and logic.
* Access to Variables and Contextual Information, making it possible to dynamically adapt workflows to different scenarios.

As we continue, we’ll use expressions more frequently and learn how to interpret and create them effectively. For now, the key takeaway is to recognize that expressions are available and can make your workflows smarter and more flexible. When you’re unsure what can be done with an expression, you can always refer to the official [GitHub Actions Expressions Documentation](https://docs.github.com/actions/learn-github-actions/expressions) for guidance.

## Contextual information

GitHub Actions provides a way to access information about a workflow run, referred to as [contextual information](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/accessing-contextual-information-about-workflow-runs). This data varies depending on the workflow’s conditions and can be used to make workflows more dynamic and responsive.

We saw an example of this in the previous section, where we used contextual information to obtain the pull request (PR) number for a workflow run. We also used it to ensure a step only ran when the workflow was triggered by a pull\_request event. This is because the PR number is only available in the context of a pull\_request event.

```yaml
- name: comment on PR
  if: ${{ github.event_name == 'pull_request' }} # contextual information
  uses: peter-evans/create-or-update-comment@v4
  with:
    issue-number: ${{ github.event.pull_request.number }} # contextual information
    body: |
      ${{ steps.hello-world.outputs.value }} # contextual information
```

It’s not feasible to memorize every piece of contextual information available. Instead, it’s more practical to understand that this information exists and know how to look it up when you need it. Whenever you identify a need for specific data in your workflows, you can refer to the official GitHub Actions documentation to see if the information is available.

Common Examples of Contextual Information:

* github.event.pull\_request.number: The number of the pull request that triggered the workflow.
* github.event.repository.name: The name of the repository where the workflow is running.
* github.event.repository.owner.login: The login of the repository owner.&#x20;
* github.event\_name: The name of the event that triggered the workflow.
* secrets.GITHUB\_TOKEN: A token automatically generated for the workflow to access the repository.

As we work through more examples, you’ll encounter additional pieces of contextual information. Over time, you’ll become more familiar with how to use them effectively. The key takeaway is to be aware of the existence of contextual information and to leverage the official [GitHub Actions Contexts Documentation](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/accessing-contextual-information-about-workflow-runs) for specifics on what is available and how to use it.

## Permissions

Every GitHub Actions workflow has access to a GITHUB\_TOKEN, which can be referenced using $\{{ secrets.GITHUB\_TOKEN \}}. This token is automatically generated at the start of each workflow run and grants limited access to the repository running the workflow. The default permissions of this token are designed to be restrictive, with read-only access for contents and packages permissions unless explicitly modified.

Default Permissions

The default permissions for the GITHUB\_TOKEN are determined by the repository settings. Repository administrators can configure these defaults to be either restrictive(read repository contents and packages permissions) or permissive(read and write permissions), for a list of the permissions each of the setting provides see [Permissions for the GITHUB\_TOKEN](https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication#permissions-for-the-github_token)

To change the default permissions to permissive:

* Go to Repository Settings.
* In the left sidebar, click Actions.
* Under Workflow permissions, select Read and write access.

Setting Permissions at the Workflow Level:

Permissions for the GITHUB\_TOKEN can also be configured directly within the workflow file using the permissions keyword. This allows granular control over what the token can access for a specific workflow. For example:

```yaml
permissions:
  contents: read    # Read-only access to repository content
  pull-requests: write # Write access to pull requests
  actions: none     # No access to actions
```

For a list of all available permissions see [Controlling permissions for the GITHUB\_TOKEN docs](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/controlling-permissions-for-github_token)

The GITHUB\_TOKEN is a powerful tool, but its default access is intentionally limited for security. By understanding how to modify its permissions at the repository or workflow level, you can tailor its capabilities to suit your workflows. As we progress through more examples, we will see how to leverage the GITHUB\_TOKEN effectively to automate tasks while maintaining security best practices.

