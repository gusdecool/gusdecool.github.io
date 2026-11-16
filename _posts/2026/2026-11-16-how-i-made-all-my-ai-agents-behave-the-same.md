---
tags: [AI]
---

# How I made all my AI agents behave the same
> One shared instruction file to manage them all

## The problem
I use more than one AI coding agent: Claude Code, Codex, Gemini and OpenCode.
Each one is good at something different, and each one has a limit on how much I can use it per day or week.
So I switch between them a lot.

The pain is that every agent behaves differently.
One asks too many questions. One commits to the main branch. One merges a pull request when I did not ask.
When I switch, I have to teach the new agent my rules from zero.

I wanted this: switch agent, and the new agent behaves the same, instantly.

## The idea
Every agent reads an instruction file at the start of a session.
Claude reads `CLAUDE.md`, Codex reads `AGENTS.md`, Gemini reads `GEMINI.md`.
The names are different, but the idea is the same: a text file that says "this is how you should work".

So I do not write three files. I write one, and make the others point to it.

## How it works
I keep one git repository called `agent-standards`.
It holds the shared instruction file for all agents.

```
agent-standards/
  global/AGENTS.md     <- the one source of truth
  templates/           <- starter files for a new project
  install.sh           <- links the file into place
```

On my computer, I clone the repository and run `install.sh`.
The script creates symlinks. A symlink is a shortcut file: it has its own name, but opening it opens another file.

```
~/.claude/CLAUDE.md   ->  agent-standards/global/AGENTS.md
~/.codex/AGENTS.md    ->  agent-standards/global/AGENTS.md
~/.gemini/GEMINI.md   ->  agent-standards/global/AGENTS.md
```

Now all agents read the same text. There is only one real file.
If I change it, every agent sees the change on the next session. No copy and paste.

Each project can also have its own `AGENTS.md` for rules that only apply there.
The rule is simple: the closer file wins. If the project file says "override: this repo uses Jest", the agent follows it.

## The self improvement loop
This is the part I like most. The file is not written once. It grows from my complaints.

1. An agent makes a mistake. For example it asks me one question, waits for my answer, then asks the next one.
2. I complain, and tell it to ask all question at once.
3. The agent does not just say sorry. It edits the `agent-standards` repository with the fix and opens a pull request.
4. I review & approve, the agent merge & apply into my system.
5. All agents, all providers, now have the new rule.

Then I wait for the next mistake, and I repeat. I have done this 46 times so far.

The key is step 3. A correction I give in chat is lost when the session ends.
A correction written into the shared file survives, and it works for every agent, not only the one I was talking to.
I even have a rule that says this: do not keep durable guidance only in one provider's private memory,
because other agents cannot see it and will repeat the mistake.

## Every rule must say why
One rule I added early: every rule must carry its reason.

Here are real examples from my file:

- Ask all clarifying questions in one round. *Why: a session asked 4 questions because the tool allows 4, kept 3 more
  back, and made me answer twice.*
- Never merge a pull request unless I say so in this conversation. *Why: a merge is one way, and no earlier note can
  authorize it.*
- Use the API token that is already configured for Jira, do not start the OAuth login. *Why: an agent started the
  OAuth flow although the token was already there.*

The reason matters for two things.
First, the agent can judge the rule instead of following it blindly.
Second, I can delete the rule when the reason stops being true. Many rules are workarounds for a bug in a tool.
When the tool fixes the bug, the rule becomes noise. Without the reason, I would never know.

## Things I learned
**Keep the file short.** A long instruction file gets ignored. I move situational rules into skills
(small instruction packages) and keep only a trigger table in the main file: "when you commit, load the git skill".
The agent loads the details only when it needs them.

**Rules in a skill can fail to load.** So I keep a short list of the most important prohibitions in the main file
as well: no commit to the default branch, no force push, no secrets, no merge without my word.

**Add a test for the setup.** A small script checks that every skill named in the trigger table really exists.
A rule that points to a missing skill is a rule that never works.

**Verify like this.** Ask each agent: "summarize the rules you loaded". If the answers differ, something is not linked.

## Result
Switching agent now costs me nothing. When one provider hits its limit or down, I open another one and continue.   
The new agent already knows my rules about branches, pull requests, questions and cost.

It is also a good way to see mistakes as an investment.
Each time an agent does something wrong, I spend one minute to turn it into a rule, and that mistake is gone
for every agent I will use later.

46 iterations in, the agents still make new mistakes. But they rarely repeat the old ones, and that is the goal.

## Try it yourself
1. Create a git repository with one `AGENTS.md`.
2. Symlink it to the place each of your agents reads: `~/.claude/CLAUDE.md`, `~/.codex/AGENTS.md`, and so on.
3. Next time an agent annoys you, ask it to write the fix into that file, with the reason, and open a pull request.

That is all. Start with one rule.


## What next?

Once we have so many rules, it gonna cost us a lot initial tokens, especially when doing as task that is not related with the rules. This is the time where I start converting it into AI Skill.