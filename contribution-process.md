#The Contribution Process

This document describes the process by which Technical Machine prefers to handle code contributions. It's designed to be very similar to other popular open-source projects.

## Contributor's License Agreement

There isn't one. Yay!

## Step 1

Create a Github issue on the appropriate repository if one hasn't been created already. If you're not sure which repository to assign it to, read our [System Overview](./system-overview.md) or shoot us an email (team@technical.io).

## Step 2

Comment on that issue to say that you're working on solving the problem so that we can avoid work duplication.

## Step 3

Create a branch of the repo that you're working on:

```
firmware >> git checkout -b add-teleporation-api
```

## Step 4

Fix the bug or add the feature. If you have any questions, feel free to comment on the Github issue and we'll get back to you as quickly as possible.

## Step 4.5

If you're adding a contribution as suggested from our [Task List](./task-list.md), we will provide a test case file that you'll need to test your addition. You'll know when you've finished the task when all the tests pass.

## Step 5

When you're ready to submit the contribution, [rebase your branch](http://git-scm.com/book/en/Git-Branching-Rebasing) on the master branch of that repository so that merging is as smooth as possible. 

## Step 6

Open a Pull Request on your branch which includes your fix as well as the passing test file. We'll either give you feedback on a few needed tweaks or accept your PR as is and merge it into master. 




