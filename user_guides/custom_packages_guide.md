# Guide: Installing Custom Python Packages

The default BERDL notebook environment comes with many pre-installed packages. However, if you need additional Python packages for your analysis, you can create a custom virtual environment that persists across sessions.

## Creating a Custom Virtual Environment

### Step 1: Open a Terminal

In JupyterLab, open a terminal:
- Click **File** → **New** → **Terminal**

### Step 2: Create a Virtual Environment That Inherits System Packages

Create a new virtual environment in your home directory with access to all the default BERDL packages:

```bash
python -m venv --system-site-packages ~/my_venv
```

The `--system-site-packages` flag allows your custom environment to inherit all packages from the default BERDL environment (PySpark, pandas, etc.) while still allowing you to install additional packages.

You can name the environment anything you like (e.g., `~/ml_env`, `~/viz_env`, etc.)

> **💡 Tip:** If you want a completely isolated environment instead (without inheriting BERDL packages), omit the `--system-site-packages` flag:
> ```bash
> python -m venv ~/my_isolated_env
> ```

### Step 3: Activate the Virtual Environment

```bash
source ~/my_venv/bin/activate
```

You should see `(my_venv)` appear in your terminal prompt.

### Step 4: Verify Inherited Packages (Optional)

You can verify that you have access to the default BERDL packages by importing them:

```bash
# Verify system packages are accessible
python -c "import pyspark; print('PySpark:', pyspark.__version__)"
python -c "import pandas; print('Pandas:', pandas.__version__)"
python -c "from berdl_notebook_utils import get_spark_session; print('BERDL utils: OK')"
```

> **💡 Note:** Running `pip list` with `--system-site-packages` can be very slow (30-60 seconds) as it scans hundreds of system packages. The commands above verify package access much faster. If you want to see all packages, be patient or skip this step and proceed directly to installing your additional packages.

### Step 5: Install Additional Packages

> **❌ Activate your venv first.** Run `pip install` only **after**
> `source ~/my_venv/bin/activate` (Step 3); your prompt should show `(my_venv)`.
> Outside an activated venv, pip stops with:
> ```
> ERROR: Could not find an activated virtualenv (required).
> ```
> That is deliberate: it keeps your packages out of the image's own Python. Don't work
> around it with `pip install --user` or `PYTHONPATH=~/.local/...` — a package in
> `~/.local` shadows the image's copy in your notebooks (see
> [Troubleshooting](#a-package-in-local-is-breaking-my-notebooks)).

Install only the additional packages you need, including `ipykernel`. In this example, we'll install `plotly` for visualization:

```bash
pip install ipykernel plotly
```

> **💡 General Pattern:**
> ```bash
> pip install ipykernel <additional-packages>
> ```
> Replace `<additional-packages>` with the packages you need (e.g., `plotly`, `scikit-learn`, `tensorflow`, etc.)

You can install multiple packages at once:

```bash
pip install ipykernel plotly seaborn scikit-learn matplotlib
```

Since you're using `--system-site-packages`, you don't need to reinstall packages that are already in the base BERDL environment.

### Step 6: Register as Jupyter Kernel

Register your virtual environment as a Jupyter kernel so you can use it in notebooks:

```bash
python -m ipykernel install --user --name my_venv --display-name "Python (my_venv)"
```

- `--name my_venv`: Internal kernel name (must be unique)
- `--display-name "Python (my_venv)"`: Name shown in JupyterLab

### Step 7: Refresh Your Browser

Refresh your browser page to see the new kernel option.

### Step 8: Select Your Custom Kernel

**For a new notebook:**
1. Click **File** → **New** → **Notebook**
2. Select "Python (my_venv)" from the kernel list

**For an existing notebook:**
1. Click on the kernel name in the top-right corner of your notebook
2. Select "Python (my_venv)" from the dropdown menu

## Using Your Custom Environment

Once you've switched to your custom kernel, you can import and use the packages you installed:

```python
import plotly.graph_objects as go
import seaborn as sns
import matplotlib.pyplot as plt

# Create a simple Plotly visualization
fig = go.Figure(data=go.Scatter(x=[1, 2, 3, 4], y=[10, 15, 13, 17]))
fig.update_layout(title="My Custom Plot")
fig.show()
```

## Managing Multiple Virtual Environments

You can create multiple virtual environments for different projects or purposes:

```bash
# Machine learning environment (inherits BERDL packages + adds ML tools)
python -m venv --system-site-packages ~/ml_env
source ~/ml_env/bin/activate
pip install scikit-learn tensorflow ipykernel
python -m ipykernel install --user --name ml_env --display-name "Python (ML)"
deactivate

# Data visualization environment (inherits BERDL packages + adds viz tools)
python -m venv --system-site-packages ~/viz_env
source ~/viz_env/bin/activate
pip install plotly bokeh altair ipykernel
python -m ipykernel install --user --name viz_env --display-name "Python (Visualization)"
deactivate

# Bioinformatics environment (inherits BERDL packages + adds bio tools)
python -m venv --system-site-packages ~/bio_env
source ~/bio_env/bin/activate
pip install biopython ipykernel
python -m ipykernel install --user --name bio_env --display-name "Python (Bioinformatics)"
deactivate
```

Each kernel runs with:
- **All the default BERDL packages** (PySpark, pandas, etc.)
- **Its own additional specialized packages**
- **Access to BERDL helper functions** (`get_spark_session()`, `display_df()`, etc.)

## Command-Line Tools

To install a Python **command-line tool** (for example a KIND app you run yourself), use
`pipx` instead of a venv you manage by hand. It puts each tool in its own environment and
its command in `~/.local/bin`:

```bash
pipx install --system-site-packages <package or git+https://… URL>
pipx ensurepath   # once, then open a new terminal so the command is on your PATH
```

`--system-site-packages` lets the tool use the image's packages (PySpark,
`berdl_notebook_utils`). Update a tool with `pipx install --force …` (same arguments) and
remove it with `pipx uninstall <name>`.

## Useful Commands

### List All Available Kernels

```bash
jupyter kernelspec list
```

This shows all registered kernels and their locations.

### Remove a Kernel

If you no longer need a custom kernel:

```bash
jupyter kernelspec uninstall my_venv
```

Note: This only removes the kernel registration, not the virtual environment itself.

### Remove a Virtual Environment

To completely delete a virtual environment and free up disk space:

```bash
rm -rf ~/my_venv
```

**Warning:** Make sure to uninstall the kernel first if you've registered it.

### Update Packages in a Virtual Environment

To update packages in your custom environment:

```bash
source ~/my_venv/bin/activate
pip install --upgrade plotly
```

## Important Notes

> **💡 Persistence:**
> - Virtual environments are stored in your home directory (`~/`) and persist across notebook sessions
> - Your files will remain available even after your JupyterHub server restarts
> - Each kernel maintains its own independent set of packages

> **💾 Disk Space:**
> - Virtual environments can consume significant disk space, especially with large packages like TensorFlow
> - Monitor your home directory usage and remove unused environments when no longer needed
> - Check disk usage with: `du -sh ~/my_venv`

> **🔄 Kernel Restart:**
> - If you install new packages in an active kernel, you may need to restart the kernel to use them
> - Click **Kernel** → **Restart Kernel** in JupyterLab

## Troubleshooting

### Kernel Doesn't Appear After Registration

1. Refresh your browser page
2. If still not visible, verify registration: `jupyter kernelspec list`
3. Check for errors in the terminal when you ran the install command

### Import Error After Installing Package

1. Verify you're using the correct kernel (check top-right corner of notebook)
2. Restart the kernel: **Kernel** → **Restart Kernel**
3. Verify package installation:
   ```python
   import sys
   print(sys.executable)
   !pip list | grep plotly
   ```

### Virtual Environment Activation Fails

Make sure you're using the correct path:
```bash
source ~/my_venv/bin/activate
```

If the environment doesn't exist, create it first with `python -m venv ~/my_venv`

### A package in ~/.local is breaking my notebooks

Symptoms: an import in a notebook fails or behaves differently than in a terminal (for
example a version conflict), or a command in `~/.local/bin` fails in a terminal with
`ModuleNotFoundError`, or a JupyterLab widget or chart renders wrongly. Cause: an older
`%pip install` in a notebook or `pip install --user` left a package in `~/.local` (new ones
are refused now). Notebook kernels load `~/.local` ahead of the image's packages, and
JupyterLab loads extensions from `~/.local/share/jupyter/labextensions` ahead of the
image's; the notebook server, terminals and KIND ignore `~/.local` Python packages.

1. See what's there:
   ```bash
   ls ~/.local/lib/python*/site-packages/ ~/.local/share/jupyter/labextensions/
   ```
2. Remove the package from a notebook cell running the default **Python 3** kernel (pip
   refuses changes outside a venv, so the variable allows this one cleanup), then restart
   the kernel (**Kernel** → **Restart Kernel**) and reload the browser tab:
   ```python
   !PIP_REQUIRE_VIRTUALENV=0 pip uninstall -y <package>
   ```
3. For a command-line tool, also delete its leftover script and reinstall it with pipx
   (see [Command-Line Tools](#command-line-tools)):
   ```bash
   rm -f ~/.local/bin/<tool>
   pipx install --system-site-packages <package>
   ```
4. Going forward, install packages into an activated venv (Steps 2–5), or with pipx
   for command-line tools.

If you're unsure which package is the culprit, contact the BERDL Platform team.

## Best Practices

1. **Use descriptive names**: Name your environments based on their purpose (e.g., `ml_env`, `viz_env`)
2. **Document your packages**: Keep a `requirements.txt` file in your notebook directory:
   ```bash
   source ~/my_venv/bin/activate
   pip freeze > ~/requirements.txt
   ```
3. **Clean up unused environments**: Regularly remove environments you no longer use
4. **Request common packages**: If you find yourself installing the same packages repeatedly, contact the BERDL Platform team to have them added to the base image

## Using Claude Code CLI with BERDL Datalake MCP Server

Claude Code CLI is an AI-powered development assistant that can interact with your BERDL data lake through the Model Context Protocol (MCP). This allows you to query your Iceberg tables using natural language directly from your JupyterHub terminal.

> **⚠️ Important Security Warning:**
> - Query results will be sent to Anthropic's servers
> - Only use with data that is safe to share publicly

### Prerequisites

Before setting up Claude Code CLI in JupyterHub, ensure you have:
- **Access to BERDL JupyterHub** (see [User Guide](user_guide.md))
- **Active Anthropic subscription** (Claude Pro/Max) or API access with billing enabled at [console.anthropic.com](https://console.anthropic.com)

### Step 1: Install Claude Code CLI

**Open a terminal in JupyterLab** (File → New → Terminal):

```bash
# Install Claude Code CLI using the native installer
curl -fsSL https://claude.ai/install.sh | bash

# Verify installation
claude --version
```

The native installer handles all dependencies automatically and doesn't require Node.js.

> **💡 Tip:** If the installation script fails, you may need to add the Claude Code binary to your PATH:
> ```bash
> echo 'export PATH="$HOME/.claude/bin:$PATH"' >> ~/.bashrc
> source ~/.bashrc
> ```

### Step 2: Authenticate Claude Code

Before configuring MCP servers, authenticate with your Anthropic account:

```bash
# Start authentication flow
claude auth login
```

This will open an OAuth flow. Follow the prompts to complete authentication.

> **Note:** You need an active Claude Pro/Max subscription or API billing enabled at console.anthropic.com

### Step 3: Get Your KBase Authentication Token

In a JupyterHub notebook cell, run:

```python
import os
token = os.environ.get('KBASE_AUTH_TOKEN')
print(f"Your KBase Token: {token}")
```

**Important:** Keep this token secure and never commit it to version control.

### Step 4: Configure BERDL MCP Server

Create an MCP configuration file in your home directory:

```bash
# Create the .claude.json configuration file
cat > ~/.claude.json << 'EOF'
{
  "mcpServers": {
    "berdl-datalake": {
      "type": "sse",
      "url": "http://datalake-mcp-server.prod:8000/apis/mcp/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_KBASE_TOKEN_HERE"
      }
    }
  }
}
EOF

# Secure the configuration file
chmod 600 ~/.claude.json
```

**Replace `YOUR_KBASE_TOKEN_HERE` with the token from Step 3.**

### Step 5: Verify MCP Server Connection

Test that the MCP server is accessible:

```bash
# Test the list databases endpoint
curl -X 'POST' \
  'http://datalake-mcp-server.prod:8000/apis/mcp/delta/databases/list' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer YOUR_KBASE_TOKEN_HERE' \
  -H 'Content-Type: application/json' \
  -d '{}'
```

**Replace `YOUR_KBASE_TOKEN` with your actual token from Step 3.**

You should see a JSON response listing your accessible databases.

**Verify Claude Code recognizes the MCP server:**

```bash
# List configured MCP servers
claude mcp list

# Check MCP server health
claude doctor
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

## Troubleshooting Claude Code CLI Integration

### "Authentication Failed" for MCP Server

If you see KBase authentication errors or KBase token is expired:

1. **Get a fresh token** from your JupyterHub session:
   ```python
   import os
   print(os.environ.get('KBASE_AUTH_TOKEN'))
   ```

2. **Update ~/.claude.json** with the new token:
   ```bash
   vim ~/.claude.json
   # Replace the Bearer token in the Authorization header
   ```

3. **Verify the update**:
   ```bash
   grep -m 1 '"Authorization"' ~/.claude.json
   ```

4. **Restart Claude Code** to reload the configuration

## Getting Help

If you need packages added to the default BERDL environment or encounter issues with custom kernels, please contact the BERDL Platform team for assistance.
