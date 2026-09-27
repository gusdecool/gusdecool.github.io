---
tags: [AI, AWS]
---

# Setting up an AI SDLC: label an issue, Claude plans and builds it

> How I set up Boma so that adding a label to a GitHub issue has Claude plan the work on
> Opus, build it on Sonnet, and open a pull request. It all runs in GitHub Actions, and the
> model bill goes to our own AWS account through Amazon Bedrock.

## TLDR
1. Add the label `ai:plan` to an issue. Claude (Opus) reads the issue and the code, asks its
   questions in the issue, then writes a spec and a plan in a draft pull request.
2. When the plan looks right, add `ai:build`. Claude (Sonnet) builds it on the same branch,
   runs the checks, and marks the pull request ready for review. I still review and merge.
3. The first real issue took 3 runs, about 29 minutes of Claude's time, and about **$5.71** of
   model usage. The GitHub Actions minutes were almost free next to that.
4. Most of the lessons came from real runs, not from the design. Plan for a few small fix-up
   changes after the first try.

## What is Boma, and why this
Boma is a dashboard I'm building at Bungamata. Like most small products, it has a long list of
small, well-understood issues. Each one is maybe an hour of work. Together they never get done.

I wanted a process where I write the issue, answer a few questions, approve a plan, and review
the result. The typing in between is done by an AI agent. I call it an AI SDLC: the normal
software development life cycle, with an agent doing the planning and building steps.

## Why I moved off OpenHands Cloud
My first try was OpenHands Cloud. It worked, but it was too slow for this. It also had a bug
that blocked the approve and reject buttons, which is exactly the part where I need to be in
control.

GitHub Actions was a better fit. Each run gets a fresh machine. The issue, the conversation and
the pull request already live in GitHub. And Anthropic ships an official action for it,
[claude-code-action](https://github.com/anthropics/claude-code-action), which can call Claude
through Amazon Bedrock. That means the model usage is billed to our AWS account, next to the
rest of our infrastructure, under a budget we already watch.

## How it works

| Step | Who | What happens |
|---|---|---|
| 1 | Me | Write the issue and add the label `ai:plan` |
| 2 | Claude on Opus | Reads the issue and the code. Posts all its questions in one comment, each with a suggested default |
| 3 | Me | Reply with `@claude` and the answers |
| 4 | Claude on Opus | Writes the spec and the plan as files in a draft pull request |
| 5 | Me | Read them. Add the label `ai:build` when they look right |
| 6 | Claude on Sonnet | Builds the plan, runs lint, types, tests and build, and marks the pull request ready |
| 7 | Me | Review and merge. A merge deploys to our test environment as usual |

A few design choices shaped the rest.

**Two stages, two models.** I chose Claude Opus 5.5 for brainstorming and planning because
it's the stronger reasoning model. This stage has to understand the issue, explore the code,
spot what's unclear, weigh the options and decide how the work should be done. A mistake here
carries into everything after it, so it's worth paying for the best reasoning.

For code generation I use Claude Sonnet 5, because it costs less. By the time it runs, the hard
thinking is done. The plan is approved and says which files to change and what to test. Sonnet
only has to read the plan and write the code, and it does that well. The build is also the
longest stage, with the most turns, so a cheaper model there saves the most money.

Each stage has its own instructions file, turn limit and time limit.

**The draft pull request holds the spec and the plan.** Each run starts from nothing. It has no
memory of the last run. So the issue thread and the files on the branch are the memory. Every
run reads them, does the next step, posts one comment and stops. Keeping the spec and plan as
files in the pull request means I review them like code, and the build continues on the same
branch.

**Claude acts as the official Claude GitHub App.** GitHub gives every workflow a built-in token.
The catch is that a push made with that token doesn't start other workflows, so our CI would
never run on Claude's work. A custom GitHub App would fix that, but it means another app and
another private key to look after. The official Claude app is free, needs no secret, and its
pushes do start CI.

**The AWS role can only call models.** GitHub Actions logs in to AWS with a short-lived token
(OIDC), so there are no AWS keys stored in GitHub. The role it gets can invoke Bedrock models
and nothing else. It is separate from our deploy role, so a workflow that runs the AI can't
deploy anything.

**A list of allowed repositories, not "any repo in the org".** I only want private repositories
to use this role. AWS can't check whether a GitHub repository is private, so the role trusts a
short, named list of repositories instead. A repository has to leave the list before it goes
public.

**Pinning the model names.** Our AWS region picks region-specific model IDs by default, and
those don't exist for these models in our account. Two environment variables point the short
names at the global versions, so `opus`, `sonnet`, and any helper agent that asks for
`sonnet`, all resolve correctly:

```yaml
env:
  ANTHROPIC_DEFAULT_OPUS_MODEL: global.anthropic.claude-opus-5-5
  ANTHROPIC_DEFAULT_SONNET_MODEL: global.anthropic.claude-sonnet-5
```

**Guardrails.** Only people with write access can start a run. Claude can use git, the GitHub
CLI, our package manager and file tools, nothing else. It can't merge, force-push, or edit
workflows or secrets. Merging always stays with me.

## Surprises found only in real runs

The workflow reacts to issue labels and comments, and GitHub only runs those workflows from the
main branch. So there was no way to test it before merging it. The first real issue was the test.
That trial was a small one: let people add Boma to their phone's home screen as an app.

Here is what broke, and what I changed.

**AWS rejected the role.** The Terraform plan looked fine, but creating the role failed. AWS has
a rule that a role trusted by GitHub's login must limit the token's `sub` value, which names the
repository and branch. My design checked a different field, `repository`, which AWS doesn't
accept on its own. A plan can't catch this, only the real apply. The fix adds `sub` patterns
built from the same repository list. GitHub's tokens carry numeric IDs in that value, so the
patterns allow for both forms:

```
repo:bungamata/boma:*
repo:bungamata@*/boma@*:*
```

**The action's default mode made a new branch every run.** The action has a mode that answers
`@claude` comments on its own. In that mode, a second run on the same issue started a new branch
from main instead of reusing the issue's branch, and it couldn't open a pull request. I switched
to agent mode, where my own instructions tell Claude to check out `claude/issue-<N>` and use
`gh` to open the pull request. The cost is that I lose the action's built-in progress comment.

**CI waited for my approval.** Claude's push showed up as `github-actions[bot]`, not as the
Claude app, so CI sat waiting for a manual approval. Version 7 of the checkout action stores its
token in a separate git config file, and the Claude action's cleanup didn't remove it. That token
won over the app's token. One line fixed it:

```yaml
- uses: actions/checkout@v7
  with:
    persist-credentials: false
```

**Replies were silently dropped.** I set up the workflow so each issue has one run at a time,
and new runs wait in line. But GitHub keeps only one waiting run per group. Any other event on
the issue, even one the job would skip, like adding another label, replaced my waiting
`@claude` reply. Moving the setting from the workflow to the job fixed it, because skipped jobs
never join the line:

{% raw %}
```yaml
jobs:
  claude:
    concurrency:
      group: ai-${{ github.event.issue.number || github.event.pull_request.number }}
      cancel-in-progress: false
```
{% endraw %}

**The build nearly ran out of turns.** A turn is one step for the agent: read a file, run a
command, edit something. I gave the build 150 turns. It used 149. One more and the run would
have failed. The limit is now 300 for build and 100 for plan.

A few smaller ones:

- The database for tests took about 90 seconds to start on every run, including plan runs that
  never use it. Now only build runs start it.
- After I replied `@claude approved`, nothing happened for minutes, which felt broken. Each run
  now posts "Got it" with a link to the run as its first step.
- The final review ran on Opus, because the review skill asks for "the most capable model".
  That was more than I needed, so all build helpers now run on Sonnet.
- Small issues needed two approvals, one for the spec and one for the plan. Now a small,
  well-bounded issue gets both in one run and one approval.
- The job log only showed the end result, so I couldn't see where 22 minutes went. Each run now
  keeps its full transcript for 14 days.

## What went well
The planning run judged the issue as small and clear, and went straight to a spec. It listed
four decisions, each with a default, so I approved it with one reply.

The build followed the three-task plan. Its final review caught something I like a lot: the
build's first fix for an icon bug worked, but it blamed the wrong cause. The review found the
real cause in how the image library applied two steps, and a simpler fix. At the end, Claude
listed the checks it couldn't do itself, like adding the app to a real iPhone and Android
phone, for me to do by hand.

CI ran on Claude's pull request and passed. From the first label to the merged pull request took
about 50 minutes. About 29 of those were Claude working. The rest was me reading and replying.

## The numbers

| Run | Stage | Started by | Job time | Claude time | Turns | Cost |
|---|---|---|---|---|---|---|
| 1 | Plan (Opus) | label `ai:plan` | 4m 58s | 3m 13s | 24 | $0.65 |
| 2 | Plan (Opus) | reply `@claude approved` | 4m 32s | 3m 36s | 32 | $0.67 |
| 3 | Build (Sonnet, review on Opus) | label `ai:build` | 23m 32s | 22m 39s | 149 | $4.39 |
| **Total** | | | **33m 02s** | **29m 28s** | **205** | **$5.71** |

The costs are Claude Code's own estimates at list prices. The AWS bill shows the real figure a
day later.

GitHub counted **34 Actions minutes**, because each job rounds up. Our organization gets 2,000
free minutes a month on private repositories, so this was free. Even at the paid rate of
$0.006 a minute, it would be about 20 cents.

That shows where the money goes. The machine running the job mostly waits for the model to
answer. **The tokens are the cost, not the compute.**

I also looked at running the agent on AWS Fargate instead of GitHub's runners. It didn't make
sense. The compute is already a few cents per issue and inside the free minutes. Fargate would
save almost nothing and add a cluster, a task definition and logging for me to maintain. If
I want to cut cost, the place to look is tokens: fewer turns, fewer runs, and the smaller model
wherever it's good enough. The fixes above (one approval for small issues, Sonnet for reviews)
do exactly that.

## What I learned

1. **Keep a human at the two gates that matter.** I approve the plan before any code is written,
   and I merge. Everything between is fine to hand off.
2. **The agent has no memory, so give it a place to keep one.** The issue thread and the branch
   are enough, as long as every run reads them first and leaves one clear comment at the end.
3. **Pick the model for the job.** A reasoning model (Opus) for deciding what to build, and a
   cheaper model (Sonnet) for writing the code from the approved plan.
4. **Some things can only be tested for real.** IAM rules, GitHub's token handling and how
   waiting runs behave all looked fine on paper. Expect a short round of small fixes after the
   first run, and write down what each one was for.
5. **Measure every run.** Each run now reports its time, turns and cost in its own comment, and
   merging the pull request posts a total on the issue. Without that, "it's cheap" is a guess.

## What's next
The next issue will be the first to use all the fixes. It should tell me whether one approval
and Sonnet-only reviews make each issue faster and cheaper. After that, I plan to add the same
workflow to our other private repositories. The AWS side already allows them. Each one only
needs the workflow file and the two labels.

----
If you have any feedback on this post, or a topic you'd like me to write about, let me know at
[budi.arsana@bungamata.com](mailto:budi.arsana@bungamata.com).
