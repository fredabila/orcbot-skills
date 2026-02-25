---
name: agi-reasoning-engine
description: Advanced reasoning and problem-solving framework for complex, multi-step tasks. Use when a task requires deep planning, cross-domain knowledge synthesis, or high-reliability execution with self-correction.
---

# AGI Reasoning Engine

This skill elevates your reasoning capabilities by enforcing a structured approach to complex problem-solving. It prioritizes first-principles thinking, rigorous verification, and adaptive strategy.

## Core Directives

### 1. Multi-Dimensional Planning
For any non-trivial task, do not jump to execution. Instead:
- **Deconstruct**: Break the objective into atomic sub-tasks.
- **Identify Dependencies**: Map the logical flow and critical paths.
- **Edge Case Analysis**: Proactively identify potential failure modes.

### 2. First-Principles Thinking
When faced with a roadblock:
- Strip away assumptions.
- Re-examine the fundamental constraints.
- Rebuild the solution from the most basic truths known about the system.

### 3. Iterative Verification Loop
Every significant action must follow the **Act -> Verify -> Refine** cycle:
- **Act**: Perform the operation.
- **Verify**: Use empirical evidence (tests, logs, outputs) to confirm success.
- **Refine**: If the result deviates from the expected outcome, diagnose the root cause and adjust the strategy.

### 4. Cross-Domain Synthesis
Leverage knowledge across multiple technical domains (e.g., combining systems architecture with security best practices) to ensure holistic solutions.

## Reasoning Patterns

Refer to [references/reasoning-patterns.md](references/reasoning-patterns.md) for specific mental models and heuristics to apply during execution.

## Triggering Self-Correction
If you detect a contradiction or a logical gap in your current approach, stop immediately. State the contradiction clearly and re-evaluate your strategy using the First-Principles directive.
