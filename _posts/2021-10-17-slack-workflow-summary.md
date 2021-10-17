---
layout: post
title:  "GitHub Actions workflow summary Slack message"
date:   2021-10-17
category: actions
tags: github actions open-source
---

TL;DR: I've released a custom [GitHub Action] that posts a summary of your
GitHub Actions workflow run to Slack once it's completed. You can find it at
[lewis-od/slack-workflow-summary].

[Github Action]: https://github.com/features/actions
[here]: https://github.com/lewis-od/slack-workflow-summary

## Background

On my current project at my day-job, we're practicing continuous deployment,
whereby each merge to the `main` branch triggers a [GitHub Actions] workflow that
deploys the changes to staging & production.

[GitHub Actions]: https://github.com/features/actions

We'd like to keep an eye on this process, so the final job in the workflow
posts a message to Slack containing the workflow outcome, and the result of each
of the jobs within the workflow. The messages look something like this:

![Original Slack message](/assets/actions/OldSlackSummary.png)

This was done using the [8398a7/action-slack] action, and passing in a *huge*
`custom_payload`, which contained a block for each job in the workflow that
looked like:
{% raw %}
```
{
    "type": "section",
    "text": {
        "type": "mrkdwn",
        "text": '${{ needs.deploy_production_terraform.result }}' === 'success' ? ':heavy-check-mark:   Deploy Production Terraform' : '${{ needs.deploy_production_terraform.result }}' === 'failure' ? ':heavy-cross-mark:   Deploy Production Terraform' : ':heavy-minus-sign:   Deploy Production Terraform'
    }
}
```
{% endraw %}
Which as you can imagine was a nightmare to edit! It also meant that every time
you added/removed a job to/from the workflow, you had to manually update the
Slack message payload. This was very error prone, and people often forgot to
update it. (You may have noticed some jobs appear twice in the image above!)

[8398a7/action-slack]: https://github.com/8398a7/action-slack

Since at the point the post to Slack action was running, the workflow hadn't yet
finished, we also had to manually compute a variable to indicate whether or not 
the workflow as a whole was successful. This consisted of a long `if` statement
of the form:
```
needs.build_cache.result == 'success' &&
needs.package_function_apps.result == 'success' &&
needs.deploy_staging_rg_terraform.result == 'success' &&
...
```
which was equally as painful to maintain as the message payload.

After a while I get fed up of fiddling around with all of this, and decided to
write my own action to automate the process!

![Automate all the things meme](/assets/actions/AutomateAllTheThings.jpeg)

## The Action
The project at work uses full-stack TypeScript, so it made sense to write the 
action in TypeScript too, using GitHub's [typescript-action] template as a 
starting point. The action calls the GitHub API to fetch data about the jobs
in the current workflow, filters out all the jobs that haven't finished running 
yet (hopefully just the one that's running this action), then uses the data to 
construct and post a Slack message.

[typescript-action]: https://github.com/actions/typescript-action

Using it is as simple as (example taken from the action repo itself):
{% raw %}
```yaml
slack_summary:
  name: 'Post summary to Slack'
  runs-on: ubuntu-latest
  if: always()
  needs:
    - test
    - lint_and_format
    - check_dist
  steps:
    - name: 'Post summary'
      uses: lewis-od/slack-workflow-summary@v1
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        slack-webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
        success-emoji: ':heavy-check-mark:'
        skipped-emoji: ':heavy-minus-sign:'
        failed-emoji: ':heavy-cross-mark:'
```
{% endraw %}
Which produces the following:

![New Slack message](/assets/actions/NewSlackSummary.png)

You can also provide a `custom-blocks` input containing some JSON representing
message components that will be inserted between the job summary and the 
timestamp footer. This could be useful for including links to the workflow run,
the deployed website, manual approval steps, etc.

## Conclusion

Now we can happily make changes to the workflow, without having to worry about
messing around with Slack JSON every time, yay! 🎉

You can find the action, along with more detailed info on how to use it at
[lewis-od/slack-workflow-summary].

[lewis-od/slack-workflow-summary]: https://github.com/lewis-od/slack-workflow-summary
