# File-based learning system: proposed data model

Status: design proposal for discussion. This document defines responsibilities and file boundaries. Detailed schemas, examples, execution protocols, and individual function designs are deferred until we discuss each functionality.

## Main decision

Treat each goal folder as a portable learning workspace. Store product state in documented, versioned files that either a direct model-call application or a Codex harness can understand.

Application and intelligence access the data layer directly within their permissions. Application owns captured interactions, learner choices, and session continuity. Intelligence owns derived assessments, learning decisions, and generated experiences. Shared data operations validate and publish records without deciding teaching policy.

The [three-layer architecture](architecture.md) defines ownership, interaction flows, and an illustrative call stack for submitting and assessing a lesson answer.

All durable product state lives on disk. Model inference can still happen remotely; file-based storage does not imply local model execution.

## Applying Unix design principles

Use small components with explicit responsibilities and documented inputs and outputs. Files provide durable interfaces between them, and components can also expose ordinary functions or libraries. These are responsibility boundaries; they do not require separate processes or shell commands for every action.

| Principle | Application to this system |
| --- | --- |
| Do one job well | Capture records interactions; assessment interprets them; recall schedules retrieval; planning chooses the next learning purpose. |
| Compose tools through clear interfaces | Each component consumes documented records and produces documented outputs that another component can use. |
| Keep durable data inspectable | Use readable JSON for structured records and native files for experiences and learner work. |
| Separate mechanism from policy | The capture library knows how to record an answer. An evaluator decides what that answer means. |
| Preserve inputs and support reassessment | Keep raw observations so an assessment can be rerun with a different evaluator while preserving the original interaction. Save the evaluator and input versions used; model-generated judgments may differ across runs. |
| Make components replaceable | Assessment or generation can use a different runner while retaining the same product records and contracts. |

The conceptual flow is: experience → capture → stored observations → assessment → capability/recall updates → planning. Each arrow is an explicit data handoff. Capturing an interaction must not depend on assessment succeeding.

## Shared capture component

A reusable JavaScript library can be imported by each learning module's HTML. The module calls it at meaningful interaction points, such as submitting an answer, requesting a hint, providing an explanation, or submitting an artifact. Importing the library alone does not identify those interaction points; the module must connect its controls to the capture interface.

The library packages the observation with the relevant module/version, question or step, attempt, and presented task context, then hands it to the application for persistence. The host supplies the session identity and records receipt. The storage adapter handles local file writes; the module's HTML does not need direct filesystem access.

Capture records what was submitted, which hints were shown, and what conditions were presented. It does not calculate correctness, assign a score, infer mastery or independence, or schedule review. Those judgments and decisions belong to separate components.

The exact library API, event schema, persistence acknowledgements, and retry behavior will be designed when we examine capture in detail.

## Proposed layout

```text
learning/
  learner.json
  schemas/                           # Shared JSON Schema contracts
  runtime/
    capture.js                       # Shared interaction capture library
  skills/                            # Shared pedagogical instructions
    discover-goal/SKILL.md
    diagnose-understanding/SKILL.md
    maintain-graph/SKILL.md
    plan-learning/SKILL.md
    create-experience/SKILL.md
    assess-attempt/SKILL.md
    update-recall/SKILL.md

  goals/
    g_001--tiny-transformer/
      goal.json
      goal-proposal.json
      conversation.json
      prerequisite-graph.json
      diagnostic.json
      capability-assessment.json
      recall-model.json
      learning-plan.json
      session.json

      modules/
        m_001/
          v001/
            module.json
            experience.html
            assets/

      attempts/
        a_001.json
      observations/
        o_001.json
      assessments/
        assessment_001.json
      artifacts/
        work_001/
          artifact.json
          files/

      history/
        commit_001/                   # Committed revision manifest and snapshots
      runs/
        run_001/
          request.json
          result.json
          runtime/                    # Provider-specific continuation/checkpoints
```

IDs above are illustrative. Use unique stable IDs in the actual system. A folder slug is a readable label; renaming it must not change identity. Shared skills require explicit discovery/configuration in a Codex adapter; this proposed `skills/` location is not a claim that Codex automatically discovers it.

Start with goal-contained prerequisite graphs. A shared subject graph can be added later without making a goal folder depend on it for reopening: save the relevant graph snapshot and provenance locally.

## Formats and conventions

- JSON: UTF-8, two-space indentation, no comments, validated against JSON Schema Draft 2020-12.
- Markdown: teaching instructions, explanatory documents, and learner notes where appropriate.
- HTML/CSS/JavaScript: generated executable learning experiences. `module.json` describes the experience and references its files.
- Native files: code, images, audio, PDFs, and other learner work. JSON contains metadata and references rather than encoded copies of large files.
- JSONL: a possible format for interaction streams. The capture design will decide between a stream and individual JSON records; the responsibility boundary stays the same.

Every product record has `schema_version`, `kind`, `id`, `goal_id` where applicable, and `created_at`. Mutable snapshots also have `revision` and `updated_at`. Timestamps use ISO 8601 UTC; learner timezone is stored separately for scheduling and display.

`schema_version` changes when the format changes. `revision` changes when the contents of one snapshot change. Module versions and skill versions are separately identified.

References use stable IDs and goal-relative file paths. A specific historical claim must reference the exact relevant record or revision, not just a current filename. Record compact provenance: creator type, run ID where applicable, input references, and a brief justification for inferred conclusions. Provider configuration belongs in run metadata.

Use `null` or an explicit `unknown` state for missing knowledge. Avoid invented scores and precise-looking confidence values without a defined estimation method.

## File responsibilities

### `learner.json`

Goal-independent information: learner ID, language, timezone, accessibility needs, declared background, interests, preferences, and their provenance. Distinguish self-reported background from demonstrated capability. Preferences can change; do not treat them as fixed learning-style classifications.

For the first version, one workspace belongs to one learner. Each goal records `learner_id`. Cross-goal reuse must preserve the origin, date, context, and assistance conditions of evidence.

### `goal.json`

Store learner-confirmed intent, clarified outcome, scope, exclusions, real-world context, required capabilities, success criteria, constraints, and goal status. Application records learner choices; intelligence reads this record and proposes changes separately.

Each success criterion has a stable ID, the capability it concerns, expected demonstration, acceptable support, and an evaluation rubric. Goal changes create a new revision and retain earlier criteria for interpreting historical attempts.

Suggested goal states: `exploring`, `confirmed`, `active`, `paused`, `achieved`, `archived`. These express product state, not a forced sequence of screens.

### `goal-proposal.json`

Intelligence creates and revises a proposed goal using the conversation. Store the proposed outcome, scope, criteria, unresolved questions, and source message references. Application presents a proposal and records an explicit learner confirmation against its revision in `goal.json`. Later proposals preserve the current confirmed direction until the learner changes it.

### `conversation.json`

Store the actual learner-visible dialogue as role-labelled messages. Each message has an ID, timestamp, role, status, typed content blocks, context references, and optional `reply_to`.

Typed content blocks support text, artifact references, and references to learning experiences. If the tutor opens a simulation, its message can reference the module version and step. Save only the content actually presented to the learner as an assistant message; unshown candidate responses stay in run records.

Keep backend tool calls and provider continuation items in `runs/`. A compact conversation summary can aid retrieval, but it records which messages it covers and never replaces the transcript. Edits and regeneration preserve prior messages and record their replacement relationship.

Conversation is evidence of interaction. Extracted goals and assessments are separately stored interpretations with links back to the relevant messages.

### `prerequisite-graph.json`

Store nodes, edges, goal requirements, provenance, unresolved questions, and revision rationale.

Nodes represent concepts or skills and contain explicit assessable capabilities. Each capability has a stable ID and an observable description. Goal requirements reference capability IDs and state the required depth and relevance.

Edges have a stable ID, source, target, relationship type, rationale, applicable context, and a status such as `proposed` or `accepted`. Define direction precisely: a prerequisite edge from A to B means A supports learning or performing B at the specified depth.

Relationship types include `prerequisite`, `component_of`, `helpful_connection`, and `interpretive_connection`. Mandatory prerequisite edges must be acyclic; other relationships may form cycles. A goal may accept alternative prerequisites, expressed as requirement groups with `all_of` or `any_of` semantics.

Record structural revisions separately from changes in learner understanding. Preserve retired node IDs and their replacement mappings when meanings change; do not silently reinterpret old evidence.

### `diagnostic.json`

Store the purpose and current state of initial or follow-up diagnosis: capability questions, hypotheses, task/module references, attempt references, coverage, uncertainty, conclusions, and the reason to continue or stop assessment.

Raw answers belong in `observations/`, grouped by records in `attempts/`; diagnostic interpretations reference records in `assessments/`. Self-report is labelled as such. The diagnostic asks whether enough is known to choose a useful starting point, rather than whether every prerequisite has been tested.

### `capability-assessment.json`

Store the current evidence-backed assessment for each capability and the current assessment against goal success criteria.

For each capability, record status, context, support conditions, assessment references, underlying observation references, misconceptions, last assessment time, unresolved questions, and a short rationale. Candidate statuses include `unknown`, `developing`, `demonstrated_with_support`, and `demonstrated_independently`.

Keep explanation, application, implementation, and performance separate when the evidence differs. Preserve contrasting evidence instead of letting the latest answer erase it. A conclusion is a current interpretation, not a timeless fact.

Goal-level readiness references criteria and evidence. A completed module does not itself satisfy a criterion.

### `recall-model.json`

Store retention and review state for capability targets: relevant retrieval evidence, last meaningful retrieval, next review time, priority, scheduling rationale, and the scheduling method/version.

Use explicit parameters for the selected scheduling method. Leave estimated retention probabilities unset until an actual method supports them. Record assistance and whether an event was instruction, practice, independent retrieval, or transfer; these events need not receive equal review credit.

A module can exercise several capabilities. Credit only component skills actually exercised. Learning about a prerequisite does not automatically refresh all related skills.

### `learning-plan.json`

Store the current proposed route and next action. Actions can include `clarify_goal`, `diagnose`, `teach`, `practice`, `repair`, `retrieve`, `transfer`, and `demonstrate_goal`.

Each action records its purpose, target capability IDs, rationale, dependencies, assessment references, expected observations, priority, and status. It can reference an existing module version or request a new experience. Record the goal, graph, assessment, and recall revisions used to make the plan.

The plan is revisable. An outline of possible future activities is not a promise to generate the whole course upfront.

### `session.json`

Store continuity: current plan action, module version, step, active attempt, saved UI state, draft artifact references, and a return stack for repair detours.

This enables exact resumption after a pause or a prerequisite detour. Saved interface position does not imply learning progress or completion.

### `modules/<module_id>/<version>/module.json`

Store an immutable experience definition: learning purpose, target capabilities, entry assumptions, interaction type, steps, support policy, difficulty dimensions, observation contract, evaluation specification, source references, UI entrypoint, generation provenance, and validation state.

The observation contract identifies stable question/step IDs, meaningful events, response formats, and the presented task conditions to capture. Examples include predictions, answer submissions, explanations, help requests, and artifacts. The experience imports the shared capture library to record them. Evaluation instructions describe a separate assessment responsibility. Raw clicks and slider movement can support context but do not independently establish understanding.

Difficulty dimensions may include complexity, novelty, ambiguity, interacting concepts, and support level. Avoid a single unexplained difficulty number.

Preserve exact module versions used in attempts. Regeneration creates a new version. Answer keys and evaluator-only rubrics can be referenced from separate evaluator files when revealing them would undermine the task.

### `attempts/<attempt_id>.json`

Group the observations belonging to one learner attempt. Store module ID/version, task or step IDs, start/end state, and references to observation records and submitted artifacts.

Raw interaction payloads have one authoritative location in `observations/`; the attempt record groups and references them. Completing an attempt means the interaction was submitted, not that the learner succeeded. A retry creates a new attempt. Drafts and unfinished work remain resumable through `session.json` and artifact files.

### `observations/<observation_id>.json`

Store raw records collected through the shared capture library: event type, submitted answer or explanation, question/step identity, attempt and module/version references, presented inputs, observed help interactions, artifact references, and timestamps.

Preserve enough task context to understand the answer later, including randomized question values or simulation conditions. Record a hint being requested or displayed as an observed event; an assessment component can later decide how that affects interpretation.

These records contain no correctness judgment, score, inferred capability, or recall decision. Capture records remain available even if assessment fails. Submitted observations are immutable; subsequent learner changes create new observations.

### `assessments/<assessment_id>.json`

Store a separate evaluator's interpretation of a defined set of observations, messages, or artifacts, using the relevant module specification or goal criteria.

Record source references, target capabilities, the judgment, its basis, context, limitations, and evaluator identity/version. Label deterministic checks, model judgments, self-report, and external feedback distinctly. Only this assessment layer interprets correctness or the learning significance of captured responses.

Assessment records are immutable. Reassessment creates a new record and identifies what it supersedes. The current `capability-assessment.json` and `recall-model.json` reference the applicable assessments. Historical observations remain intact when judgments change.

### `artifacts/<artifact_id>/`

Store actual learner work in native files. `artifact.json` records identity, type, ownership, versions, source attempts, and local file references. Preserve submitted versions so later edits cannot change the work an earlier assessment evaluated.
