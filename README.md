# Multi-Agent Meeting Minutes Pipeline

An office-hour demo of a true multi-agent AI system that automatically processes meeting minutes, extracts action items, performs legal research, and emails results to the relevant team members — across two separate machines on the same local network.

This is for a professional education course on agentic AI. It builds on an earlier [demo](https://github.com/tobah59x/n8nTaskExtractorDemo) where the n8n workflow used a single agent to exract tasks, then sent the emails, all from the same machine. Here, a second AI agent on another machine retrieves the minutes, and performs additional documentation research for legal tasks before the workflow sends the final email.

---

## What It Does

1. **n8n** kicks off the workflow and emails a task to Claude Desktop
2. **Claude Desktop** reads `MeetingMinutes.txt` from its Desktop folder and emails the contents back to n8n
3. **n8n** passes the minutes to a **Llama model via Groq**, which extracts per-attendee action items, then:
   - Emails action items directly to Sales and Engineering attendees
   - Sends a second task to Claude Desktop requesting legal research via DocFind
4. **Claude Desktop** searches its local documentation index for information relevant to the legal tasks and emails the findings back to n8n
5. **n8n** sends the legal attendee their action items along with the research findings

All inter-agent communication happens via email through **smtp4dev**, a fake SMTP/IMAP server running in Docker. The two AI agents in this system are **Claude Desktop** (file access and documentation research) and a **Llama model on Groq** (task extraction from unstructured text).

---

## Repository Contents

| File | Description |
|------|-------------|
| `MultiAgentEmailFlow.json` | The n8n workflow — import this into your n8n instance |
| `MCPServers.json` | MCP server definitions to add to your Claude Desktop config file |
| `ContinuousPoll.txt` | The prompt to paste into Claude Desktop to start the demo |
| `StartupScripts.txt` | Docker commands to start n8n and smtp4dev |
| `MeetingMinutes.txt` | Dummy meeting minutes with dummy attendees and email addresses |
| `.gitignore` | Standard gitignore |
| `LICENSE` | License file |

---

## Two-Machine Setup

This demo runs across two machines on the **same local WiFi network**. Both can be ordinary laptops — no special hardware is required.

- **Agent Machine** — runs **Claude Desktop** with MCP servers. This is where the Claude Desktop agent lives.
- **Workflow Machine** — runs **n8n** and **smtp4dev** in Docker. This is where the workflow orchestration happens.

The Workflow Machine's local IP address must be reachable from the Agent Machine. Each user's network will have different IP addresses — the address `10.0.0.29` appearing in the config files is specific to the demo environment and will need to be replaced with your own Workflow Machine IP address wherever it appears.

> **Note:** Both machines must be on the same local network because smtp4dev's SMTP and IMAP ports are exposed on the Workflow Machine's local IP address, and Claude Desktop on the Agent Machine connects to those ports directly.

---

## Prerequisites

**On Workflow Machine (n8n + smtp4dev):**
- [Docker](https://docs.docker.com/get-docker/) installed and running
- A **Groq API key** (free at [console.groq.com](https://console.groq.com))

**On Agent Machine (Claude Desktop):**
- [Claude Desktop](https://claude.ai/download) installed
  - On Linux: see [this community repo](https://github.com/aaddrick/claude-desktop-debian) for a Debian/Ubuntu build
- [Node.js](https://nodejs.org) (v18 or higher) — needed for the `mcp-mail-server` MCP package
- [npx](https://docs.npmjs.com/cli/v8/commands/npx) — comes with Node.js
- [uv](https://github.com/astral-sh/uv) — Python package manager, needed for the DocFind MCP server
- The **MyFiles** and **DocFind** MCP servers set up as described in [this article](https://towardsdatascience.com/how-i-finally-understood-mcp-and-got-it-working-irl-2/) (covers Mac and Windows; Linux users see the repo above)

---

## Setup Instructions

### Step 1 — Start Docker services on Workflow Machine

Run the commands in `StartupScripts.txt`. You may need to run them with `sudo` depending on your Docker installation. These commands will:
- Create a Docker network called `n8n-network`
- Start **n8n** on port `5678`
- Start **smtp4dev** on ports `3025` (SMTP), `3143` (IMAP), and `8080` (web UI)

After running them, verify:
- n8n is accessible at `http://localhost:5678`
- smtp4dev web UI is accessible at `http://localhost:8080`

### Step 2 — Import the workflow into n8n

1. Open n8n at `http://localhost:5678` and log in
2. Click **New Workflow** → **Import from file**
3. Import and save `MultiAgentEmailFlow.json` — it will save automatically under the file name
4. Set up your credentials inside n8n:
   - **Groq**: add your Groq API key under Credentials
   - **SMTP**: point to `smtp4dev` hostname, port `25`, no authentication
   - **IMAP**: point to `smtp4dev` hostname, port `143`, no authentication, user `n8n@landdev-example.com`

### Step 3 — Configure Claude Desktop on Agent Machine

1. Locate your Claude Desktop config file:
   - **Mac:** `~/Library/Application Support/Claude/claude_desktop_config.json`
   - **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`
   - **Linux:** `~/.config/Claude/claude_desktop_config.json`
2. If you already have a config file with other MCP servers defined, add the three server entries from `MCPServers.json` into the `mcpServers` section of your existing file. If you don't have a config file yet, you can use `MCPServers.json` as your starting point.
3. Update the file paths in the `MyFiles` and `DocFind` sections to match your actual directory structure
4. Replace `10.0.0.29` in the `smtp4dev` section with the Workflow Machine's actual local IP address
5. Restart Claude Desktop

### Step 4 — Place the meeting minutes file

Copy `MeetingMinutes.txt` to the Desktop folder that the `MyFiles` MCP server is configured to access (as set in your Claude Desktop config file).

### Step 5 — Run the demo

1. Open Claude Desktop on Agent Machine
2. Start a new conversation and paste in the contents of `ContinuousPoll.txt`
3. Claude Desktop will clear its mailbox and begin polling every 10 seconds
4. Switch to n8n on Workflow Machine and click **Execute Workflow**
5. Watch the pipeline run — you can monitor emails arriving in smtp4dev at `http://Workflow-Machine-IP:8080`

---

## How the Polling Works

Claude Desktop polls for emails using `search_messages` with IMAP criteria `["ALL"]`, tracking processed message UIDs to avoid reprocessing. This approach works reliably across new conversations because it does not depend on read/unread status.

---

## Monitoring

- **smtp4dev web UI** at `http://Workflow-Machine-IP:8080` — shows all emails sent and received during the demo
- **n8n execution log** — open your workflow in n8n, go to the **Overview** page and click the **Executions** tab to see the full run history with per-node results

---

## Troubleshooting

**Claude Desktop can't connect to smtp4dev**
Verify the Workflow Machine's IP address is correct in your Claude Desktop config file and that both machines are on the same local network. Test connectivity with `ping Workflow-Machine-IP` from the Agent Machine.

**n8n workflow gets stuck polling**
Check smtp4dev's web UI to confirm Claude Desktop's reply email has arrived. If not, check that Claude Desktop was restarted after updating the config file.

**Groq returns no output**
Verify your Groq API key is correctly entered in n8n's credentials. Check the Basic LLM Chain node for error messages.

**Emails from a previous run interfere**
The workflow's first step clears the smtp4dev inbox automatically. You can also clear it manually via the smtp4dev web UI.

---

## References

- [How I Finally Understood MCP and Got It Working IRL](https://towardsdatascience.com/how-i-finally-understood-mcp-and-got-it-working-irl-2/) — Hailey Quach — explains MCP setup for Mac and Windows including the MyFiles and DocFind servers used here
- [claude-desktop-debian](https://github.com/aaddrick/claude-desktop-debian) — community build of Claude Desktop for Linux
- [smtp4dev](https://github.com/rnwood/smtp4dev) — fake SMTP/IMAP server for development
- [n8n](https://n8n.io) — open-source workflow automation
- [mcp-mail-server](https://github.com/yunfeizhu/mcp-mail-server) — MCP server for IMAP/SMTP email operations used by Claude Desktop
