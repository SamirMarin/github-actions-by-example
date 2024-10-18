# Section 1 - What is a custom action

Throughout the workflow examples we’ve covered so far, we’ve been using custom actions. In GitHub Actions, each step in a workflow either calls a custom action or executes shell commands (such as bash). We’ve been leveraging custom actions shared by the GitHub community to build our workflows efficiently.

One custom action that we’ve used in every single workflow so far is the [checkout action](https://github.com/actions/checkout). This is a custom JavaScript action developed and maintained by GitHub Actions, and it’s responsible for checking out your repository’s code so that subsequent workflow steps can interact with it.

There are three types of custom actions:

1. Docker container actions
2. &#x20;JavaScript actions
3. Composite actions

In this chapter, we’ll focus on JavaScript actions and composite actions, explaining how they work and how you can create and use them in your own workflows.

## The get-labels custom action

In the previous examples, we’ve used the [get-labels action](https://github.com/SamirMarin/get-labels-action), particularly in Chapter 1, Section 3: Building a Workflow, where we built the release workflow for the get-labels action. We’ve also utilized this action in every release workflow we’ve covered. The purpose of this action is to read a label from a pull request and determine what kind of release to perform—whether it’s a patch, minor, or major release.

In this chapter, we’ll implement the [get-labels action](https://github.com/SamirMarin/get-labels-action) in two different ways: as a composite action and as a JavaScript action. This will give us a deeper understanding of how custom actions work and how they can be structured to suit different needs.

