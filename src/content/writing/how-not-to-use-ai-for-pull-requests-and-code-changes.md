---
title: How (not) to use AI for pull requests and code reviews
description: We don't need to review all code changes. Make it easy for reviewers to focus on what's important.
publishedDate: 2026-08-13
tags: [AI]
draft: false
---

The number of pull requests most teams produce has drastically increased because of the help of coding agents. 
How do we review all those pull requests?

When we look at a PR description created with the help of an agent, it’s typically a summary of all the things changed. 
It’s complete, but very long.
Even understanding what the pull request is about takes a lot of time.
Should we spend our days reading these long summaries, all the code (AI produced or not), line by line, to make sure it meets our expectations? 
Or could we be smarter about what we spend our time on?

The first question to answer is who is ultimately responsible for the pull request. 
That should be the developer creating the PR!
It doesn’t matter if and how an agent was used in the process. 
The developer must have an understanding of the change, the impact of it, risks it carries, etc. 
They don’t have to know each and every line written by an agent, but they do need to know what public APIs are changed or introduced, if there are behavioral changes, gaps, etc. 
They must also do an appropriate amount of testing and verification, and it’s perfectly fine to use coding agents for this. 
Coding agents are really good at discovering bugs, this is not where we need to spend our team's time.

There must be enough trust in a team that everyone does their due diligence. 
Code reviews are not supposed to be a gate to check if someone did their job.

The second question is what the purpose of code reviews is. 
As discussed, the developer responsible for the PR should already do the work to ensure the quality of the code. 
Thought about the design, implications, the code itself, verified behavior etc.
Coding agents are our peers in this process to help you catch that bug that in the past we needed a code review for.
When we review code with our agents, before even creating a PR, there is little point in having others in the team to review this code.
Do we really expect to find glaring bugs in code already verified by the author and his coding agents?
Instead of making the author of the pull request responsible for reaching this level of quality, can't we just have a review agent run once the pull request is created?
In my opinion, this just adds delay and causes context switching.
You have a coding agent right at your fingertips, ready to help you improve your pull request before submitting it. Use it!
I'm not saying it's bad to have a review agent in your PR process.
However, I do think it's bad to submit PRs that are clearly not ready (unless explicitly called out) and just rely on the review agent to help you out.
Even more so if your team also get notified of the PR, and might start reviewing already.

We *should* involve our team in decision-making about the things that really matter, and things that we are unsure of.
The pull request description should clearly call out what you want your team to give feedback on.
Some examples:

* "I introduced this new public interface X. Any suggestions for a better name? I considered Y and Z as well".
* "The default value of this property has flipped, which should be safe because we already migrated X. Do you see any possible failure scenarios that I might have missed?"
* "I'm introducing this new feature. It requires the user to do X first. Do you see any way to automate that step?"

Now it's clear for the reviewer what to focus on, and what feedback is requested.
The reviewer might not even have thought about commenting on it was buried in less important changes that take away attention.

Many pull requests won't have anything in particular we need help with.
What do we do with such PRs?
At the very least; it should have a description that indicates in a single sentence what and why changed, and that it doesn't require further review.
If there is enough trust in the team, and the system has sufficient pre-merge and deploy verification, just merge these without further review.
Not ready for that level of trust? With clear PR descriptions, we at least make reviews easier.
And yes, you should probably [write those descriptions by hand](no-ai-for-writing.md) :-)

It can be very useful to have tools and agents detect how risky a change is.
For example, we could use tools that statically verify if there are public API changes, which should always get extra attention. 
Use tools and agents to help focus on what really matters, so that we, the humans, can spend time thinking about the interesting stuff!