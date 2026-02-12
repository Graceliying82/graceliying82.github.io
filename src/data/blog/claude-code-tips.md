---
title: How to use Claude Code without breaking the bank (or hitting rate limits)
author: Grace Li
pubDatetime: 2026-02-12T07:00:00Z
postSlug: claude-code-tips
featured: false
draft: false
tags:
  - ai
  - productivity
  - coding
  - claude
description: I found a workflow to make my AI token budget much more efficient. Here are 4 tips to save money and stay within limits.
---

I’ve been testing **Claude Code** for a few weeks, comparing it directly with Gemini Pro.
Claude Code is absolutely a winner, especially when solving difficult coding problems where other models might struggle.

However, Google is way more budget-friendly. Claude can easily burn through your token budget and get throttled quickly if you aren't careful.

After some research and testing, I found a workflow to make my AI token budget much more efficient. Here are 4 tips to save money and stay within limits:

## 1. Leverage CLAUDE.md for Context

**Don't waste tokens explaining your project from scratch every time.**

Create a `CLAUDE.md` file in your root directory with your project's basic description, architecture, and coding standards. Claude reads this automatically, giving it immediate context without the extra token cost.

## 2. Set a Budget-Friendly Default Model

By default, the tool often uses the most expensive model (Opus). In your `.claude/settings.json`, set a more efficient model (like Sonnet 3.5) as your default. It’s significantly faster and cheaper for 90% of daily tasks.

## 3. Implement "Model Escalation" Rules

I added a custom rule to my `CLAUDE.md`:

> "If you cannot solve the request after 2 attempts, explicitly ask to switch to the Opus model."

This ensures I only use the expensive "heavy lifter" when the efficient model actually fails.

## 4. Keep Conversations Short

Long conversations bloat the context window, increasing costs exponentially. I instructed the AI to remind me to start a new chat (or use the `/clear` command) when moving to a new topic. This prevents carrying unnecessary history into new tasks.
