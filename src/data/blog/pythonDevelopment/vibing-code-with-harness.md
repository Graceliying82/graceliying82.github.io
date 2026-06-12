---
title: "Successfully Vibing Code with Harness"
author: Grace Li
pubDatetime: 2026-06-12T07:30:00Z
postSlug: vibing-code-with-harness
featured: true
draft: false
tags:
  - ai-development
  - software-engineering
  - collaboration
  - hacking
description: "Insights and lessons learned on vibe coding—effectively using AI to build complex systems while maintaining architectural integrity."
---

## Introduction

I’ve spent the last few months figuring out how to build complex systems with AI without losing my grip on the architecture. It’s a delicate balance: you want the speed of a high-level flow, but you need the rigour of a senior engineer to keep things from falling apart.

Through a variety of projects—from a fast-paced hackathon to numerous subsequent builds, both big and small—I've developed a set of specific rules that have turned AI-assisted development from a gamble into a reliable strategy. Here is what I’ve learned about building a proper harness for your coding flow.

## The Reality of Vibe Coding

When you start working with an AI agent, everything usually feels smooth for the first few hundred lines of code. But as the system expands and more moving parts are added, things start to break in predictable ways.

Here is what I’ve found to be the most critical challenges:

1.  **Scope is everything**: The second your scope gets too broad, the AI starts to lose the thread. Keeping tasks small and modular isn't just a best practice—it's the only way to stay sane.
2.  **Specific requirements win**: Vague prompts are the enemy. If you aren't precise about what you need, the AI will fill in the blanks with assumptions that probably won't match your design.
3.  **Collaboration is messy**: Trying to manage multi-agent workflows or collaborating with other humans in an AI-driven project is surprisingly difficult.

### Case Study: The DeepPulse Hackathon

I recently teamed up with Mert for a hackathon project called DeepPulse. If you're interested, you can check out the project here:

- **GitHub**: [DeepPulse Repository](https://github.com/graceliying82/DeepPulse)
- **Video Demo**: [Watch on YouTube](https://www.youtube.com/watch?v=Sf34_RCBBVw&feature=youtu.be)

<iframe width="100%" height="315" src="https://www.youtube.com/embed/Sf34_RCBBVw" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

We started by focusing strictly on Cardiac data, and it worked perfectly. But then we got ambitious and decided to add Respiratory and Neural signals. Suddenly, the complexity exploded. Since we were working in different time zones, we could hand off the work easily, but the merge conflicts were a nightmare. The AI struggled to grasp the new multi-signal architecture, and we spent way too much time fixing things it had broken.

## What I Noticed When Things Went Wrong

I’ve seen a few recurring patterns when AI agents hit a wall:

*   **The Redo Loop**: When an AI can't solve a specific problem after a few tries, it often defaults to "let's just rewrite everything." This is almost always the wrong move and usually introduces even more bugs.
*   **Architectural Amnesia**: AI agents have a short memory for design history. They often forget the original design philosophy, leading to inconsistent code that feels like it was written by ten different people.
*   **Cheating the Tests**: I've caught AI agents modifying test data or even the tests themselves just to get a "pass" result. It’s a shortcut that hides underlying issues, and it's something you have to watch for constantly.

## My Rules for Success

I’ve refined my approach to vibe coding into a set of rules that I now use for every project. These strategies have made a world of difference.

### 1. Radical Clarity
Be ruthless about your scope and requirements. If you can’t explain the task simply to a human colleague, you shouldn't expect an AI to get it right.

### 2. Set the Ground Rules
Before the first line of code is written, define the general principles. For example, I’ve started separating roles:
- **The Development Agent**: Focuses strictly on the implementation.
- **The Testing Agent**: Runs tests and produces reports. It only touches code to fix the tests themselves—it never touches the dev code just to make a test pass. 
- **The Golden Rule**: Don't lie. Never change code just to hide an issue.

### 3. Design First
Treat your AI like a senior developer, not just a code monkey. Before any coding starts, ask for:
- **High-level and Low-level designs**.
- **A Breakdown**: For anything large, have the AI break the work into manageable stages and stories first.

### 4. Lean on TDD
Define your test strategy—unit tests, integration tests, the whole works—before you start coding. This creates a harness that keeps the AI's implementation grounded and verifiable.

### 5. Multi-AI Reviews
Don't rely on just one model. I like to use Claude Code as my primary builder and have Gemini review the designs. This "second opinion" catches architectural flaws and edge cases that one agent alone might miss.

### 6. Staged Development and Logging
Write code in distinct stages. Test everything thoroughly before moving on. I also have the AI document its troubleshooting logs and test results into markdown files. This way, any future AI agent can read that history and understand the context before making changes.

## Final Thoughts

Vibe coding isn't an excuse to be lazy; it's a way to act as a high-level architect. When you provide the right structure—clear requirements, rigorous design, and a solid testing harness—you can build complex systems at an incredible pace without losing quality.

---
*Happy coding.*
