# Introducing GitHub Actions
GitHub Actions is an automation solution framework built within GitHub's ecosystem.

## Automation platform
Automation is the process of using a system to perform tasks that would typically require human intervention. Within the sphere of GitHub Actions, this automation is intricately linked to a given GitHub repository.

To make this concept more relatable, let's take the well-known "Hello World" example. GitHub Actions allows you to configure the system such that every time you push a change or create a pull request in the repository, "Hello World" will run automatically.
```yaml
on:
  push:
    branches:
      - master
  pull_request:
      branches:
      - master
          
    
jobs:
  hello-world:
    runs-on: ubuntu-latest
    steps:
      - name: Hello world
        run: echo "Hello world"
```

## Framework
In computing, a framework is a set of tools that are used to develop, manage, or run a specific type of software solution. For GitHub Actions, the framework is the YAML syntax employed to define automation workflows. The framework also comprises a supportive community offering a wide range of ready-to-use actions that can be incorporated into your workflows.

For instance, imagine extending the previous "Hello World" example to include commenting "Hello World" on every pull request. While you could use the [GitHub API](https://docs.github.com/en/rest/pulls/comments?apiVersion=2022-11-28#create-a-review-comment-for-a-pull-request) to create a bash step that adds this comment, it's easier and more efficient to use the [create or update comment](https://github.com/peter-evans/create-or-update-comment) community action.

```yaml
on:
  push:
    branches:
      - master
  pull_request:
      branches:
      - master
      
jobs:
  hello-world:
    runs-on: ubuntu-latest
    steps:
      - name: Hello world
        id: hello-world
        run: |
          hello_world="Hello world"
          echo "value=${hello_world}" >> $GITHUB_OUTPUT
          echo $hello_world

      - name: Comment on PR
        if: ${{ github.event_name == 'pull_request' }}
        uses: peter-evans/create-or-update-comment@v3
        with:
          issue-number: ${{ github.event.pull_request.number }}
          body: |
            ${{ steps.hello-world.outputs.value }}
```

By using a community action, you can add the comment in a single step, bypassing the need to delve into API implementation details. This illustrates how the surrounding GitHub Actions framework is as crucial as the automation it enables.

## Summary
In these examples, we touched on several concepts that might be new to you. Don't worry! We'll dive deeper into these subjects in subsequent sections. For now, just remember that GitHub Actions is an automation platform deeply ingrained in the GitHub environment, facilitating GitHub repository-specific tasks. The framework also provides a robust foundation that can significantly streamline your automation workflows.