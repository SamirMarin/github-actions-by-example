# Terminology
Before diving deeper into more complex github action workflows, let's take a moment to define some of the key terms used in GitHub Actions.

## Workflow
This is the name given to the automation process, workflows are defined in YAML files under `.github/workflow/` directory. 

A workflow is comprised of one or more jobs, which are executed in parallel by default. Each job contains one or more steps, which are executed sequentially by default.

For a workflow to run, it must be triggered by an event. There are various types of events, most commonly `push` and `pull_request` trigger workflows.

## Triggers
triggers are responsible for starting a workflow. Triggers are defined in the `on` section of a workflow file.

For example, to define a trigger that starts a workflow when a pull request is created or updated, you would use the following syntax:

```yaml
on:
  pull_request:
    branches:
      - main
```
This trigger will ensure anytime a pull request based on the `main` branch is created or updated, the workflow will run.

There are number of available triggers, some of the most common used ones including `push`, `pull_request`, `workflow_dispatch`, `repository_dispatch` and `label`. For more details on triggers, see the [GitHub Actions documentation](https://docs.github.com/en/actions/reference/events-that-trigger-workflows).

In this course we will focus mainly on `push` and `pull_request` triggers, as these are the most commonly used triggers.

## Jobs
A job is a collection of steps that are executed on the same runner. By default, jobs are executed in parallel, but you can configure them to run sequentially.

let's start of with the minimum requirement for a job definition:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Hello world
        run: echo "Hello world"
```

All jobs within a workflow definition are defined under the `jobs` section. Each job must have a unique name, in the above example the job name is `build`. Each job must specify which runners it will run on, in the above example the job will run on the `ubuntu-latest` runner. Finally, each job must contain one or more steps, in the above example the job contains a single step that prints `Hello world` to the console.

### runners
Runners are the machines that execute the jobs. Runners are defined in the `runs-on` section of a job definition.

The easiest way to get up and running is with standard Github-hosted runners, which are [free to use in public repositories](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions). There are a number of different runners available, including `ubuntu-latest`, `windows-latest` and `macos-latest`. For a more details on Github-hosted runner see [using Github-hosted runners](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners#using-a-github-hosted-runner).