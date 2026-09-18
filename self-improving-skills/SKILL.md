---
name: "self-improving-skills"
description: "Use when creating self-improving agent skills."
---

# Self-Improving Skills

## Paper: arxiv 2602.06043
**Self-Improvements in Modern Agentic Systems** — 2026 survey

## Six Self-Improvement Mechanisms

### 1. Self-Reflection and In-Loop Feedback
- Reflexion: agent solves task, sees failure, writes critique, retries
- Self-Rewarding Language Models: model scores own outputs as reward signal
- Self-Consistency: multiple reasoning chains, pick majority answer
- On HumanEval: verbal RL bumped pass@1 from GPT-4 baseline to ~91%

### 2. Self-Generated Data and Curricula
- Agents create the data they learn from
- Difficulty-aware sampling (Never Give Up paper)
- Adaptive compute allocation: spend more on hard problems

### 3. Self-Adapting Models
- Agents fine-tune or edit themselves
- LoRA updates from execution traces
- Weight-space self-modification

### 4. Self-Improving Code Agents
- Agents modify their own code, policies, or architecture
- EvoSkill: iterative skill refinement from failure traces
- GEPA: gradient-free policy adaptation
- SkillWeaver: closing the loop with analyze-propose-evolve

### 5. Embodied Self-Improvement
- Agents learning by acting in environments
- World models and simulators for safe exploration

### 6. Verification, Safety, and Control
- Keeping self-improvement from going off the rails
- Skill Trust and Lifecycle Governance Framework
- Four-tier gate-based permission model

## Self-Improving Skill Architecture

### Component: Skill Generator
- Iteratively refines skills from execution traces
- Maintains persistent cross-branch context
- Produces skill bundles: SKILL.md + scripts + references

### Component: Surrogate Verifier
- Co-evolves with skill generator
- Provides informative feedback without ground-truth access
- Distinguishes real improvements from overfitting

### Component: Meta-Skill Evolver
- Two-timescale framework
- Fast timescale: task skill improvement
- Slow timescale: meta-skill evolution (how to improve)
- Five components: Analyzer, Retriever, Allocator, Proposer, Evolver

## Never Give Up (NGU) — Adaptive Sampling

### Key Insight
RL shows Matthew Effect: easy problems improve, hard problems don't.
NGU adaptively reallocates sampling compute to hard problems.

### How It Works
- Instead of generating same completions for every prompt
- Stop spending computation on solved prompts
- Probabilistically continue sampling unsolved prompts
- A prompt's treatment can change during training

### Results
- Deepscaler math: improves performance per compute on harder problems
- Manufactoria coding: iteratively solves harder tests until full solve

## Applying to Agent Skills

### Current Agent Skill Lifecycle
1. User requests skill
2. Agent loads skill from disk
3. Agent follows skill instructions
4. Agent completes task
5. Skill unchanged

### Self-Improving Skill Lifecycle
1. Agent loads skill from disk
2. Agent follows skill instructions
3. Agent executes task, records traces
4. Agent analyzes failure traces
5. Agent proposes skill edits
6. Agent applies edits to on-disk skill files
7. Agent verifies result is coherent
8. Skill evolved

### Skill Evolution Triggers
- Task failure: analyze trace, propose fix
- Task success: optional refinement
- User feedback: explicit correction
- Performance regression: rollback

### Skill Evolution Methods
- **Prompt-level**: Reflexion-style critique and retry
- **Structural**: Reorganize skill sections based on usage patterns
- **Content**: Add missing steps, remove outdated info
- **Verification**: Add new test cases from failure traces

## Self-Improving Skill Architecture

### Trace Collection
- Log all task executions with inputs, outputs, errors
- Store traces in ~/.hermes/profiles/agency-analytics-specialist/traces/
- Include: task description, skill used, result, errors

### Failure Analysis
- On task failure, analyze trace
- Identify root cause: missing step, wrong approach, outdated info
- Categorize: skill gap, knowledge gap, tool failure

### Skill Edit Proposal
- Propose specific edits to skill file
- Include: what changed, why, evidence
- Flag: architecture changes, irreversible edits

### Self-Improvement in Action: MiniCPM5-2B Thinking Mode Fix
- **Problem**: All models produce reasoning_content instead of code (thinking mode always on)
- **Analysis**: llama-server has `--reasoning off` parameter to disable thinking mode
- **Fix**: Restart MiniCPM5-2B with `--reasoning off`
- **Result**: Produces working Python code at 37 tok/s on GTX 1650 (1.8GB VRAM)
- **Verification**: Tested on LCS dynamic programming task — correct output
- **Lesson**: Thinking mode is the default for Qwen3.5 and Gemma-4 on llama.cpp; always test with `--reasoning off` for coding tasks

### Activation
- Activate verified edits
- Rollback on regression
- Log: change, evidence, verification result

## Skill Improvement Budget

### Compute Allocation (NGU-style)
- Easy tasks: single attempt, move on
- Hard tasks: multiple attempts, reflection, revision
- Failed tasks: analyze trace, propose fix, verify
- Repeated failures: escalate to user, don't loop forever

### Improvement Depth
- Level 1: Prompt refinement (fast, reversible)
- Level 2: Structural reorganization (medium, reversible)
- Level 3: Content addition (medium, reversible)
- Level 4: Architecture change (slow, requires verification)

## Safety and Governance

### Skill Trust Framework
1. **Tier 1**: Built-in skills (highest trust)
2. **Tier 2**: User-created skills (medium trust)
3. **Tier 3**: Auto-evolved skills (low trust)
4. **Tier 4**: Community-contributed skills (lowest trust)

### Guardrails
- Never modify built-in skills
- Auto-evolved skills require verification before activation
- Rollback on performance regression
- User approval for architecture changes
- Audit trail of all skill changes

## Implementation Steps

### Step 1: Trace Collection
- Log all task executions with inputs, outputs, errors
- Store traces in ~/.hermes/profiles/agency-analytics-specialist/traces/
- Include: task description, skill used, result, errors

### Step 2: Failure Analysis
- On task failure, analyze trace
- Identify root cause: missing step, wrong approach, outdated info
- Categorize: skill gap, knowledge gap, tool failure

### Step 3: Skill Edit Proposal
- Propose specific edits to skill file
- Include: what changed, why, evidence
- Flag: architecture changes, irreversible edits

### Step 4: Verification
- Test edited skill on similar task
- Compare result with original
- Verify: improvement, no regression, no new errors

### Step 5: Activation
- Activate verified edits
- Rollback on regression
- Log: change, evidence, verification result

## Evidence Sources
- arxiv 2602.06043: Self-Improvements in Modern Agentic Systems
- arxiv 2609.13443: Learning to Solve Hard Problems by Never Giving Up
- selfimproving-agent.github.io: survey webpage
- CoEvoSkills: self-evolving agent skills via co-evolutionary verification
- MetaSkill-Evolve: recursive self-improvement via two-timescale meta-skill evolution
- Agent Skills survey: arxiv 2602.12430
- Better Ways to Build Self-Improving AI Agents: yoheinakajima.com
