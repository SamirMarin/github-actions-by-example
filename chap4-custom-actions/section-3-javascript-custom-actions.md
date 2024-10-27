# Section 3 - Javascript Custom Actions

JavaScript custom actions allow you to define an action using JavaScript, giving you the full power of a programming language. To demonstrate the capabilities of JavaScript in custom actions, we’ll build a get-labels action. This action will be similar to the composite get-labels action but will also include additional features, leveraging the flexibility of JavaScript.

Although you can use any programming language to create a custom action, only JavaScript actions can run natively on GitHub Action runners. If you want to use another language, you would need to set it up as a Docker container custom action. We’ll focus on JavaScript because it runs natively on GitHub runners, making it more efficient and straightforward to use.

However we’ll use TypeScript instead of Javascript to implement our action. TypeScript is a superset of JavaScript that adds static typing, which helps catch errors early and improves code maintainability. Using TypeScript for our action allows us to leverage these benefits while still compiling down to JavaScript for native execution on GitHub runners.

Let’s dive into implementing the get-labels action in JavaScript.

## Get-labels Javascript custom actions

Just like with composite actions, we define the inputs and outputs for a JavaScript custom action in an action.yaml file. The main difference is that in the runs section, we specify node as the runtime and reference the JavaScript file containing our action logic.

Step 1: Initialize the JavaScript Project:

We’ll create the JavaScript get-labels custom action in its own repository: [Get-Labels Action Repository](https://github.com/SamirMarin/get-labels-action). Once you’ve created and cloned the repository, initialize a JavaScript project in the root:

```bash
npm init -y
```

Step 2: Set Up TypeScript

Since we’ll be using TypeScript, install TypeScript and generate the configuration file:

```
npm install typescript --save-dev #intalls typescript
npx tsc --init #generates the tsconfig.json:
```

Step 3: Define the Action Index.yaml

Next, create an index.yaml file to define the action’s inputs and outputs. These will be similar to the inputs and outputs for the composite action, but with some added flexibility, given the power of JavaScript.

```bash
touch index.yaml
```

Add the following inputs and outputs to index.yaml:

```yaml
name: 'Get labels action by key'
description: 'Gets PR label by a specified key'
inputs:
  label_key:
    description: 'the key of a keyed label i.e <label_key>:<label_value>'
    required: false
    default: ''
  default_label_value:
    description: 'the default value if key does not match any label'
    required: false
    default: ''
  label_value_order:
    description: 'order of preference if multiple keyed labels found'
    required: false
    default: ''
  github_token:
    description: 'github token used to obtain labels on non-pull request events'
    required: false
outputs:
  label_value:
    description: 'the value of the keyed label'
  labels:
    description: 'list of all pr labels, seperated by a comma'
```

Explanation of Additional Inputs and Outputs

Compared to the composite action, we’ve added one new input and one new output:

* label\_value\_order (input): Allows us to specify a preference order for labels. For example, if the labels bump:patch and bump:major are found, and label\_value\_order is set to major,minor,patch, then the action will output major as it appears first in the order.
* labels (output): Provides a list of all labels found in the PR, separated by commas, which can be used in subsequent workflow steps if needed.

Specifying the Runs Section

In the runs section of the index.yaml file, we specify that the action will use Node.js 20 and provide the path to the JavaScript code to run.

```yaml
...
runs:
  using: 'node20'
  main: 'dist/index.js'
```

Here’s what each part does:

* using: 'node20' specifies the Node.js version that GitHub Actions will use to run our action.
* main: 'dist/index.js' points to the compiled JavaScript file that TypeScript outputs in the dist directory.

With this configuration in place, our get-labels JavaScript action is set up to run, leveraging JavaScript’s full flexibility and TypeScript’s type-checking capabilities.
