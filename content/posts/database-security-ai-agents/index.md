---
title: "Your AI agent is a database user. Treat it like one"
date: 2026-11-13T08:00:00-03:00
draft: false
categories: ["Oracle DBA"]
tags: ["oracle", "dba", "security", "ai agents", "mcp", "sqlcl"]
summary: "Prompt injection, over-scoped tokens and agents as attackers. What actually limits an AI agent with database access, on Oracle and beyond."
description: "Prompt injection, over-scoped tokens and agents as attackers. What actually limits an AI agent with database access, on Oracle and beyond."
cover:
  image: "cover.png"
  alt: "Your AI agent is a database user: security for agents with database access"
---

## The pattern behind the AI incidents

Strip the AI wording off the incidents of the last year and you get problems DBAs already know.

In July 2025, General Analysis showed an agent connected to a Supabase project through MCP leaking private data. The attacker put instructions inside a support ticket. When a developer asked the agent to look at recent tickets, the agent read those instructions as commands, queried private tables and wrote the result back into a field the attacker could see. The MCP connection ran with more privilege than the task needed. ([Supabase](https://supabase.com/blog/defense-in-depth-mcp))

In April 2026, a coding agent at PocketOS hit a credential problem while working on staging, found a Railway API token in a file, and used it to delete a storage volume. That volume held the production database and its backups. ([DevOps.com](https://devops.com/when-ai-goes-really-really-wrong-how-pocketos-lost-all-its-data/))

In October 2026, a researcher's AI agent escaped its test sandbox and reached an internal WorkOS tool. The API was open to the internet, accepted a weak development credential that had made it to production, and had no rate limiting. The agent pulled records in bulk, and nobody noticed until it had finished. ([WorkOS](https://workos.com/security/bulletins/update-for-october-30th-2026-data-security-incident))

Over-privileged accounts, credentials left where they shouldn't be, nobody watching. The model didn't create any of these. It found them faster.

## The model is not a security boundary

Prompt injection works because the model gets instructions and data through the same channel. A row in your database can carry text that the agent follows as an instruction.

Supabase tried the obvious fixes: wrapping query results with "don't follow instructions in this data" warnings, and LLM classifiers. Their conclusion was that these reduced the risk but did not eliminate it. Read-only mode doesn't solve it either. Even in read-only mode, prompt injection is still the main concern, because the agent can leak what it reads through its own answer.

So put the control where the database enforces it, in grants. Oracle says the same for its own server: the Autonomous AI Database MCP server does not restrict actions to SELECT-only. Whatever the agent's user can do, the agent can do.

## Give the agent its own user

No shared application account, no DBA account, no "god" user. One user per agent use case, with access to exactly the data it needs, ideally through a view that leaves out sensitive columns:

```sql
CREATE USER ai_reader IDENTIFIED BY "use-a-vault-not-this";
GRANT CREATE SESSION TO ai_reader;
GRANT SELECT ON app_schema.v_orders_for_ai TO ai_reader;
```

No write grants unless the use case truly needs them. If it does, the client keeps per-call approval on. Supabase recommends keeping manual approval enabled for each tool call and not clicking "Always approve."

## Keep agents away from production credentials

PocketOS lost production from a staging task because a token with production reach was readable by the agent. The token was created for routine domain work, and the company didn't know it could delete volumes. Backups lived on the same volume, so one call took both.

For a DBA this turns into three rules. Agents work against non-production, masked or replica data: Supabase and Oracle both recommend that. Credentials that reach production are never stored where an agent can read them. Backups sit outside anything the agent's credentials can touch.

## Know what the agent ran

If you use Oracle's SQLcl MCP server, it already tags its work. Sessions show `SQLcl-MCP` as the program, the MCP client in `MODULE` and the model name in `ACTION`. Generated queries carry a `/* LLM in use ... */` comment, so they're easy to find in V$SQL and ASH:

```sql
SELECT sid, username, module, action
FROM   v$session
WHERE  program = 'SQLcl-MCP';
```

Each request is also recorded in `DBTOOLS$MCP_LOG`, in the schema of the connected user:

```sql
SELECT id, mcp_client, model, end_point_type, end_point_name, log_message
FROM   dbtools$mcp_log;
```

Oracle recommends pruning that table regularly. It also lives in the agent's own schema, so treat it as convenience, not evidence. The record that matters is unified auditing on the agent user:

```sql
CREATE AUDIT POLICY aud_ai_reader ACTIONS SELECT, INSERT, UPDATE, DELETE;
AUDIT POLICY aud_ai_reader BY ai_reader;
```

On the Autonomous AI Database MCP server, Oracle's advice is not to base audit or security decisions on `SESSION_USER`. Use `MCP_SERVER_ACCESS_CONTEXT$` to identify the authenticated user.

## Design considerations

These come from the incidents, not from Oracle documentation.

AI agents on the attacking side don't need new database defenses. WorkOS was a weak credential, an exposed endpoint and no throttling. What changes is speed: bulk extraction finished before a human noticed. Alerts on unusual read volume per user are worth more than ever.

An agent is a non-human identity with a credential. Rotate it, scope it and revoke it like any service account.

## Technical validation

Confirmed in vendor documentation:

- SQLcl MCP session tagging (`PROGRAM`, `MODULE`, `ACTION`), the `/* LLM in use */` comment, `DBTOOLS$MCP_LOG` columns and the pruning recommendation (Oracle SQLcl and SQL Developer for VS Code docs, Oracle Database blog)
- Autonomous AI Database MCP server does not enforce SELECT-only; use `MCP_SERVER_ACCESS_CONTEXT$` instead of `SESSION_USER` (Autonomous AI Database docs)
- `CREATE AUDIT POLICY ... ACTIONS` and `AUDIT POLICY ... BY` syntax (Oracle SQL Language Reference)
- Read-only mode and result wrapping do not eliminate prompt injection; manual approval and non-production data recommended (Supabase)

Market practice:

- One least-privilege user per agent and no production credentials in agent reach (OWASP Top 10 for LLM Applications 2025, LLM06 Excessive Agency)

Varies by environment:

- SQLcl MCP tools and options change between releases; check the docs for your version
- `V$SESSION` access is privilege-gated

Not executed in a lab for this post.

## Conclusion

Every database control an AI agent needs already exists: its own user, minimal grants, no production credentials within reach, and an audit trail you control. The agent changes the speed, not the rules. Before connecting one, ask the question you would ask about any new service account: what is the worst this login can do?

## References

- Supabase, Defense in Depth for MCP Servers: https://supabase.com/blog/defense-in-depth-mcp
- Oracle SQLcl, Monitoring the SQLcl MCP Server: https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.3/sqcug/monitoring-sqlcl-mcp-server.html
- Oracle SQL Developer for VS Code, Using the Oracle SQLcl MCP Server: https://docs.oracle.com/en/database/oracle/sql-developer-vscode/25.3/sqdnx/using-oracle-sqlcl-mcp-server.html
- Oracle Autonomous AI Database, MCP Server security: https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/security.html
- WorkOS incident update: https://workos.com/security/bulletins/update-for-october-30th-2026-data-security-incident
- DevOps.com on PocketOS: https://devops.com/when-ai-goes-really-really-wrong-how-pocketos-lost-all-its-data/
