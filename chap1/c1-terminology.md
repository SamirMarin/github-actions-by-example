# Terminology
Before diving deeper into more complex GitHub Action workflows, let's take a moment to define some of the key terms used in GitHub Actions.

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