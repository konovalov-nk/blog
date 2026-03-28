---
title: "Solving the Code Review Bottleneck, Part 1: Stacked Commits"
date: 2026-03-23
draft: true
tags: [git, code-review, stacked-commits, minimal-viable-context, software-engineering]
description: "Decompose your work into logical, single responsibility, atomic commits."
---
# Solving the Code Review Bottleneck, Part 1: Stacked Commits

## Preamble

Here's a few useful resources on git/code review processes I stumbled
upon some time ago:

- [Mind Your Git Manners](https://8thlight.com/blog/kevin-liddle/)
- [5 Useful Tips For A Better Commit Message][ref-thoughtbot-commit]
- [A Note About Git Commit Messages][ref-tbaggery-commit]
- [RailsConf 2015: Strong Code-Review Culture][ref-railsconf-review]

I recommend reading it before reading this post. This is not a new
problem, and it's not going to be solved by AI. It's a problem of human
nature, and it's going to be solved by human nature. And I'm going to
share my thoughts on how to solve it. And I'm making a Cargo CLI tool
(TBA: `stacked-commits`) to help us!

## Table of Contents

- [TL;DR](#tldr)
- [Why this is even more important today][toc-why-important]
- [Why use Version Control at all?](#why-use-version-control-at-all)
- [LLMs](#llms)
- [Okay, how to do it then?](#okay-how-to-do-it-then)
- [What is good Context? (aka Minimal Viable Context, MVC)][toc-mvc]
- [What should go into Why?](#what-should-go-into-why)
- [Automating this](#automating-this)
- [Conclusion](#conclusion)
- [Acknowledgments](#acknowledgments)

## TL;DR

Use `stacked-commits` (TBA: link to the repository) to decompose your
work into logical, single responsibility, atomic commits.

- Decompose your work into logical, single-responsibility, atomic
  commits.
  How exactly small or large? Read on.
- Use both commit title (one line, max 50 characters) and body (one
  blank
  line after title), multiline at 72 chars with context + why, details,
  lists, and so on.
- Adapt granularity to your workflow: stacked diffs or a regular pull
  request.
- Every single mergeable unit of work must pass the CI pipeline.
- Separate code/spec changes from refactor and linter commits.
- Use `.git-blame-ignore-revs` for lint/format commits unless they
  change
  behavior.
- If a lint/format commit is entangled with behavior changes, untangle
  it.
- Keep work small, easy to verify, and documented for reviewers.
- Commits are a history of changes that capture context/why behind a
  change.
- This helps you remember what the original problem was.
- It guides other developers to understand context and why decisions
  were made.
- It provides LLMs with better context; always pull the last 5-10
  commits.
- Use `git log -p -- <filename>` to see changes to a specific file.
- Use `git log -p` to see changes in a branch.
- Do not use Jira/bugtracker references as a substitute for proper
  descriptions. Example: "See TICKET-XYZ-123" as the only commit body
  makes context disappear when the tracker is gone.

## Why this is even more important today

Even before explosion of AI/LLM, this was the flow I've been using
successfully:

- Dive into codebase, figure out which files are important
- IDE displays latest commit message for every single line of code you
  step into
- Commit messages contain previous features and bugfixes, changes, and
  context
- `git blame` further helps you recover history of a file and understand
  even more
- Instead of immediately adding feature/patching a bug, ask questions:\
  "What previous developers struggled on? What assumptions they had?
  Were they correct?" "Am I fire-fighting or I can fix a root-cause
  problem nobody managed to solve yet?" "Is there anything important I
  should know around problem I'm solving?" "Requirements say "do X", but
  `git blame` shows that X is incorrect, should I bring it up?"

Unless you work in a company that maintains proper product specs and
have elaborate testing suite, the codebase always captures specs,
implicitly.

But also it means specs are just tribal knowledge, and you cannot trust
codebase. This is a separate issue, which I'll talk about in later
articles, and proper commit workflow is one of the important foundation
layers for the future product spec work.

Failure to document work artifacts is as severe as not understanding
what we're doing. If you notice this point team to it, become a role
model on how to do it, but things continue as it is, it can only mean
proper work is not encouraged, and so the team is set to failure. And
your best option is to switch teams or even leave the company. Sad, but
that's SWE for you.

Recently I even stopped accepting Merge Requests unless author writes
down what they did and why, while providing minimal viable context
(MVC). This might sound extreme but so far I only had positive
experience, where people realized it makes them better engineers,
because they start caring about "what and why I did this".

## Why use Version Control at all?

To ensure we have no merge conflicts.

But also to track changes over time, and time travel to better
understand past decisions that led to current code. Lack of proper
commit messages, feature work intertwined with refactor and linting
All this adds friction when you're trying to understand intent of
a previous change. And it makes a poor impression of a genuine effort.
We're pressed for time, constraints, lack of requirements.

The code gets merged but the tech debt that was added on top of it, with
lack of explanation of why makes it even worse. And it reaches a point
where you cannot reason about the codebase anymore.

It makes you want to start from scratch. But you can't. So you come up
with workarounds. TODOs. Patches. Inventing bicycles.

The slop work you've been forced into.

And it all starts right there, with the mental model: "Am I presenting
my work properly or not?".

## LLMs

Lately, we have new tooling that allows us to process and write a lot of
text, quickly.

And it's our responsibility to make sure they work with absolute best
practices, acting as role models for us, for future developers.

The power of LLM is to predict a next token. What does this mean for
engineers? After 3 years of using them, I'm still convinced this is
true:

- Describe your problem and goal
- Include existing spec, how exactly system operates and what is
  important
- What are invariants? Constraints?
- What is current context? What is past context?
- Which decisions in the past have been made and why?

LLM can predict what should happen next as an effort to improve
system.

That's what prompt/context engineering is for.

If your commit history looks like this:

```text
* 7d0fc3e typo
* 8fc509a more changes
* efe5fc5 add test
* 447c5a0 updates
* 29f73ed more changes
* cefaa18 add file
```

This is not useful to anybody. It just increases entropy for your
project. By definition, this is "software rot" (or specifically, "Active
rot").

Don't ever do this. Not for your pet project, not even under the stress
of failing production that costs you annual coffee budget per 1 minute
of downtime. Always capture intent of your work, and why your change is
supposed to work. Assume that after pushing a commit you gonna get hit
by a bus in the next second.

And if you have trash in your commit history, you have trash in
your brain. You stop thinking clearly when all you see is bunch of
meaningless commits.

## Okay, how to do it then?

Let's assume you start with some idea on how to improve code, or add a
feature. You might not have a clear picture on changes you are about to
make. That's fine. Programming is about changing things and seeing what
happens anyway.

So after some time, you end up with something like this:

```shell
$ git status --porcelain -u | awk 'NF>1 {print $2}' \
  | sort -u | tree --fromfile
.
├── docs
│   └── docker-32gb-rtx5090.md
├── examples
│   └── monitoring
│       ├── docker-compose.yaml
│       ├── grafana
│       │   ├── dashboards
│       │   │   ├── config
│       │   │   │   └── dashboard.yaml
│       │   │   └── json
│       │   │       └── fish-speech-dashboard.json
│       │   └── datasources
│       │       └── datasource.yaml
│       ├── prometheus.yml
│       └── README.md
├── fish_speech
│   ├── inference_engine
│   │   └── \_\_init\_\_.py
│   └── models
│       └── text2semantic
│           └── inference.py
├── Makefile
├── pyproject.toml
├── scripts
│   ├── e2e_vram_log.sh
│   ├── plot_vram_steps.py
│   └── run_server_32gb.sh
├── tools
│   ├── api_server.py
│   └── server
│       ├── inference.py
│       ├── metrics.py
│       ├── model_manager.py
│       └── views.py
└── uv.lock

16 directories, 20 files
```

Now, there's a lot of things:

- Documentation
- Scripts
- Makefile changes
- Tools
- New packages
- Grafana/prometheus examples, dashboards

If you just squash this into a single commit, congrats! Now everybody
hates you. Even if you added just one feature, one commit for 20 files
is terrible. Unless it's autoformatter/linter commit, which we MUST
always add to `.git-blame-ignore-revs`.

> What to do instead?

Make up a story. Everything starts with something:

- "Once upon a time, in a faraway kingdom ..."
- "... there lived a poor woodcuter"
- "... a quick snapshot of the life before things happen ..."

If you take a look at previous `git diff`, what we can use to start
it? To me, it seems like `uv.lock` + `pyproject.toml` can be a perfect
beginning:

```markdown
Add prometheus-client to dependencies

# Context

We are adding instrumentation for our TTS model inference metrics.
The goal is to have insights into VRAM usage, performance, TTFA.
This would help us better understand where is the bottleneck
and tune performance and RAM usage accordingly.

# Why

For instrumentation, we decided to use Grafana/Prometheus stack.
Grafana and Prometheus would not be managed by this repository,
however we would provide with examples on how to set it up later.

For this repo, we only focus on exposing `/metrics` endpoint for
Prometheus.
Therefore, adding `prometheus-client` is the first step.
```

That's it. This is your setup. What should be next? Remember the rule:
every mergeable unit of work (single commit or a merge request) must
pass CI step. In practice, that isn't always possible, and quick example
could be updating linter settings. If you make linter changes as a
single commit, and then obviously entire codebase has to be updated, it
means you are making two commits, and previous one would fail on CI.

However, this is almost always an edge case. And normally you should be
able to design your changes in a way that prevents a single commit to
depend hard on previous one. If you ever find yourself in a situation,
where commit A depends on commit B, and B depends on A, congrats! You
just created a circular dependence.

And so good practice is to use bottom-up approach:

- Packages
- Utilities/tooling
- Models/ORM
- Services/Workflows
- Controllers
- Routes -> /metrics
- Documentation

## What is good Context? (aka Minimal Viable Context, MVC)

If you think about this for just one minute here what is going to
happen:

- What are minimal details to include?
- Am I sure it is enough for other engineers?

If you are following specific engineering standards, it is reasonable
to say:

- "The absolute minimum is the feature we're building, and the current
  state of it." For example:

```markdown
This is Text-to-Speech API, and we are trying to add streaming mode.
For the moment being, we are not emitting every VQ code from LLM, but
instead generating N tokens, which are then sent to codec (DAC) to
render it as PCM chunk. Then chunks are stitched together and response
is served as wav/mp3 file.

To add streaming mode, we have to start emitting VQ codes to DAC as
soon as they are ready, and allow DAC to batch process them
independently of LLM. However, the problem is that memory consumption
can significantly rise, so we have to add memory profiling first to see
exactly when we need to optimize.
```

To me this is comprehensible, because I just spent 4 days learning all
of that. But to this other poor engineer who is going to review it? Does
they know what DAC is? VQ codes? How LLM-based TTS engines are working?

That's a huge assumption. And if true, this means your stakeholders are
either PhDs in Machine Learning, or your median company tenure is at
least 5 years.

Still, this is better than nothing, as LLMs can do a good job explaining
what all of this means to a normal person.

> But can we do better?

After digging through it, we can define context problem like this:

```python
minimal_viable_context = f(mvc_engineer, mvc_stakeholder)
# where f -> min/max/avg
```

This is always going to be a tradeoff between "this is too difficult"
and "too much water". And of course it depends on two things:

- Who is going to review this code today
- Who is going to review this code 5 years from now on

My take on this: if we optimize for stakeholder understanding, they
could verify your work without spending much time learning about your
domain. And in theory it gives everyone ability to review your PR and
think outside the scope:

- Someone would say "oh but we actually wrote a Python library for
  adding `/metrics` endpoint recently"
- Someone would mention "this prometheus client version has security
  vulnerability we found last month, use version X.Y." Getting this
  feedback can help a lot and is valuable.

> However if all you see is nitpicking, then go fix your engineering
> process first, then come back here.

OK, hope you fixed it and nobody mentions a single comment about how
many tabs and spaces, and what linter settings should be. And the only
real questions are domain-specific.

But what if we can put less words on paper and optimize for engineer
productivity? Think again -- are you sure engineers in 2035 would
still operate on same context as you? Can you even guarantee this for
yourself? When was the last time you had to look into kernel code?
(hardware engineers: yeah, you're free to go, you can skip this)

Are you definitely, most likely sure what `DAC` means in 2035? I mean,
to me it's still "Digital to Audio Converter", but this codebase has it
as "Descript Audio Codec" (Kumar et al., 2023)

And so the very minimum I'd do to clear this up, is to just add a
reference in the footnote [1]. Which means: if you don't understand what
it is, GO READ IT, MAKE YOURSELF SMARTER.

```markdown
This is Text-to-Speech API [1], and we are trying to add streaming mode.
For the moment being, we are not emitting every VQ [2] code from LLM,
but instead generating N tokens, which are then sent to codec (DAC) [3]
to render it as PCM/Wav chunk. Then chunks are stitched together and
response is served as wav/mp3/opus file.

To add streaming mode, we have to start emitting VQ codes to DAC as
soon as they are ready, and allow DAC to batch process them
independently of LLM. However, the problem is that memory consumption
can significantly rise, so we have to add memory profiling first to see
exactly when we need to optimize.

[1](https://github.com/fishaudio/fish-speech/blob/v2.0.0-beta/)
[2](https://en.wikipedia.org/wiki/Vector_quantization)
[3](https://github.com/descriptinc/descript-audio-codec)

Part of: TICKET-XYZ-123 (effort to add low-latency streaming mode)
```

As you can see, I omitted some links because I don't think anyone needs
explanation of what LLM is, especially in the context of a TTS inference
service.

Now, why this format? Honestly, I just came up with it, while I was
writing 🤣 That's the power of writing your thoughts down. And it just
feels right to me.

> Don't duplicate knowledge. Link it

Context is just knowledge. Your scribes. My scribes. Someone else's
repositories. It's all just mainly text and images. LLMs are perfect for
processing it. So why should we fight this idea and not include Minimal
Viable Context to every single commit?

Here's one small issue though I want to point out. We wrote this 3
paragraph context block. Links, footers, references, all that is good.
But hey, very next logical commit after that also connects to this very
tiny, minimal context, and maybe adds just a tiny bit more context, or
none at all.

> Do I want to copy this over and over?

And I don't think it make sense to do it. Again, this is simply a
tooling problem. Here's what should actually happen that you probably
haven't realized yet.

> When you are reading a commit message, context is pulled for you.
> Automatically.

What this means is that your commit preview tool should look into
references, extract relevant links, disambiguation, enrich it if needed,
and provide you with knowledge you lack, immediately.

If adding and getting context gives you friction, nobody is going to
deal with it. And so it is in our best interest we solve this problem.
Today. Unless it is solved already (I didn't do research yet, please
excuse me 🤣)

Anyway, what to with overlapping context? I think you already know the
answer:

> references to previous commits

And if we standardize how commit viewer pulls context, then this should
have first-class support. So the next commit that adds one more thing on
top of same context can look like this:

<!-- markdownlint-disable MD053 MD034 -->

```markdown
# Context

See [1].

... extra context if needed ...

[1]\: git:3e5c1e60269ae0329094de131227285d4682b665
```

<!-- markdownlint-enable MD053 MD034 -->

## What should go into Why?

Now, if context is your setup, then Why is your punchline.

```markdown
We are adding necessary packages. Prometheus/Grafana is chosen stack,
and we decided to use minimal prometheus-client to implement the
`/metrics` endpoint.
```

This is simple enough. But sometimes you get feedback, you fight with
problems, and you learn something:

```markdown
Turns out, this specific version vX.Y contained security vulnerability,
and so right now as patch is still not released, version is fixed
at X.Z < X.Y to prevent deploying problematic code until it is fixed.
```

And there's also one more thing, which I think matters: how to verify
this work. For simple changes like typo there's nothing to verify. But
for diff that changes workflow, you want to test E2E flows that are
affected. And this is yet another problem with code reviews, which I'm
not gonna solve today, but it is on my radar. I believe this step could
be automated as well, but still some testing instructions are very
helpful for another engineer.

Better yet, don't test manually. Everything should be automated. E2E
included. If you have to test manually? At least document it: add it to
repo, and link to testing steps in your commit messages, same as you do
with context. Human or agent can go through them, and you don't have to
duplicate knowledge again.

This is bread and butter of `git log`, `git blame`, and yours truly.
This is how problems are captured for future generations to observe,
learn existence of, or be completely ignored (with consequences of
course).

> And so I don't think there's anything else to it, really.

But I'm not done yet.

## Automating this

So the idea is quite simple at first glance

- You have unstashed work as input
- You want to have meaningful commits as output
- While spending as little time on it as possible
- And tooling to help bridge the gap between "Nobody really knows what
  should go into commit body" and "commits are first-class decision
  documentation"

First, we need data structure to stage work in layers:

- `git` itself
- alternatively, a folder structure representing all commits:

```shell
.git
└── stacked-commits
    └── ticket-xyz-1231
        ├── BASE_BRANCH
        ├── BASE_COMMIT
        ├── meta.json
        └── v1
            ├── 001
            │   ├── context.md
            │   ├── deleted.json
            │   ├── files
            │   │   ├── pyproject.toml
            │   │   └── uv.lock
            │   └── why.md
            └── 002
                ├── context.md
                ├── deleted.json
                ├── files
                │   └── tools
                │       └── server
                │           └── metrics.py
                └── why.md
```

Folder idea is terrible because now I need to track deleted files
somehow. And initially I thought I could just copy file from specific
commit id to capture it. But how do you copy a deleted file? 🤣 You
write it down somewhere. Renaming files is deleting old one and copying
new one. Anyway, I think this is terrible to implement.

However, I think you noticed that context/why, and commit configuration/
meta are simple files. This way you can arrange them in any way you
want, display it, condense and so on, using simple CLI.

The workflow essentially this:

- Make sure `git status` shows `has nothing to commit, working tree
  clean`
- If you need to commit, do it. If there are changes you don't need,
  stash them: `git stash -u`
- Make a new branch out from current:
  `git switch -c "stacked-commits/$(git branch --show-current)/v1"`
- Choose path, whichever is easier: (1) interactive rebase
  (2) soft-reset (3) hard-reset + cherry picking
- The first iteration is figuring out which files/changes should go
  first, second, etc.
  To verify this step, that you didn't miss any files during rebase/
  reset/cherry-picking:

```shell
current="$(git branch --show-current)"
# We will detect them from folder structure:
# .git/stacked-commits/ticket-xyz-1231/ + any vN folders inside
original=""
new=""
base=""
# ... lots of code for detecting base branches / commits
git diff --exit-code "$base..$original" "$base..$new"
```

If this shows any output, it means that your stacked-commits branch tree
is different from original. You decide whether you want to go forward
or not.

- Now, finally we get to nitty-gritty, and we get to call three things
  for every commit in our /v1 branch:

1. Update context.md -> a script/tool
2. Update why.md -> a script/tool
3. Amend commit message ->\
  takes commit title (based on context/why/files) + context.md + why.md,
  verifies that .md files are correct, links are not broken, etc, then
  amends commit
4. Human review step (non-negotiable for now!). Every commit should be
   reviewed by human and fully understood.

After all commit messages are processed, we should know that all of them
were reviewed by a human (because of step 4) If you skipped review and
still accepted it, that's on you. This MUST be recorded: exactly who was
responsible for this commit message and diff. Fortunately, `git` already
does this for us out of the box.

## Conclusion

A structured approach to git commit messages makes code review more
granular and feedback more precise. Larger companies already adopted so
called [stacked changes][ref-stacked-changes]
process to stay unblocked and ship faster. The moment you realize code
review is a bottleneck, the moment you should adopt it.

We should strive to have concise commits that are easy to read and
reason about.

- 50 commits with a single-line change are as bad as having one
  500-line commit.
- Keep it under 500 lines for a PR, one commit containing no more than
  50-100 lines.
- If you see that PR is getting large — create subtasks and split it
  into multiple branches, create one main branch that you merge PRs
  into.
- Target a single PR under 100 lines and observe how easy it is to
  review / comment / iterate on / deploy.

And that's about it for now, thank you for reading!

Feel free to add suggestions and comments, or message me directly on
Slack.

## Acknowledgments

Thanks to:

- John for suggestions, pairing on this topic, and providing feedback.
- Hikari for suggestion to make a Confluence document out of the
  original Slack thread, pairing on the topic and feedback.
- #trust team for testing out v1 of this document on agents, which
  helped me realize how lacking it was around context/why sections.

[toc-why-important]: #why-this-is-even-more-important-today
[toc-mvc]: #what-is-good-context-aka-minimal-viable-context-mvc
[ref-thoughtbot-commit]: <https://thoughtbot.com/blog/>
[ref-tbaggery-commit]: <https://tbaggery.com/2008/04/19/>
[ref-railsconf-review]: <https://www.youtube.com/watch?v=PJjmw9TRB7s>
[ref-stacked-changes]: <https://news.ycombinator.com/item?id=29255195>
