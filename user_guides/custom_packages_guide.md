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

> **💡 Tip:** If you want an environment without the image's packages instead, omit the `--system-site-packages` flag (only PySpark stays importable; pandas, `berdl_notebook_utils` and the pre-loaded helpers do not). Such a venv also lacks `ipykernel`, so add it to your installs in Step 5 (`pip install ipykernel <packages>`) to use the venv as a kernel:
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

> **💡 Note:** `pip list` here also shows the several hundred packages the venv inherits from the image; `pip list --local` shows only the ones you installed.

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

Install only the packages the image doesn't already have. Your venv was created with `--system-site-packages` (Step 2), so it already sees every package in the image, including `ipykernel`, which Step 6 uses. In this example, we'll install `xgboost`:

```bash
pip install xgboost
```

> **💡 General Pattern:**
> ```bash
> pip install <additional-packages>
> ```
> Replace `<additional-packages>` with the packages you need (e.g., `xgboost`, `polars`, `tensorflow`, etc.)

You can install multiple packages at once:

```bash
pip install xgboost lightgbm polars
```

Already in the image (no install needed): pandas, PySpark, SciPy, scikit-learn, statsmodels, matplotlib, seaborn, plotly, bokeh, altair, Biopython, scikit-bio, scanpy, umap-learn, networkx, boto3, ipykernel.

> **🧩 Widgets:** JupyterLab loads extensions only from the image, so a package that needs its own JupyterLab extension (e.g. `ipyleaflet` and most ipywidgets add-ons) installs in your venv, but its widgets won't display. Ask the BERDL Platform team to add it to the image. Plotly and anywidget-based widgets work.

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

Once you've switched to your custom kernel, you can import and use the packages you installed alongside the image's:

```python
from sklearn.datasets import make_classification
from xgboost import XGBClassifier

# Train a small model on synthetic data
X, y = make_classification(n_samples=1000, n_features=20, random_state=0)
model = XGBClassifier(n_estimators=50).fit(X, y)
print(f"Training accuracy: {model.score(X, y):.3f}")
```

## Managing Multiple Virtual Environments

You can create multiple virtual environments for different projects or purposes:

```bash
# Machine learning environment (inherits BERDL packages + adds ML tools)
python -m venv --system-site-packages ~/ml_env
source ~/ml_env/bin/activate
pip install tensorflow xgboost
python -m ipykernel install --user --name ml_env --display-name "Python (ML)"
deactivate

# Graph and map visualization environment (inherits BERDL packages + adds viz tools)
python -m venv --system-site-packages ~/viz_env
source ~/viz_env/bin/activate
pip install folium pyvis
python -m ipykernel install --user --name viz_env --display-name "Python (Visualization)"
deactivate

# Bioinformatics environment (inherits BERDL packages + adds bio tools)
python -m venv --system-site-packages ~/bio_env
source ~/bio_env/bin/activate
pip install pydeseq2 dendropy
python -m ipykernel install --user --name bio_env --display-name "Python (Bioinformatics)"
deactivate
```

Each kernel runs with:
- **All the default BERDL packages** (PySpark, pandas, etc.)
- **Its own additional specialized packages**
- **Access to BERDL helper functions** (`get_spark_session()`, `display_df()`, etc.)

## Command-Line Tools

To install a Python **command-line tool**, use `pipx` instead of a venv you manage by
hand. It puts each tool in its own environment and its command in `~/.local/bin`:

```bash
pipx install --system-site-packages <package or git+https://… URL>
printf '\nexport PATH="$PATH:$HOME/.local/bin"\n' >> ~/.custom_profile   # once; then open a new terminal
```

`~/.local/bin` is not on your `PATH` until you add it. Add it in `~/.custom_profile`, not
with `pipx ensurepath`: that edits `~/.bashrc` and `~/.bash_profile`, which BERDL replaces
every time your server starts.

`--system-site-packages` lets the tool use the image's packages (PySpark,
`berdl_notebook_utils`). Update a tool with `pipx install --force …` (same arguments) and
remove it with `pipx uninstall <name>`. For KIND, KOROS or an app build you want to test
in KIND, follow the [KIND guide](kind_guide.md#running-your-own-build-of-kind-koros-or-an-app)
instead.

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
pip install --upgrade xgboost
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
   print(sys.executable)   # should be ~/my_venv/bin/python
   %pip show <package>     # "Location:" shows where it is installed
   ```
   Use `%pip`, not `!pip`, in notebooks: `!pip` runs the image's pip, which lists the image's packages.

### Virtual Environment Activation Fails

Make sure you're using the correct path:
```bash
source ~/my_venv/bin/activate
```

If the environment doesn't exist, create it first with `python -m venv --system-site-packages ~/my_venv` (Step 2).

### A package in ~/.local is breaking my notebooks

Symptoms: an import in a notebook fails or behaves differently than in a terminal (for
example a version conflict), or a command in `~/.local/bin` fails in a terminal with
`ModuleNotFoundError`, or a JupyterLab widget or chart renders wrongly. Cause: an older
`%pip install` in a notebook or `pip install --user` left a package in `~/.local` (new ones
are refused now). The default **Python 3 (ipykernel)** kernel loads `~/.local` ahead of the image's packages (your venv kernels don't), and
JupyterLab loads extensions from `~/.local/share/jupyter/labextensions` ahead of the
image's; the notebook server, terminals and KIND ignore `~/.local` Python packages.

1. See what's there:
   ```bash
   ls ~/.local/lib/python*/site-packages/ ~/.local/share/jupyter/labextensions/
   ```
2. Remove the package from a notebook cell running the default **Python 3 (ipykernel)** kernel (pip
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
2. **Document your packages**: Keep a requirements file for each environment:
   ```bash
   source ~/my_venv/bin/activate
   pip freeze --local > ~/my_venv-requirements.txt
   ```
   (`--local` leaves out the packages the venv inherits from the image.)
3. **Clean up unused environments**: Regularly remove environments you no longer use
4. **Request common packages**: If you find yourself installing the same packages repeatedly, contact the BERDL Platform team to have them added to the base image

## Getting Help

If you need packages added to the default BERDL environment or encounter issues with custom kernels, please contact the BERDL Platform team for assistance.
