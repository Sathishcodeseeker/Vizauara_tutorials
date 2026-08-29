# Steps to Build an Agent Harness

An agent harness is the runtime surrounding a large language model (LLM). The LLM supplies learned intelligence, while the harness supplies context, tools, memory, permissions, execution control, verification, and the user interface.

```text
Training data -> LLM (brain) -> Agent harness (runtime) -> Tools and environment
```

## Part 1: Build or Select the LLM

Most teams use an existing model through an API or deploy an open-weight model. Building a frontier model from scratch requires substantial data, infrastructure, and computing resources.

### Base-model development

1. Define the use case and target capabilities.
2. Create capability, quality, and safety evaluations.
3. Collect and license training data.
4. Clean, filter, deduplicate, and classify the data.
5. Remove sensitive, unsafe, and low-quality content.
6. Choose the data mixture and sampling weights.
7. Train or select a tokenizer.
8. Design the model architecture.
9. Determine the model size and scaling strategy.
10. Build the distributed training infrastructure.
11. Pre-train the base model using next-token prediction.
12. Perform continued pre-training or domain adaptation.
13. Mid-train capabilities such as long-context reasoning and tool use.

### Post-training and alignment

14. Perform supervised fine-tuning (SFT) using high-quality demonstrations.
15. Collect human preference rankings.
16. Train a reward model using those preferences.
17. Optimize the LLM with reinforcement learning, such as PPO.

Steps 15-17 form the classic **reinforcement learning from human feedback (RLHF)** pipeline.

Depending on the project, RLHF may be replaced by or combined with:

- Direct preference optimization (DPO).
- Reinforcement learning from AI feedback (RLAIF).
- Reinforcement fine-tuning (RFT) using verifiable rewards.
- Constitutional or rule-based alignment.
- Distillation from a stronger model.

### Evaluation and deployment

18. Perform safety tuning and refusal training.
19. Red-team the model.
20. Run capability, safety, and regression evaluations.
21. Optimize inference using techniques such as quantization and caching.
22. Deploy the model-serving infrastructure.
23. Monitor quality, safety, latency, and cost.
24. Collect feedback and repeat the training cycle.

```text
Data
  -> Tokenizer and architecture
  -> Pre-training
  -> Continued or mid-training
  -> Supervised fine-tuning
  -> Preference data
  -> Reward model
  -> RLHF, DPO, RLAIF, or RFT
  -> Safety and evaluations
  -> Optimization and deployment
```

## Part 2: Build the Agent Harness

Coding agents such as Claude Code and Codex combine an LLM with repository discovery, tool use, an iterative agent loop, safety controls, and verification.

### 1. Define the agent's job

1. Identify the use cases the agent must support.
2. Define what successful completion means.
3. Define stopping and escalation conditions.
4. Decide which actions the agent may perform autonomously.
5. Identify actions that require human approval.

### 2. Select the model

6. Evaluate candidate models for reasoning and coding ability.
7. Compare their context limits, latency, reliability, and cost.
8. Confirm that the selected model supports the required tool-calling interface.
9. Add model routing or fallback models if necessary.

### 3. Design the instruction hierarchy

10. Create system-level safety and behavioral instructions.
11. Add product- or organization-level policies.
12. Load project instructions from files such as `AGENTS.md`.
13. Combine those instructions with the user's request without losing priority boundaries.

### 4. Build context discovery and retrieval

14. Discover the repository structure and development environment.
15. Search for relevant files instead of loading the entire repository.
16. Retrieve documentation, code, logs, and prior decisions as needed.
17. Rank and limit retrieved content to fit the model's context window.
18. Defend against untrusted instructions embedded in retrieved content.

Retrieval-augmented generation (RAG) belongs primarily to this runtime layer; it does not retrain the underlying model.

### 5. Define the tools

19. Create strict input and output schemas for every tool.
20. Add tools for reading files and searching text.
21. Add a controlled patch-based file-editing tool.
22. Add shell, build, test, lint, and formatting tools.
23. Add Git and code-review tools.
24. Add web, database, API, or Model Context Protocol (MCP) integrations when required.
25. Clearly describe each tool's permissions, side effects, and error responses.

### 6. Implement the agent loop

26. Give the model the current task and relevant context.
27. Ask the model to choose the next response or tool action.
28. Validate the proposed action against policy.
29. Execute an approved tool action.
30. Return the observation to the model.
31. Let the model adjust its approach based on the result.
32. Repeat until the task succeeds, needs escalation, or reaches a stopping limit.

```text
Understand -> Plan -> Act -> Observe -> Adjust -> Verify -> Finish
                         ^                 |
                         +-----------------+
```

### 7. Add execution safety

33. Isolate commands in an operating-system or container sandbox.
34. Restrict filesystem access to explicitly allowed locations.
35. Restrict network access by default.
36. Validate commands, paths, URLs, and tool arguments.
37. Require approval for destructive, privileged, costly, or external actions.
38. Protect credentials and secrets from prompts, logs, and tool output.
39. Add defenses against prompt injection and malicious repository content.

### 8. Manage state and memory

40. Store conversation and task state across turns.
41. Track completed and pending work.
42. Compact or summarize old context as the session grows.
43. Preserve important decisions, constraints, and evidence during compaction.
44. Add persistent memory only when it has a clear purpose and deletion policy.
45. Prevent the agent from repeating completed actions.

### 9. Handle errors and long-running work

46. Classify transient, permanent, and permission-related failures.
47. Retry transient failures with strict limits.
48. Let the model choose a different approach after a failed attempt.
49. Add limits for turns, tokens, time, tool calls, and cost.
50. Checkpoint long-running work so that it can resume safely.
51. Escalate to the user when progress requires missing information or authority.

### 10. Add planning and optional coordination

52. Create explicit plans for complex, multi-step tasks.
53. Keep only one directly dependent step active at a time.
54. Run independent tool calls concurrently when safe.
55. Optionally delegate independent, bounded work to sub-agents.
56. Merge delegated results without duplicating work or losing accountability.

### 11. Verify the work

57. Inspect the final diff and affected files.
58. Run relevant unit, integration, lint, type, and build checks.
59. Verify user-visible behavior rather than relying only on compilation.
60. Confirm that no unrelated files were modified.
61. Require evidence before the agent reports successful completion.

### 12. Build the user experience

62. Provide a terminal, IDE, desktop, web, or API interface.
63. Stream concise progress updates and tool activity.
64. Display approval requests with the exact proposed action and impact.
65. Present patches, test results, citations, and failures clearly.
66. Return a concise final summary with verification results and remaining risks.

### 13. Add observability

67. Trace model requests, tool calls, latency, retries, and failures.
68. Record token and infrastructure costs.
69. Redact credentials and sensitive user data from telemetry.
70. Make individual agent runs reproducible enough to diagnose failures.

### 14. Evaluate the complete agent

71. Build a suite of realistic end-to-end tasks.
72. Include successful, ambiguous, adversarial, and failure scenarios.
73. Test prompt injection, unsafe tool use, and destructive-action prevention.
74. Measure task correctness, test-pass rate, latency, cost, and completion rate.
75. Compare model and harness changes against the same evaluation suite.
76. Convert production failures into regression tests.

### 15. Deploy and improve

77. Roll out the harness gradually.
78. Monitor safety, reliability, latency, and cost in production.
79. Investigate failed or abandoned tasks.
80. Improve instructions, retrieval, tools, policies, and evaluations.
81. Upgrade or post-train the underlying model when harness improvements are insufficient.

## Final Distinction

```text
LLM
  = Intelligence learned during training

Agent harness
  = Instructions
  + Context and retrieval
  + Tools
  + Agent loop
  + State and memory
  + Sandboxing and approvals
  + Error handling
  + Verification
  + Observability
  + User interface
```

RLHF changes how the model behaves. The harness determines what the model can observe, execute, verify, and accomplish in a real environment.
