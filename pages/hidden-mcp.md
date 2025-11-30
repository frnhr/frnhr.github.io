# Hidden MCP

Pattern for Claude Code to use to keep specific MCP tools away from main agent's context while making them available to a subagent (and / or packed in a skill).


Specifically, use a dedicated subagent to operate a browser via Google's [chrome-devtools-mcp](https://github.com/ChromeDevTools/chrome-devtools-mcp/) without exposing `chrome-devtools-mcp` to the main agent. Thus keeping the context free.

## Keywords

 * Restrict MCP in Claude Code to subagent
 * Claude Code context optimization
 * MCP context bloat / token usage
 * Lazy-load MCP tools
 * Subagent-only MCP pattern
 * Hide MCP from main agent
 * MCP content bloat antidote :)
 
## TL;DR

 * run [mcp-proxy](https://github.com/sparfenyuk/mcp-proxy) in front of a local stdio MCP
 * dump the MCP tools descriptions into a skill
 * in the same skill describe how to call the proxy directly (`curl`)
 * remove the MCP config
 * $ profit $

Probably also make a subagent to use that skill...

## Whaat?

Some MCP tools return massive responses when used. For example, [`chrome-devtools-mcp`](https://github.com/ChromeDevTools/chrome-devtools-mcp/) regularly returns 30k tokens or more! And this makes sense, it parses a whole HTML page, and it's not [1995](https://www.webdesignmuseum.org/early-websites) any more...

### Ok, use a subagent maybe?

Sure, already using a subagent.

### So, what's the problem?

The problem is that the MCP descriptions still add 4-5k tokens to the main agent's context.

I never plan to use this tool in the main agent so this is a waste. 4-5k is not terrible, but it's not great either.

### Can't you just remove the MCP from the main agent?

No, that's the problem! Or, I can but this also makes it unavailable to the subagent.

## Ahhh, ok... so... How?

Didn't you read the TL;DR up above?

### ...

Here are the steps, "high level":

0. The MCP is already configured and working in Claude Code.
1. Tell Claude to dump the MCP context of the offending MCP to a file, verbatim.
2. Install and run [mcp-proxy](https://github.com/sparfenyuk/mcp-proxy) in front of the MCP
3. Make a new skill, starting with the file from "1.", that explains how to call the MCP via HTTP, i.e. using `curl` (I named the skill `browser`).
4. Make a subagent, I named it `browser-inspector`. Actually, Claude named it that, don't look at me. Instruct teh subagent to always load the skill.
5. Lose the MCP config.


Fancy diagram:

```
┌─────────────────────────────────────────────────────────┐
│ Claude Code (Main Agent)                                │
│ - No massive MCP tool in context any more               │
│ - Spawns browser-inspector subagent when needed         │
└─────────────────────┬───────────────────────────────────┘
                      │ 
                      ▼
┌─────────────────────────────────────────────────────────┐
│ browser-inspector Subagent                              │
│ - Loads browser skill (tool documentation)              │
│ - Calls tools via curl to HTTP API                      │
└─────────────────────┬───────────────────────────────────┘
                      │ curl http://localhost:9221/mcp
                      ▼
┌─────────────────────────────────────────────────────────┐
│ mcp-proxy                                               │
│ - Bridges HTTP ↔ stdio                                  │
│ - Exposes JSON-RPC over HTTP POST                       │
└─────────────────────┬───────────────────────────────────┘
                      │ stdio
                      ▼
┌─────────────────────────────────────────────────────────┐
│ chrome-devtools-mcp                                     │
│ - Actual MCP server                                     │
│ - Controls Chrome browser                               │
└─────────────────────────────────────────────────────────┘
```

### A bit more details

#### `mcp-proxy`

This is a python tool that exposes a stdio-based MCP to HTTP. This is important because we will remove the MCP config from Claude, so we need something to actually run the MCP process. MCPs are stateful so we need something "between" the MCP and our subagent.

Install:
```bash
uv tool install mcp-proxy
```

run:

```bash
mcp-proxy --port=9221 -- npx -y chrome-devtools-mcp@latest
```

Port is arbitrary, used with `curl` later on.

If using a different MCP, replace `npx -y chrome-devtools-mcp@latest` according to the docs.

#### `chrome-devtools-mcp`

This is the MCP that I'm "hiding".

It works nicely, automatically opens a new window when needed, no fiddling with manual connecting.

#### Skill and `curl`

Here is my skill file for the "browser" skill. YMMV. 

````markdown
---
name: browser
description: This skill provides browser automation via the chrome-devtools MCP server, accessed through mcp-proxy HTTP API. Never load this skill unless explicitly asked to do so.
---


# Browser Automation Skill

This skill provides browser automation via the chrome-devtools MCP server, accessed through mcp-proxy HTTP API.

## Prerequisites

The mcp-proxy must be running with chrome-devtools:
```bash
mcp-proxy --port=9221 -- npx -y chrome-devtools-mcp@latest
```

## CRITICAL: Request Format

All requests MUST include these headers:
```
-H "Content-Type: application/json" -H "Accept: application/json"
```

After initialization, also include:
```
-H "Mcp-Session-Id: <session-id>"
```

## CRITICAL: Use Self-Contained Curl Commands

To avoid permission prompts, use **self-contained curl commands** with literal values (no bash variables or scripts).

**BAD** (triggers approval prompt):
```bash
SESSION_ID="abc123"
curl ... -H "Mcp-Session-Id: $SESSION_ID" ...
```

**GOOD** (auto-approved):
```bash
curl -s -X POST http://localhost:9221/mcp -H "Content-Type: application/json" -H "Accept: application/json" -H "Mcp-Session-Id: abc123" -d '...'
```

## Workflow

### Step 1: Initialize Session

Run this curl to get a session ID (check the response headers):

```bash
curl -s -i -X POST http://localhost:9221/mcp -H "Content-Type: application/json" -H "Accept: application/json" -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"browser-agent","version":"1.0.0"}}}'
```

Look for `mcp-session-id` in the response headers. Copy that value.

### Step 2: Send Initialized Notification

Replace `SESSION_ID_HERE` with the actual session ID from step 1:

```bash
curl -s -X POST http://localhost:9221/mcp -H "Content-Type: application/json" -H "Accept: application/json" -H "Mcp-Session-Id: SESSION_ID_HERE" -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'
```

### Step 3: Call Tools

Use the session ID in all subsequent requests. Replace `SESSION_ID_HERE` with your session ID:

```bash
curl -s -X POST http://localhost:9221/mcp -H "Content-Type: application/json" -H "Accept: application/json" -H "Mcp-Session-Id: SESSION_ID_HERE" -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"TOOL_NAME","arguments":{...}}}'
```

## Tool Reference

... the rest is just the MCP tools description ...
````


You might notice the emphasis on using `curl` directly - this is so that Claude can give us the option to "not ask for curl in this session" (or similar). This option will not be shown for bash scripts, resulting in a confirmation prompt for every MCP action. 

#### Subagent

Here is the subagent file. Claude is very keen on using WebFetch tool (built-in) for any tasks with URLs, or at least it seemed so to me...

````markdown
---
name: browser-inspector
description: "This agent MUST be used for any task involving:\n - DOM structure, DOM complexity, DOM analysis, or element counting\n - Analyzing actual DOM structure, element counts, or HTML hierarchy\n - Inspecting computed CSS styles or layout\n - Capturing screenshots or visual analysis\n - Reading browser console logs or errors\n - Any task requiring a real browser environment (JavaScript execution, etc.)\n\nIMPORTANT: If the user's question contains \"DOM\", \"HTML structure\", \"element count\", \"page structure\", or similar: use this agent - NOT WebFetch.\n\nInvoke this agent whenever the request includes:\n - inspecting or querying the DOM or a web page\n - checking CSS styles or layout\n - capturing screenshots\n - reading console logs\n - performing browser automation or frontend debugging\n\nDo NOT perform browser interactions outside this agent.
tools: Bash, Read, Grep, Write, TodoWrite, Skill, AskUserQuestion, mcp__sequential-thinking__sequentialthinking"
model: opus
color: green
---

You are an expert frontend inspector and browser automation specialist. Your role is to efficiently navigate web pages, inspect DOM structures, verify CSS styles, and debug frontend issues.

## CRITICAL: Load the Browser Skill First

Before doing ANY browser operations, you MUST load the browser skill:

```
Use the Skill tool with skill: "browser"
```

This skill contains all the tool documentation for browser automation via HTTP API.



... Other specific instructions, will depend on the MCP ...
````


#### Remove MCP config

Remove it from `.mcp.json` in your project file and / or the main `~/.claude/settings.json`. Might want to disable it instead of removing, use `/mcp` for this.

## What else?

This whole thing is essentially a workaround until this issue is implemented: [anthropics/claude-code#12633](https://github.com/anthropics/claude-code/issues/12633)
Please upvote the issue.

## Why use a skill at all?

Fair point, we could dump all the instructions into the subagent. It would be a bit faster too, wouldn't have to load the skill separately. 

This seems nicer, though. And offers a bit more flexibility for the future.

Maybe sometimes we will want to use it from the main agent, after all. 🤷‍♂️
