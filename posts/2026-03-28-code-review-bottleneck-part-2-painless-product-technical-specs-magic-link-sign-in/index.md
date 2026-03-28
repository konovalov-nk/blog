---
title: "Solving the Code Review Bottleneck, Part 2: Painless Product/Technical Specs, 2026 Edition — 'Magic Link Sign-In'"
date: 2026-03-28
draft: true
tags: [product-specs, technical-specs, magic-link-sign-in, software-engineering]
description: "Decompose your product/technical specs into logical, single responsibility, atomic parts. And everything can be described as a graph. And CI/CD will take care of the rest."
---
# Painless Product/Technical Specs, 2026 Edition — "Magic Link Sign-In"

## Problem we (still) have today

We've all been there.

You're working on a new feature.
Even better -- a feature that was a "quick MVP" from 2014 that was never finished.
Code from 2014 that made it into 2026 codebase, unchanged, no documentation, no spec.
All the technical debt that was built on top of it.
When you try `git blame`, all you see is garbage and noise:

```text
* linting (2019)
* refactoring (2018)
* very important bug fix (2016, bug tracker: TICKET-123)
* fix
* migrating entire codebase to new linting rules (2015)
* implementation (2014, bug tracker: TICKET-67)
```

Here's one more thing to add on top: bug tracker tickets mentioned in
commit messages from 2014-2016 are long gone. Because, well, if you have
a bug tracker, you don't need proper commit messages, right? Just write
a ticket number and you're done.

I covered this specific problem in [Stacked Commits][stacked-commits]
post.

Anyway, it was decided that it was not worth it investing into importing
old tickets into a new system. And so now your history is gone.
And people who were involved in the feature are long gone.

Well, not dead but good luck finding them and asking a question from 10+
year old git commit message. And receiving an answer that would be
useful.

And so, here are all first questions you have to answer:

- What was the actual spec and requirements? Was the code even working?
- What happened in 2016?
- Who was the developer who filed the bug? What was even the bug?
- What did they found? What did they do?
- What was the root problem they were trying to solve in first place?

Only thing you are left with is `git diff` and useless commit messages.

Your customers are angry: "This feature isn't working, go figure, fast".
And you're the one who has to come up with a solution.

### Two options

The very first option is to start from scratch and write a new spec and
a new implementation. That's what everyone does when they get assigned
a Jira customer support ticket, right? Right?

Wrong.

From my 12 years of experience, I never saw another engineer to dig the
codebase and "reverse-engineer the original spec and requirements".
That's what I call it:

> Reverse-engineering product specs and requirements is the process of
> understanding the original problem developers were trying to solve,
> using only existing codebase and limited context.

And as you can imagine, this is a very difficult task. So it is not that
surprising that most engineers just give up and make a hack/workaround

If you are lucky, a hack can be "aligned" to the original spec. But I've
seen plenty of cases where developers come up with a fix to a solution
that was never intended to be used in the first place.

I even seen commits that say "temporary fix". Made 3 years ago!
If anything, [there's nothing more permanent than a temporary solution][temporary-solution].

It was only MVP to test the idea. And you are spending time fixing wrong
assumptions made in 2014, instead of solving the actual 2026 problem.

## Solution from 2000: Product/Functional Spec

First of all, I suggest to go read Joel Spolsky's classic article on "Painless Functional Specifications".

There are four chapters:

- [Why Bother?][joel-pfs-part-1]
- [What's a Spec?][joel-pfs-part-2]
- [But How?][joel-pfs-part-3]
- [Tips][joel-pfs-part-4]

Did you read it?

No you didn't. Go [read it now][joel-pfs-part-1] and then come back,
so we can talk more about what a good spec should and shouldn't have in
it. I'll wait here for you. Thanks.

(waiting patiently...)

Ah, good. You're back.

So, I would not duplicate Joel's work here. I just want to expand on it
a bit and add a few more details, that matter today, in 2026.

What I think is crazy, is that nothing changed at all since 2000. We are
still making same mistakes Joel mentioned in his article. Still writing
terrible specs and requirements. Or not writing them at all. Terrible
mistake that costs us a lot of time and money.

And I'm pretty sure if we dig deeper, Joel isn't first one to notice
this. But for some unexplainable reason, I managed to stumble upon it,
back in 2015 or so. I can't even remember how I found it. And 10 years
later, we're nowhere close to fixing this problem. And so my mission
is very clear: to fix this problem once and for all.

This following is an example spec Joel wrote back in 2000.

> [What Time Is It?][joel-spec-example]

And here are few things I want to highlight:

- This spec is huge and overwhelming, with lots of details, even though
  it's high-level, you can easily get lost on what matters to you.
  It is good as an example what to write about, but structure is not
  optimal for modern development practices.

- The spec mentions `mind-numbing details will be written elsewhere`,
  which I agree with. But the problem is how do we connect these product
  and technical specs together?

  This is the key problem we want to solve.

- Scenarios and users are important, but you don't want to write them
  from scratch every single time, as Joel mentioned in the article.
  And while flows can change over time, at least you should be able to
  keep a fixed set of personas and connect them to scenarios.

  Persona is a great way to capture what kind of problems a user is
  trying to solve. And one persona can have multiple scenarios.

  In your product spec, you should link scenarios to personas, so people
  can understand who are we solving problems for. This is the cheapest
  way to connect your developers to your customers.

  And it is also how you can build empathy.

- What happens when your spec grows so much, that problems can arise
  between seemingly unrelated parts of the spec? How do you connect?

  One spec says, `user can sign in only via Magic Link`. Another spec
  says, `users should be able to reset their password`, assuming they
  have ability to manage passwords. But the problem is Magic Link is
  passwordless.

  So you just created a spec that is impossible to implement.

  As you can see, specs can be connected to each other. And we should
  design a system that allows us to do it. And whenever we change any
  spec, we should be able to tell: `this spec is connected to that one, go figure out if it is still valid`.

- Spec drift is a thing. And no, not just code. Entire countries
  disappear (ever heard of Yugoslavia?). Stars disappear. Galaxies cease
  to exist. And yet you think your spec is an eternal entity, like
  New York traffic at 5pm on a Friday.

So, here is what I think we should do.

## Example: “Magic Link Sign-In” as a structured spec (template)

This is example of a spec that I want to be used as a template. It's not
year 2000, and we have progressed a lot since then. So I'm adding few
more interesting details to it.

The spec itself should be as tiny as possible. However, there is certain
set of things that should be included in every spec.

- **SCIP index** (unique identifier)

- **SCIP links** to codebase artifacts: controllers, models, views, etc.
  This is our bread and butter: we connect code to spec.
  If a feature is spread across multiple repositories, you want to link
  to all of them, but files linked should be scoped only to the spec in
  question.

- **Feature name**

  Naming is always a hard programming problem. Poorly named features
  create confusion and pain for everyone.

  One specific problem I can already foresee, is when system evolves,
  we need to clearly distinguish between levels of abstraction. You
  don't want to mix features from different levels of abstraction into
  one.

  And a simple example to solve this, is to divide into three levels:

  - Product Level
  - Technical Level
  - Infrastructure Level

  **Product Level** is the highest level of abstraction. It is the problem
  we are solving. It is the full user story, end-to-end the problem we
  defined, and the solution we are providing. And how it works on high
  level.

  **Technical Level** is for example, parts of the that flow:

  - Authentication
  - Token generation
  - Sending email

  These are very important to capture, because even simplest email
  delivery can be complex: SPF, DKIM, DMARC, bounce/complaint webhooks,
  warm-up plan, monitoring for blocklists, etc. And this level also can
  have scenarios and user stories, as well as acceptance criteria,
  alerts, SLOs, etc.

  I'm also certain that this level can be either omit or divided into
  yet another level, which is the **Infrastructure Level**.

  **Infrastructure Level** is the lowest level of abstraction. It is the
  nitty-gritty. Think RFC spec of email protocol. How exactly email is
  delivered? How exactly Redis is used to store tokens? What happens
  during failover? How exactly SES is used to send emails?

  And if we fail to capture this level, or update it when it drifts due
  to new requirements, we will have a big problem that will affect the
  product level. And that is also a key insight.

  If you don't care about this level, then it means your product level
  is not business-critical. And my first question to you: why is it even
  deployed? Why are you spending money on it? And there might be valid
  answers to this: `we are making POC`, `not enough budget`,
  `this is a low-risk feature`. But most importantly, you must
  internalize this, so your developers know what to expect, and how much
  effort they want to spend on verifying it.

  And so, I think what we need, is

- **Specification Level**

  It is a simple enum that captures level of abstraction of the spec.

  Good examples are:

  - `product` -> most stakeholders are on this level
  - `technical` -> developers/owners/architects are on this level
  - `infrastructure` -> devs/ops are on this level

  I'm unsure if we need hierarchy of levels, but I think it's a good
  first approximation.

- **Scenarios** / **User Stories** + SCIP links to codebase artifacts

  Ideally, these are code-first. We need to make sure that these are
  readable by your product managers and non-technical stakeholders.
  However you don't want to test them manually, so we need to formalise
  them as set of E2E tests.

  So when we type a human-readable story, we should be able to link to
  the codebase artifacts that implement it. And this is key, because we
  can immediately dive into the codebase and verify if it is correct.

  This is where drift can happen. And so it's crucial to link the very
  ambiguous parts of the spec to the codebase artifacts that implement
  it.

- **Owner(s)**

  Owner(s) are the people who are responsible for the feature. They are
  people who know context, understand the problem, and have a vision or
  a roadmap. If you lack owners, nobody is responsible/accountable, and
  this creates ambiguity and confusion.

- **Documentation links/artifacts/context**: PFS, TFS

  This is essentially an answer to questions:

  - `What do I need to know to understand this spec?`
  - `How can I make myself smarter around this domain?`
  - `What is the minimal context I need to grasp the problem?`
  - `What should I know to review code that touches this spec?`

  And there are two types of documentation:

  - Product Feature Spec (PFS)
  - Technical Feature Spec (TFS)

  PFS is for product managers and non-technical stakeholders. It is the
  user-view of the feature. It is the "what/why" of the feature. As high
  level as possible, without skimming on critical details.

  And yes, for Technical/Infrastructure levels, you want to have PFS as
  well but usually you don't expect non-technical stakeholders to read
  it. Let's say, in this case PFS is like a layman explanation of the
  feature.

  TFS is for developers and architects. It is the "how" of the feature.
  It is the technical implementation details, RFCs, nitty-gritty.

  One important remark about TFS, is that you don't want to write it as
  a single huge blob of text. Usually this is a sign that your spec is
  too large, and you want to split it into multiple smaller specs, that
  are more reasonable to read and understand.

  For example, email sending can be split into multiple smaller specs:

  - DKIM
  - bounce/complaint webhooks
  - email gateway integration (SES, SendGrid, etc.)

- **Create/Modify dates**

  Any change to the spec, including code, should update these dates.
  This is how we can track the freshness of the spec.

- **Status**: 'draft', 'review', 'approved', 'published', 'deprecated'

  The list is not exhaustive. You can add more as needed.

- **Acceptance Criteria**, formalised as measurable goals, for example:

  - p95 E2E under **150 ms** for API;
  - deliver email in **< 5 s** median;
  - Availability **99.95%** monthly;
  - Error rate alert at **0.1%** in 15-min SLO window.

  I'm thinking this could be Prometheus metrics/SLOs, something that we
  can query. Because if you can't measure it, you can't manage it.

- **Alerts and thresholds**: can be links to DataDog/Prometheus/Terraform

  These are very important to capture, because every feature we build is
  usually a part of a larger system. And alerts also have levels of
  abstraction and severity.

  Technical/Infrastructure level alerts can capture for example:

  - `Email gateway is down`.
  - `API response time is > 2 seconds`.

  Product level alerts can capture for example:

  - `X% of users are not able to sign in`
  - `Median end to end latency between clicking on the link and being
    signed in is too high: > 10 seconds`

  And surely, lower-level alerts will trigger higher-level alerts. And
  this is by design. **So your engineers that updated spec can
  immediately track down what kind of high-level business critical
  impact this is going to have. And so your code reviews immediately
  become more meaningful and valuable.**

---

Now that we have fields of a spec, we can make an example spec for our
`Magic Link Sign-In` feature.

[joel-pfs-part-1]: <https://www.joelonsoftware.com/2000/10/02/painless-functional-specifications-part-1-why-bother/>
[joel-pfs-part-2]: <https://www.joelonsoftware.com/2000/10/03/painless-functional-specifications-part-2-whats-a-spec/>
[joel-pfs-part-3]: <https://www.joelonsoftware.com/2000/10/04/painless-functional-specifications-part-3-but-how/>
[joel-pfs-part-4]: <https://www.joelonsoftware.com/2000/10/15/painless-functional-specifications-part-4-tips/>
[joel-spec-example]: <https://www.joelonsoftware.com/whattimeisit/>
[stacked-commits]: <https://blog.br11k.dev/2026-03-23-code-review-bottleneck-part-1-stacked-commits>
[temporary-solution]: <https://en.wiktionary.org/wiki/there_is_nothing_more_permanent_than_a_temporary_solution>
