+++
title = 'Automating Reviewer Assignments for NEO Sense and Monika'
date = 2024-05-14T15:55:00+07:00
draft = true
images = ['/automating-reviewer-schedule.webp']
+++

In Hyperjump, we handle two main projects: [NEO Sense](https://neosense.bgnlab.id) and [Monika](https://github.com/hyperjumptech/monika). Each day, we get assigned to review pull requests for these projects. To ensure fairness and randomness, at the start of every month, our team lead randomize the assignments, making sure each project has two reviewers, and no one is assigned to both projects on the same day. Our team lead used to manage this process manually in Excel, which was a pain the ass I'm sure.

To automate this task, I created a script that handles the randomization and assignment process, and then sends the results to Microsoft Teams using a webhook.

## The script

```js
const crypto = require("node:crypto");

// Add more reviewers here
const reviewers = [
  "Budhi Widagdo",
  "Denny Pradipta",
  "Hari Cahya Nugraha",
  "Kevin Hermawan",
  "Lukman Adisaputro",
  "Muslim Ilmiawan",
  "Raosan Fikri Lillahi",
  "Sinta Herena",
];

(async () => {
  // Generate random number
  async function rand(min, max) {
    return (
      (min +
        ((max - min + 1) * crypto.getRandomValues(new Uint32Array(1))[0]) /
          2 ** 32) |
      0
    );
  }

  // Create the placeholders
  const selected = [];
  const results = {
    monika: [],
    symon: [],
  };

  // Select the reviewers
  do {
    const selectedIndex = await rand(0, reviewers.length - 1);
    if (!selected.includes(selectedIndex)) {
      console.info("Selected", reviewers[selectedIndex]);
      selected.push(selectedIndex);
    }
  } while (selected.length < 4);

  // Assign reviewers to the projects
  for (const index of selected) {
    if (results.monika.length < 2) {
      console.info("Added", reviewers[index], "as a Monika Reviewer");
      results.monika.push(reviewers[index]);
    } else {
      console.info("Added", reviewers[index], "as a Symon Reviewer");
      results.symon.push(reviewers[index]);
    }
  }

  // Create a teams message
  fetch(process.env.TEAMS_WEBHOOK_URL, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      title: "Monika and Symon Reviewers",
      summary: "Here are the assigned reviewers for Monika and Symon today.",
      text: `Here are the assigned reviewers for Monika and Symon today:\n\n\n**Monika**: ${results.monika.join(
        ", "
      )}\n\n**Symon**: ${results.symon.join(", ")}\n`,
      themeColor: "#309942",
    }),
  }).then((response) => response.json());
})();
```

### How it Works

- **Random Reviewer Selection**: The script uses the Node.js crypto module to generate random numbers, ensuring the selection process is truly random.
- **Assignment Logic**: It then selects four unique reviewers from the list. The first two are assigned to Monika, and the next two to Symon.
- **Notification**: The script formats the assignment results and sends a message to our Microsoft Teams channel using a webhook, keeping everyone informed.

## The GitHub Workflow

To run this script daily, I use a GitHub Workflow scheduled to execute at midnight UTC every day, which is 07:00 GMT+7 in Indonesia. This timing is perfect as it runs just before the workday starts. However, for the initial setup, since the script was created at 10 AM, I need to be able to trigger it manually as well.

```yaml
name: Run Node.js script daily

on:
  schedule:
    - cron: "0 0 * * *"
  workflow_dispatch:

jobs:
  run-script:
    runs-on: ubuntu-latest

    env:
      TEAMS_WEBHOOK_URL: ${{ secrets.TEAMS_WEBHOOK_URL }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "20"

      - name: Run script
        run: node index.js
```

### Triggers for the Workflow

- **Daily Schedule**: The workflow runs every day at 00:00 UTC (07:00 GMT+7) to ensure the assignments are ready before the workday starts.
- **Manual Trigger**: The workflow_dispatch event allows us to run the workflow manually. This is especially useful for the first-time setup or any urgent reassignments during the month.

## Benefits

- **Time-Saving**: Automating this process saves us from the repetitive and error-prone task of manual assignments in Excel.
- **Fairness**: Randomizing assignments ensures a fair distribution of review tasks among team members.
- **Transparency**: By using Microsoft Teams, no one gets left behind and keeps everyone in the loop.

This script and workflow setup have significantly simplified our daily review assignment process, allowing the team to focus more on core development tasks rather than administrative overhead.
