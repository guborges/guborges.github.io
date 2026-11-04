---
title: "WAITING_FOR_HUMAN is not an approval gate: human-in-the-loop in Oracle Select AI Agent"
date: 2026-11-04T18:26:00-03:00
draft: false
categories: ["IA", "Oracle 26Ai", "Oracle DBA"]
tags: ["oracle", "dba", "agentic ai", "select ai agent", "plsql"]
summary: "Why enable_human_tool in Oracle Select AI Agent doesn't block write tools, and a propose/approve/apply PL/SQL pattern that does."
description: "Why enable_human_tool in Oracle Select AI Agent doesn't block write tools, and a propose/approve/apply PL/SQL pattern that does."
cover:
  image: "cover.png"
  alt: "Not an approval gate: human-in-the-loop in Oracle Select AI Agent"
---

If you give an Oracle Select AI agent a tool that changes data, the first question is obvious: how do I make it stop and ask before it runs? The framework has an answer, `enable_human_tool` and the `WAITING_FOR_HUMAN` state. I read the docs carefully and I don't think that answer is enough on its own for anything you'd care about in production.

This post covers what the pause actually does, where it falls short, and a small PL/SQL pattern that moves the approval out of the prompt and into the database.

## What the docs say about the human tool

`enable_human_tool` exists on both the agent and the task. The documented behavior:

- Default is `true`.
- The task value overrides the agent value. If the agent has it off and the task has it on, "the LLM is expected to ask questions" during the task.
- The attribute is described as letting the agent ask the user for information or clarification.

When the agent asks, the run pauses. The history views list the run states as `RUNNING`, `WAITING_FOR_HUMAN`, `RESUMING`, `SUCCEEDED` and `FAILED`. You resume in one of two ways. In SQLcl or SQL*Plus, the next `SELECT AI AGENT` in the same session picks up the paused run. With `DBMS_CLOUD_AI_AGENT.RUN_TEAM`, you pass the same `conversation_id`, and the `user_prompt` becomes the human response.

Read that again: the agent *can* ask. Nothing in the documentation says the framework blocks a tool call until a human answers. Whether the model pauses before calling your write tool depends on the model following instructions. Oracle's own return-processing example handles this the way you'd expect, by writing "Always confirm user name and order number with user before update status" into the tool instruction.

That's a prompt. It works most of the time. Most of the time is not a control.

## The second thing to know: autonomous transactions

The write tools in Oracle's examples use `PRAGMA AUTONOMOUS_TRANSACTION` and commit inside the function. I follow the same pattern below because I couldn't find documentation on how the framework handles the transaction of a tool that doesn't commit, and I'd rather not guess.

The consequence is simple. When the tool returns, the change is committed. There is no "let me look at what the agent did and roll it back". If the confirmation didn't happen before the call, it isn't going to happen.

## The pattern: propose, approve outside, apply

The idea is to split the write into three steps and only expose two of them as tools:

1. `PROPOSE_PAUSE_JOB` (tool): validates the request and writes a pending action. Changes nothing else.
2. `APPROVE_ACTION` (procedure, **not** a tool): a human runs it by hand.
3. `APPLY_ACTION` (tool): applies the action only if it is approved.

An agent can only call tools that are registered and listed on the task it is running. `APPROVE_ACTION` is neither. If the model skips the question and calls `APPLY_ACTION` right after proposing, the function refuses. If the user types "yes, go ahead" in the chat without running the approval, it still refuses. The gate is a row with `status = 'APPROVED'`, not a sentence in a prompt.

The example action is pausing a batch job through an application control table. It's reversible and easy to test, which is what you want while you're learning how the agent behaves.

## The objects

Run this as the schema that owns the agent (here `AGENT_OPS`).

```sql
CREATE TABLE app_job_control (
  job_name  VARCHAR2(128) PRIMARY KEY,
  enabled   VARCHAR2(1) DEFAULT 'Y' NOT NULL CHECK (enabled IN ('Y','N'))
);

INSERT INTO app_job_control (job_name) VALUES ('NIGHTLY_LOAD');
COMMIT;

CREATE TABLE agent_pending_actions (
  action_id    NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  action_type  VARCHAR2(30)  NOT NULL,
  target_name  VARCHAR2(128) NOT NULL,
  status       VARCHAR2(20)  DEFAULT 'PROPOSED' NOT NULL
               CHECK (status IN ('PROPOSED','APPROVED','REJECTED','APPLIED')),
  proposed_at  TIMESTAMP DEFAULT SYSTIMESTAMP,
  approved_by  VARCHAR2(128),
  approved_at  TIMESTAMP,
  applied_at   TIMESTAMP
);
```

The propose tool:

```sql
CREATE OR REPLACE FUNCTION propose_pause_job (
  p_job_name IN VARCHAR2
) RETURN CLOB
AUTHID DEFINER
IS
  PRAGMA AUTONOMOUS_TRANSACTION;
  l_count NUMBER;
  l_id    NUMBER;
BEGIN
  SELECT COUNT(*)
    INTO l_count
    FROM app_job_control
   WHERE job_name = UPPER(p_job_name)
     AND enabled  = 'Y';

  IF l_count = 0 THEN
    RETURN 'No enabled job named ' || p_job_name || '. Nothing was proposed.';
  END IF;

  INSERT INTO agent_pending_actions (action_type, target_name)
  VALUES ('PAUSE_JOB', UPPER(p_job_name))
  RETURNING action_id INTO l_id;

  COMMIT;
  RETURN 'Proposed action ' || l_id || ' (PAUSE_JOB ' || UPPER(p_job_name) ||
         '). A human must approve it outside this conversation before it can be applied.';
EXCEPTION
  WHEN OTHERS THEN
    ROLLBACK;
    RETURN 'Error in propose_pause_job: ' || SQLERRM;
END propose_pause_job;
/
```

The approval procedure. Note the status is computed in PL/SQL before the `UPDATE`, so this also compiles on 19c, where SQL has no `BOOLEAN` type:

```sql
CREATE OR REPLACE PROCEDURE approve_action (
  p_action_id IN NUMBER,
  p_approve   IN BOOLEAN DEFAULT TRUE
)
AUTHID DEFINER
IS
  l_status agent_pending_actions.status%TYPE;
BEGIN
  l_status := CASE WHEN p_approve THEN 'APPROVED' ELSE 'REJECTED' END;

  UPDATE agent_pending_actions
     SET status      = l_status,
         approved_by = SYS_CONTEXT('USERENV', 'SESSION_USER'),
         approved_at = SYSTIMESTAMP
   WHERE action_id = p_action_id
     AND status    = 'PROPOSED';

  IF SQL%ROWCOUNT = 0 THEN
    RAISE_APPLICATION_ERROR(-20001, 'Action ' || p_action_id || ' is not pending.');
  END IF;

  COMMIT;
END approve_action;
/
```

The apply tool:

```sql
CREATE OR REPLACE FUNCTION apply_action (
  p_action_id IN NUMBER
) RETURN CLOB
AUTHID DEFINER
IS
  PRAGMA AUTONOMOUS_TRANSACTION;
  l_type   agent_pending_actions.action_type%TYPE;
  l_target agent_pending_actions.target_name%TYPE;
BEGIN
  SELECT action_type, target_name
    INTO l_type, l_target
    FROM agent_pending_actions
   WHERE action_id = p_action_id
     AND status    = 'APPROVED'
     FOR UPDATE;

  IF l_type = 'PAUSE_JOB' THEN
    UPDATE app_job_control
       SET enabled = 'N'
     WHERE job_name = l_target;
  END IF;

  UPDATE agent_pending_actions
     SET status     = 'APPLIED',
         applied_at = SYSTIMESTAMP
   WHERE action_id = p_action_id;

  COMMIT;
  RETURN 'Action ' || p_action_id || ' applied: ' || l_type || ' ' || l_target || '.';
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    ROLLBACK;
    RETURN 'Action ' || p_action_id || ' is not approved. Ask the user to approve it and wait.';
  WHEN OTHERS THEN
    ROLLBACK;
    RETURN 'Error in apply_action: ' || SQLERRM;
END apply_action;
/
```

The refusal message is written for the model. When the agent jumps ahead, it gets back a sentence telling it what to do next.

## Registering the agent

Only the two functions become tools. The task keeps `enable_human_tool` on, because we still want the agent to ask. We just don't depend on it.

```sql
BEGIN
  DBMS_CLOUD_AI_AGENT.CREATE_TOOL(
    tool_name  => 'PROPOSE_PAUSE_JOB_TOOL',
    attributes => '{"instruction": "Creates a pending request to pause a batch job. It does not pause anything.",
                    "function": "PROPOSE_PAUSE_JOB",
                    "tool_inputs": [{"name": "p_job_name", "description": "Job name as the user wrote it."}]}');

  DBMS_CLOUD_AI_AGENT.CREATE_TOOL(
    tool_name  => 'APPLY_ACTION_TOOL',
    attributes => '{"instruction": "Applies a pending action by id. Works only after a human has approved it.",
                    "function": "APPLY_ACTION",
                    "tool_inputs": [{"name": "p_action_id", "description": "Numeric id returned by PROPOSE_PAUSE_JOB_TOOL."}]}');

  DBMS_CLOUD_AI_AGENT.CREATE_TASK(
    task_name  => 'PAUSE_JOB_TASK',
    attributes => '{"instruction": "Handle this request: {query}. Call PROPOSE_PAUSE_JOB_TOOL, give the user the action id, and ask them to approve it outside the conversation and reply when done. Only after the user replies, call APPLY_ACTION_TOOL with that id and report the result.",
                    "tools": ["PROPOSE_PAUSE_JOB_TOOL", "APPLY_ACTION_TOOL"],
                    "enable_human_tool": "true"}');

  DBMS_CLOUD_AI_AGENT.CREATE_AGENT(
    agent_name => 'JOB_OPS_AGENT',
    attributes => '{"profile_name": "OPS_LLM",
                    "role": "You help operators pause batch jobs. You never claim a job was paused unless APPLY_ACTION_TOOL says so."}');

  DBMS_CLOUD_AI_AGENT.CREATE_TEAM(
    team_name  => 'JOB_OPS_TEAM',
    attributes => '{"agents": [{"name": "JOB_OPS_AGENT", "task": "PAUSE_JOB_TASK"}],
                    "process": "sequential"}');
END;
/
```

`OPS_LLM` is an AI profile created beforehand with `DBMS_CLOUD_AI.CREATE_PROFILE`. It has no `object_list`, so the agent gets an LLM and nothing that generates SQL.

## Running it

Start the conversation and keep the id. You need it to resume:

```sql
VARIABLE conv VARCHAR2(128)

DECLARE
  l_answer CLOB;
BEGIN
  :conv := DBMS_CLOUD_AI.CREATE_CONVERSATION();

  l_answer := DBMS_CLOUD_AI_AGENT.RUN_TEAM(
                team_name   => 'JOB_OPS_TEAM',
                user_prompt => 'Pause the NIGHTLY_LOAD job',
                params      => '{"conversation_id": "' || :conv || '"}');

  DBMS_OUTPUT.PUT_LINE(l_answer);
END;
/
```

Check what was proposed and where the run stopped:

```sql
SELECT action_id, action_type, target_name, status
  FROM agent_pending_actions
 ORDER BY action_id DESC;

SELECT team_name, state, start_date
  FROM user_ai_agent_team_history
 ORDER BY start_date DESC
 FETCH FIRST 1 ROW ONLY;
```

If the agent did what the task says, the action is `PROPOSED` and the run is `WAITING_FOR_HUMAN`. If the run already shows `SUCCEEDED`, the model didn't wait. The table tells you whether that cost anything: the action should still be `PROPOSED`, and `app_job_control` untouched.

The human approves, outside the agent:

```sql
EXEC approve_action(1)
```

Then resume the same conversation:

```sql
DECLARE
  l_answer CLOB;
BEGIN
  l_answer := DBMS_CLOUD_AI_AGENT.RUN_TEAM(
                team_name   => 'JOB_OPS_TEAM',
                user_prompt => 'Approved, go ahead',
                params      => '{"conversation_id": "' || :conv || '"}');

  DBMS_OUTPUT.PUT_LINE(l_answer);
END;
/
```

And confirm with data, not with the agent's answer:

```sql
SELECT job_name, enabled FROM app_job_control;

SELECT action_id, status, approved_by, approved_at, applied_at
  FROM agent_pending_actions
 ORDER BY action_id DESC;

SELECT tool_name, input, output, start_date
  FROM user_ai_agent_tool_history
 ORDER BY start_date DESC;
```

The tool history is where you'd catch an `APPLY_ACTION_TOOL` call made before the approval. It shows up with the refusal in `OUTPUT`.

## Why the extra table is worth it

The documentation says the agent history views show each user only what that user owns, access can't be granted to others, and a DBA connecting to them also sees only their own objects. So as the DBA you can't open `USER_AI_AGENT_TOOL_HISTORY` and review another schema's agent runs.

`AGENT_PENDING_ACTIONS` is a normal table. You can grant `SELECT` on it, audit it, and keep it for as long as your retention policy requires. It also records who approved, which the agent views don't.

## Design considerations (not validated in a lab)

These follow from the design. I haven't reproduced them:

- **The gate only holds if no other tool can write.** Add the built-in `SQL` tool or a generic "execute statement" function to the same task and the model has a path around `APPLY_ACTION`.
- **Approval by the same user is weak separation.** Here the human who runs the agent can also approve. For real separation of duties, grant `EXECUTE` on `APPROVE_ACTION` to a different user or role than the one that runs the team.
- **Pending rows pile up.** Abandoned conversations leave `PROPOSED` actions. Expire them with a scheduler job, or `APPLY_ACTION` will happily apply something approved last month.
- **The agent can still say the wrong thing.** The refusal protects the data. It doesn't stop the model from telling the user "done". Check the table.

## Technical validation

Confirmed in Oracle documentation:

- `enable_human_tool` on agent and task, default `true`, and task precedence over agent.
- Resuming a paused run via `SELECT AI AGENT` in the same session or `RUN_TEAM` with the same `conversation_id`.
- Run states `RUNNING`, `WAITING_FOR_HUMAN`, `RESUMING`, `SUCCEEDED`, `FAILED`.
- `USER_AI_AGENT_TEAM_HISTORY`, `USER_AI_AGENT_TASK_HISTORY` and `USER_AI_AGENT_TOOL_HISTORY` columns, and the visibility rule that history is owner-only, DBA included.
- Use of `PRAGMA AUTONOMOUS_TRANSACTION` and prompt-level confirmation in Oracle's write-tool examples.

Documented for Autonomous AI Database Serverless only:

- The history views above. `DBMS_CLOUD_AI_AGENT` is also available on Oracle Database 19c from 19.29 and on Oracle AI Database 26ai from 23.26, but the 26ai package reference I checked doesn't document these views. Check `ALL_VIEWS` on your release before relying on them.

One inconsistency worth knowing: the history view reference lists a `COVERSATION_PARAM` column on `USER_AI_AGENT_TASK_HISTORY`, while Oracle's own example query uses `conversation_params`. Describe the view on your release before you write queries against it. I left that column out of the queries above.

Not run in a lab: my test database is 19.3, below 19.29, so this pattern hasn't been run end to end. The PL/SQL objects use only features available in 19c. Test in a non-production database first.

Varies by environment: the model and provider behind the AI profile, which directly affects how often the agent actually waits before calling a tool.

## Wrap-up

`WAITING_FOR_HUMAN` is useful. It gives you a clean place to resume a conversation and a state you can query. But it describes what the agent chose to do, not what it was allowed to do. If a tool writes, put the approval somewhere the model can't reach: a row, a status, a procedure that isn't registered as a tool. Then the prompt is just a nice-to-have.

## References

- [DBMS_CLOUD_AI_AGENT Package, Autonomous AI Database Serverless](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/dbms-cloud-ai-agent-package.html)
- [Examples of Using the Select AI Agent Framework](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/examples-using-select-ai-agent.html)
- [DBMS_CLOUD_AI_AGENT History Views](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/dbms-cloud-ai-agent-views-history.html)
- [DBMS_CLOUD_AI_AGENT, Oracle AI Database 26ai PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/26/arpls/DBMS_CLOUD_AI_AGENT.html)
