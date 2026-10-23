# Claude Code with the BERDL Data Lake

Claude Code is an AI-powered assistant that can work with your BERDL data lake through the BERDL Datalake MCP server (Model Context Protocol). It lets you explore and query your Iceberg tables in natural language from a terminal in your BERDL notebook.

> **⚠️ Important Security Warning:**
> - Query results will be sent to Anthropic's servers
> - Only use with data that is safe to share publicly

## Prerequisites

Before setting up Claude Code in JupyterHub, ensure you have:
- **Access to BERDL JupyterHub** (see [User Guide](user_guide.md))
- **Active Anthropic subscription** (Claude Pro/Max) or API access with billing enabled at [console.anthropic.com](https://console.anthropic.com)

## Setup

### Step 1: Open Claude Code

Claude Code is pre-installed in BERDL notebooks. **Open a terminal in JupyterLab** (File → New → Terminal) and check:

```bash
claude --version
```

### Step 2: Authenticate Claude Code

Before configuring MCP servers, authenticate with your Anthropic account:

```bash
# Start authentication flow
claude auth login
```

This will open an OAuth flow. Follow the prompts to complete authentication.

> **Note:** You need an active Claude Pro/Max subscription or API billing enabled at console.anthropic.com

### Step 3: Your KBase Authentication Token

Your current KBase token is kept in `~/.berdl_kbase_session`; the next steps read it from there, so you never need to display it. Don't `print()` the token in a notebook: cell output is saved in the notebook file and copied to the platform's logs. Keep it out of version control.

### Step 4: Configure BERDL MCP Server

In a terminal, add the BERDL MCP server to Claude Code. `$DATALAKE_MCP_SERVER_URL` is already set in your notebook environment:

```bash
claude mcp add --transport http --scope user berdl-datalake \
  "$DATALAKE_MCP_SERVER_URL/mcp" \
  --header "Authorization: Bearer $(cat ~/.berdl_kbase_session)"

# Secure the configuration file
chmod 600 ~/.claude.json
```

This adds the server to `~/.claude.json`, Claude Code's own settings file; don't overwrite that file by hand.

### Step 5: Verify MCP Server Connection

Test that the MCP server is accessible:

```bash
# Test the list databases endpoint
curl -X 'POST' \
  "$DATALAKE_MCP_SERVER_URL/delta/databases/list" \
  -H 'accept: application/json' \
  -H "Authorization: Bearer $(cat ~/.berdl_kbase_session)" \
  -H 'Content-Type: application/json' \
  -d '{}'
```

You should see a JSON response listing your accessible databases.

**Verify Claude Code recognizes the MCP server:**

```bash
# List configured MCP servers and their connection status
claude mcp list
```

### Step 6: Start Using Claude Code

Launch Claude Code in your project directory:

```bash
# Navigate to your work directory
cd ~/your-project

# Start Claude Code
claude
```

Claude Code will start an interactive session. You can now ask questions about your BERDL data!

**Example first prompt:**
```
List all databases in the BERDL data lake
```

## Using Claude Code with Your Data

Once configured, you can interact with your BERDL data using natural language. Claude Code will use the MCP server to execute queries against your Iceberg tables.

### Example Prompts

**Database Exploration:**
```markdown
- "List all databases in the BERDL data lake"
- "Show me the tables in my personal database my.demo_personal"
- "What's the schema of the personal_test_table in my.demo_personal?"
- "List all tables in the kbase.ontology_source database"
```

**Data Analysis:**
```markdown
- "Count the total rows in kbase.ontology_source.entailed_edge"
- "Show me 10 sample rows from the kbase.ontology_source.entailed_edge"
- "What columns are available in the kbase.ke_pangenome database tables?"
- "Get the schema for all tables in globalusers.demo_shared"
```

### Best Practices for Claude Code Queries

1. **Be specific about database and table names**: Use the fully qualified `catalog.namespace.table` name (e.g., `my.analytics.products`)
2. **Start with small queries**: Use `LIMIT` to avoid fetching large datasets
3. **Verify schemas first**: Ask for table schemas before running complex queries
4. **Test queries incrementally**: Build up complex queries step by step
5. **Remember data privacy**: Only query data that's safe to share with Anthropic's servers

## Troubleshooting

### "Authentication Failed" for MCP Server

If you see KBase authentication errors or KBase token is expired:

1. **Re-add the server** with your current token, in a terminal:
   ```bash
   claude mcp remove berdl-datalake --scope user
   claude mcp add --transport http --scope user berdl-datalake \
     "$DATALAKE_MCP_SERVER_URL/mcp" \
     --header "Authorization: Bearer $(cat ~/.berdl_kbase_session)"
   ```

2. **Check the connection**: `claude mcp list`

3. **Restart Claude Code** to reload the configuration

## Getting Help

If the MCP server keeps failing after you re-add it, contact the BERDL Platform team.
