---
layout: post
title: The New AI Era - What Makes a Developer Valuable?
date: 2026-06-01 10:59:00+0100
description: AI is changing how software is built, but the skills that make a developer valuable are becoming broader, not less important.
tags:
  - artificial intelligence
  - software engineering
  - career
categories:
giscus_comments: false
related_posts: false
toc:
  beginning: true
---

<span style="margin-left: 10px;"></span>
There is a lot of anxiety around AI and software development. Some developers are worried that writing code is becoming 
less valuable. Some companies are treating AI as a shortcut to ship more features with fewer people. Others are adding 
an assistant to the editor and expecting the rest of the engineering process to take care of itself.

I understand the concern, but I do not think the future of development is about choosing between people and AI. I think 
it is about becoming better at the parts of engineering that require context, judgment, and responsibility, while using 
AI to remove more of the mechanical work.

That is a good trade.

AI can already generate a reasonable first draft, explain an unfamiliar API, suggest tests, translate code between 
languages, and help investigate a problem. Those are useful capabilities. But a first draft is not a production system, 
and a plausible answer is not the same thing as a correct one.

The developer's role is not disappearing. The role is moving up the stack.

## AI is a tool, not an engineering strategy

The most useful way I have found to think about AI is as an extremely fast, occasionally unreliable teammate. It can 
help me explore options, get past a blank page, and spend less time on repetitive work. It cannot own the consequences 
of a bad architectural decision, understand every constraint in a business domain, or be accountable when a change 
causes an incident.

That distinction matters.

The [2024 DORA research](https://dora.dev/research/2024/dora-report/2024-dora-accelerate-state-of-devops-report.pdf), based on responses from more than 39,000 technology professionals, found that AI 
adoption was associated with improvements in perceived productivity, code quality, documentation, and job satisfaction. 
It also found a more uncomfortable result: delivery performance and stability could decline as AI adoption increased. 
The report's conclusion is worth keeping in mind: AI can improve parts of the development process without automatically 
improving the software delivery system.

> “AI does not appear to be a panacea.” — DORA, *Accelerate State of DevOps Report 2024*

That is not an argument against using AI. It is an argument for using it with engineering discipline. If a team starts 
producing larger changes, skips review, or trusts generated tests that do not test the right behavior, faster code 
generation simply makes the problems arrive sooner.

The same caution appears in the [METR randomized study of experienced open-source developers](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/). In that early-2025 
study, 16 developers completed real tasks in repositories they already knew well. When AI tools were allowed, they took 
19% longer on average, despite expecting to be faster.

This is only one study, in a specific setting, and AI tools continue to change quickly. Still, it makes an important 
point: productivity is not the same as typing speed. The time spent reviewing, correcting, integrating, and validating 
generated code counts too.

## Technical skills still matter — probably more than before

I do not buy the idea that developers no longer need to understand code because an AI system can produce it. In fact, 
understanding the fundamentals becomes more important when it is easy to generate a lot of code that looks convincing.

A developer who understands concurrency can spot a race condition in generated code. A developer who understands 
databases can recognize an inefficient query, an unsafe migration, or an isolation problem. A developer who 
understands distributed systems can question a design that quietly assumes reliable networks, exactly-once delivery, 
or perfect consistency.

The same applies to security, testing, observability, performance, and operations. AI can suggest an implementation, 
but technical knowledge is what lets me evaluate whether that implementation belongs in the system.

The fundamentals are not being replaced. They are becoming a quality filter.

This is also why I would be careful about defining an AI-capable developer as someone who knows a collection of prompts. 
Prompting can help, but it is only one small part of the workflow. The real advantage comes from knowing what to ask, 
supplying the right context, recognizing a weak answer, and turning a suggestion into a maintainable change.

## The skills that become more valuable

If code generation becomes cheaper, the scarce skills are the ones that help us decide what should be built, why it 
should be built, and how to know that it works.

### 1. Problem framing

Before asking AI to implement something, I need to understand the problem. What is the user actually trying to 
accomplish? What constraint matters most? What does success look like? What should not change?

An unclear request produces an unclear solution, no matter how impressive the model is. Developers who can turn a v
ague request into a precise problem, a small plan, and measurable acceptance criteria will create much more value than 
developers who can only generate code quickly.

### 2. System thinking

A feature is never just a function. It affects data, APIs, queues, deployments, permissions, monitoring, support, and 
the people who will maintain it six months from now.

AI is very good at operating inside the local context I give it. It is much less reliable at discovering all of the 
context I forgot to mention. That is where architecture and system thinking matter: understanding boundaries, 
dependencies, failure modes, and trade-offs across the whole system.

### 3. Verification and healthy skepticism

The ability to review code critically is becoming a core development skill. I want to know:

- What assumption is this code making?
- Which behavior is covered by a test, and which behavior is not?
- What happens when the dependency is unavailable?
- Is this safe with real input, real traffic, and real data?
- How will we observe and roll back the change?

This is not about distrusting every AI suggestion. It is about putting trust in the right place. I am happy to accept 
generated code when I can explain it, test it, and support it in production.

### 4. Communication

Software is built with other people. Clear design notes, useful pull requests, good questions, and calm incident 
communication are not secondary skills. They are part of the engineering work.

As AI makes individual implementation faster, coordination can become the bottleneck. A developer who can explain 
trade-offs, align with product and operations, and make a decision visible to the team becomes more valuable, not less.

### 5. Product and domain understanding

The best technical solution is not always the best product decision. Knowing the domain helps me identify the 
important edge cases, challenge unnecessary complexity, and choose a solution that creates value instead of just 
adding functionality.

AI can help me learn a domain faster by summarizing documentation or generating questions. But it cannot replace 
conversations with users, operators, and subject-matter experts. Context is still earned.

### 6. Adaptability and continuous learning

The tools will keep changing. Editors, models, agents, frameworks, and workflows will evolve faster than most of us 
can predict. Trying to memorize one tool's interface is less useful than building the habit of learning, experimenting, 
and measuring whether a new approach actually improves the work.

The [World Economic Forum's *Future of Jobs Report 2025*](https://www.weforum.org/publications/the-future-of-jobs-report-2025/in-full/3-skills-outlook/) makes a similar observation from a broader labor-market 
perspective: analytical thinking remains the top core skill identified by employers, while creative thinking, resilience, 
flexibility, technological literacy, and curiosity are all expected to grow in importance. That combination feels very 
relevant to software engineering today. The future is not purely technical or purely interpersonal. It is both.

## How I want to use AI

For me, responsible AI-assisted development looks something like this:

1. Understand the requirement and the surrounding system before generating an implementation.
2. Ask AI to explore alternatives, identify risks, or create a first draft when that is useful.
3. Keep the change small enough to review properly.
4. Run tests, inspect the diff, and add tests for the behavior that actually matters.
5. Use profiling, logs, metrics, and real feedback instead of assuming that a confident answer is a correct answer.
6. Keep ownership of the result.

The last point is the most important. If I merge code, I am responsible for understanding it and supporting it. “The 
model wrote it” is not an explanation that helps during an incident.

## Becoming a better developer, not just a faster typist

The interesting opportunity in this new era is that AI can remove some of the friction that has always been part of 
development. It can help us get through boilerplate, explore unfamiliar code, write documentation, and learn new 
technologies. That gives us more room to focus on design, reliability, users, and the hard questions.

But that benefit only appears if we use the time well. Generating more code is not the goal. Delivering useful, 
understandable, secure, and reliable software is the goal.

So I am not trying to compete with AI at producing lines of code. I am trying to become the kind of developer who can 
use it well: technically grounded enough to evaluate its output, curious enough to learn from it, skeptical enough to 
verify it, and experienced enough to know when a simpler solution is better.

Technical skills still matter. They give us the foundation to make good decisions. But in a world where implementation 
is increasingly assisted, the developers who stand out will also be the ones who can frame problems, communicate clearly, 
understand the business, reason about systems, and take responsibility for the outcome.

That is the direction I want to move in. AI is not the end of software engineering. Used well, it is another tool that 
can help us practice it at a higher level.

## Sources

- [DORA — Accelerate State of DevOps Report 2024](https://dora.dev/research/2024/dora-report/2024-dora-accelerate-state-of-devops-report.pdf)
- [METR — Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/)
- [World Economic Forum — Future of Jobs Report 2025: Skills Outlook](https://www.weforum.org/publications/the-future-of-jobs-report-2025/in-full/3-skills-outlook/)
