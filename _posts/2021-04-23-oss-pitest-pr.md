---
layout: post
title:  "Pitest on fork PRs for OSS projects"
author: henry
categories: [pitest, java]
image: assets/images/fork.jpg
description: "How to setup pitest on pull request"
featured: true
hidden: false
---

I [wrote recently](/dont-let-your-code-dry) about the work I've been doing with CDG to create a smooth process for running pitest against pull requests on github[^octocat]. This way is a great way to mutation test large projects, or to introduce it to projects where its not been used before.

There was a problem though. In order to update a PR, the plugin needs an api token with write access. For very good reasons github does not give write access to PRs that come from forks of a repo. So the plugin couldn't be used by most open source projects.

## The problem

When github actions runs a workflow triggered by a pull request from a fork, it provides it with an api token with a minimal set of privileges. 

This makes sense. A CI server is really just one huge arbitrary code execution vulnerability. It takes minimal effort to exploit. A PR could update the code in a repo to do absolutely anything, so giving it a token that could delete all your issues, merge itself and then drop your private repos would be a poor decision. For this reason, although we *could* make our own token with write access for the workflow, we absolutely shouldn't.

## The solution

Fortunately, updating things from a PR is not an uncommon requirement, and github provides a way to do it safely. It requires more moving parts, but the folk at github have done all the heavy thinking, so we don't have to.

It works like this. We have two workflows:

* [receive-pr](https://github.com/GroupCDG-Labs/pitest-github-demo/blob/main/.github/workflows/receive-pr.yml)
* [update-pr](https://github.com/GroupCDG-Labs/pitest-github-demo/blob/main/.github/workflows/update-pr.yml)

Receive PR runs on `pull_request` as normal. It does all the actual work of mutation analysis. The `GITHUB_TOKEN` it is supplied with is read only, but that doesn't matter as we don't use it. Instead, we use the `pitest-git:aggregate` goal to collect together all the json files that pitest generates to support pull requests. At the end of the workflow we upload these to github as an artifact.

Update PR is a little more tricksy. It's triggered by the github `workflow_run` event when the `receive-pr` workflow has finished. Github gives this workflow a write token, so it can update the PR. We do this with a new maven goal `updatePR`. This reads the files from the artifact after they've been extracted, and uses them to update the PR. It doesn't need a maven project to run, so we don't even need to check the code out in this workflow.

So, how does this solve the problem? We still have a write token, so someone could just update the `update-pr` workflow to do something nasty, couldn't they?

They could, but it wouldn't help them, because the version of `update-pr` that is run is always the one in the main branch. 

This makes the whole thing a bit fiddly and confusing to setup, as it won't work at all until the workflow has been merged in, but it also makes it secure. Arbitrary code can only be run with a privileged token if a project maintainer hits merge.

## Wrinkles

I hit a few surprised setting this up. The details of the PR are missing in the github json payload when the PR comes from a fork (they're there when the PR comes from a branch in the same repo). Presumably this is for security reasons. 

I can't imagine what nefarious things could be done with the data we need (the SHA of the merge request and the PR number), but perhaps other parts of the payload are more sensitive. To work around the issue, the SHA and PR number are also included in the json data collected by the `aggregate` goal.

There's some logic in the `updatePR` goal, so it is important that the version of the code running there is kept in sync with the code performing the analysis. As the version is hidden away in a yaml file, it will be very easy for this to get out of sync. In the future we'll look at shifting the logic out of there, so the code will rarely need to be updated. 

## How to set it up

The [demo project](https://github.com/GroupCDG-Labs/pitest-github-demo) has been updated to use both workflows, updating the PR directly when it comes from a branch, and via the two stage process when it comes from a fork. 

Feel free to create some PRs to try it out.

If you'd like to use it on your OSS project, please [get in touch](mailto:pitest.demo@groupcdg.com) and we'll give you a free licence. 

Really, don't be afraid to ask. We'd like to get as many people mutation testing as possible.

[^octocat]: Octocat image 
