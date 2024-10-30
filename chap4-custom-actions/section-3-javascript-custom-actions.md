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

### Writing the TypeScript Code for the Action

Now, let’s jump into writing the TypeScript code for our get-labels action. We’ll write the entire program in TypeScript, then compile it to a dist/index.js file for GitHub Actions to run.

#### Step 1: Set Up the Entry Point

Start by creating an `index.ts` file. This will serve as the main entry point for our action.

```bash
touch index.ts
```

```typescript
import * as core from '@actions/core';
import { processTrigger } from "./src/action";

async function run() {
    try {
        const labels = await processTrigger();
    } catch (error) {
        if (error instanceof Error) {
            core.setFailed(error.message);
        }
    }
}

run();

```

This file is straightforward and ensures a clean structure for our action:

* We import @actions/core from GitHub’s Actions toolkit, which provides utilities like setting outputs and handling failures.
* We also import processTrigger from ./src/action, which will contain the primary logic for processing labels.

**Explanation of run Function**

The run function acts as the entry point for our action:

* Calling processTrigger: processTrigger is the function that will handle the logic for processing PR or push event triggers.
* Error Handling: If processTrigger encounters an error, we catch it. We use core.setFailed(error.message) to log it, marking the GitHub Action as failed.

This setup keeps index.ts minimal, delegating the work to the src/action.ts file. In the next step, we’ll start by defining the processTrigger function.

#### Step 2: processTrigger function

Next, let’s create the core logic for fetching PR labels in the processTrigger function.

First, create the necessary directory and file:

```bash
mkdir src
touch src/actions.ts
```

In src/action.ts, start by defining the processTrigger function, which will retrieve labels based on the event type (either pull\_request or push).

<pre class="language-typescript"><code class="lang-typescript">import * as github from "@actions/github";

<strong>export async function processTrigger() {
</strong>    let labels;
    if (github.context.eventName === 'pull_request') {
        labels = github.context.payload?.pull_request?.labels || [];
    } else {
        labels = await getPushEventLabels();
    }

    setOutputs(labels);
}
</code></pre>

The processTrigger function handles retrieving labels based on the GitHub event type:

* Pull Request Event: When the event type is pull\_request, labels are readily available through the github.context.payload object. We can directly access them using github.context.payload?.pull\_request?.labels || \[].
* Push Event: If the event is not a pull request, we use the getPushEventLabels function. This function will handle making a GET request to GitHub to find the associated PR for the commit and retrieve its labels, similar to the approach in the composite action.

Once we’ve obtained the labels, we pass them to the setOutputs function. This function will set the labels as outputs for other steps to use, similar to the output handling in the composite action.

With processTrigger in place, the next step is to define the getPushEventLabels function, which will retrieve labels when the event is a push. This function will make an API call to GitHub to find the associated pull request and gather its labels.

#### Step 3: getPushEventLabels funcition

Below the processTrigger function, add the getPushEventLabels function in src/action.ts:

```typescript
import * as core from "@actions/core";
import { Octokit } from "@octokit/core";
import * as github from "@actions/github";

async function getPushEventLabels() {
    const github_token = core.getInput('github_token');
    if (!github_token) {
        core.error("github_token required for push events");
        return [];
    }

    const octokit = new Octokit({ auth: github_token });

    const pulls = await octokit.request('GET /repos/{owner}/{repo}/commits/{commit_sha}/pulls', {
        owner: github.context.repo.owner,
        repo: github.context.repo.repo,
        commit_sha: github.context.sha,
        headers: {
            'X-GitHub-Api-Version': '2022-11-28'
        }
    });

    return pulls.data[0]?.labels || [];
}
```

This function retrieves the labels associated with a commit during a push event. It performs a GET request to GitHub’s API to find the pull request (PR) related to the specific commit.

* GitHub Token: We start by getting the github\_token input with `core.getInput('github_token')`. This token is necessary for authenticating API requests to GitHub. If the token is not provided, an error is logged, and an empty array is returned.
* Octokit Initialization: We create a new instance of Octokit (GitHub’s REST API client) and authenticate it using the `github_token`.
* API Request: The function makes a request to the endpoint `GET /repos/{owner}/{repo}/commits/{commit_sha}/pulls` to find pull requests associated with the current commit. The request uses values from github.context to dynamically fill in the repository owner, name, and commit SHA.
* Extract Labels: Since this is a push event, we expect the commit SHA to be associated with only one PR. We retrieve the labels of the first (and only) PR in the response list with pulls.data\[0]?.labels. If no PR is found, it returns an empty array.

With getPushEventLabels complete, the action can now handle both pull\_request and push events, effectively retrieving labels for each case.

The last step is to define the setOutputs(labels) function, which will complete the final piece of our puzzle by setting the outputs for other steps to use.

#### Step 4: setOutputs(labels) function

Below the getPushEventLabels function, add the setOutputs function in src/action.ts:

```typescript
function setOutputs(labels: { name: string }[]) {
    const labelNames = labels.map(label => label.name);
    core.setOutput("labels", labelNames.join(','));

    const labelKey = core.getInput('label_key');
    const keyedValues = labelNames.filter(
        labelName => labelName.startsWith(labelKey + ":")
    ).map(
        keyedLabel => keyedLabel.substring(labelKey.length + 1)
    );

    const valueOrder = core.getInput('label_value_order');
    const valueOrderArray = valueOrder.split(',');
    let outputValue = '';

    for (let value of valueOrderArray) {
        if (keyedValues.includes(value)) {
            outputValue = value;
            break;
        }
    }

    if (!outputValue) {
        outputValue = keyedValues.length > 0 ? keyedValues.sort()[0] : core.getInput('default_label_value');
    }

    core.setOutput("label_value", outputValue);
}

```

This function takes in an array of label objects of the type `{ name: string }`, representing the list of labels obtained from either a pull\_request or push event. Here’s how each part works:

**Extract All Label Names**

We start by extracting the label names with const labelNames = labels.map(label => label.name); and then set our first output, which is a comma-separated list of all labels found. This output is set using `core.setOutput("labels", labelNames.join(','))` providing the additional benefit of listing all labels thanks to JavaScript’s flexibility.

**Filter for Specific Label Key and Extract Values**

Next, we filter labels to find the specific label value associated with the input `label_key`. We retrieve the `label_key` input using const `labelKey = core.getInput('label_key')` and then filter and map the labels to only include those that start with `<label_key>:`. This results in a list of values associated with the key.

**Determine the Preferred Output Value**

If multiple values are found, we use the `label_value_order` input to specify a preference order for outputting values. The valueOrder input is split into an array, and we loop through it, outputting the first matching value found. If no preference order is specified, we sort the list of values and output the first one. If no labels are found, we default to `core.getInput('default_label_value')`.

**Set the Final Output**

Finally, we set the output value using `core.setOutput("label_value", outputValue)`, making it available for other actions to use, just as we did in the composite action.
