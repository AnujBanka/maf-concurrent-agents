# Setup guide: Concurrent multi-agent pipeline (Microsoft Agent Framework + Azure AI Foundry)

These steps take you from a fresh machine to a working three-agent concurrent pipeline, and then delete everything again.
Run every command in a **PowerShell** terminal opened at the repo folder (`E:\VSTS\AI103\unit_code_maf_concurrent`).

> The `scripts\*.sh` files hold PowerShell commands (the backtick `` ` `` continues a line), not bash.
> Paste them into PowerShell one block at a time. Don't run them as scripts.

---

## What you are building

```
                 ┌─► summarizer-agent ─┐
You ─► input ────┼─► researcher-agent ─┼─► combined answers
                 └─► critic-agent ─────┘
                 (all three run in parallel)
```

| Piece | File | What it does |
|---|---|---|
| Infrastructure | `resource_deployment.bicep` | Creates a Foundry resource, a Foundry project, and a `gpt-5-mini` model deployment |
| Agent definitions | `config.yaml` | The name and system prompt for each agent (one active at a time) |
| Agent registration | `agent_deployment.py` | Registers the active agent in `config.yaml` with the Foundry project |
| Pipeline | `agent_execution.py` | Runs the three agents in parallel with `ConcurrentBuilder` and opens an interactive chat |
| Run configs | `.vscode/launch.json` | F5 run configurations with the environment variables already filled in |

---

## What you need

- **Python 3.10 or newer.** Check with `python --version`.
- **Azure CLI.** Install it from <https://aka.ms/installazurecli> and restart VS Code. Check with `az version`.
- **An Azure subscription** where you are Owner or Contributor. This guide used *Visual Studio Professional*.
- **Quota for `gpt-5-mini`, GlobalStandard, in the region you choose.** The Bicep file asks for capacity 50.
- **You never need API keys.** The Python code uses `DefaultAzureCredential`, which reuses your `az login` session.

---

## Step 0: Pick your names (do this once)

The Foundry resource name becomes a public subdomain (`<prefix>.services.ai.azure.com`), so it must be
**unique across all of Azure**. `courseunit623` from the course material is likely taken.

Set these variables at the start of **every new terminal session**. The commands later in this guide use them.

```powershell
$prefix   = "anujunit621"          # Foundry resource name (globally unique, lowercase letters and numbers)
$rg       = "aiagent-course-rg"    # resource group name
$location = "australiaeast"        # Azure region
```

If you change `$prefix`, also change it in:
- `.vscode/launch.json` (both `PROJECT_ENDPOINT` values and `LLM_MODEL_DEPLOYMENT_NAME`)
- `scripts/01_resource_deployment.sh`
- `scripts/02_resource_deletion.sh`

Check a name is free (run this after `az login` in Step 2):
```powershell
$sub = az account show --query id -o tsv
'{"subdomainName":"' + $prefix + '","type":"Microsoft.CognitiveServices/accounts","kind":"AIServices"}' | Set-Content -Encoding utf8 "$env:TEMP\chk.json"
az rest --method post --url "https://management.azure.com/subscriptions/$sub/providers/Microsoft.CognitiveServices/checkDomainAvailability?api-version=2024-10-01" --body "@$env:TEMP\chk.json" --query isSubdomainAvailable
```
If it prints `true`, the name is free.

---

## Step 1: Python environment (do this once per machine)

```powershell
python -m venv tutorials            # create a private Python environment in .\tutorials
tutorials\Scripts\activate          # use it in this terminal; the prompt now starts with (tutorials)
pip install -r requirements.txt     # takes several minutes; agent-framework pulls in about 230 packages
```

- **The environment must be named `tutorials`.** `launch.json` points at `tutorials/Scripts/python.exe`.
- **Activation only lasts for the current terminal.** Run `tutorials\Scripts\activate` again in every new terminal.
- **If you see "running scripts is disabled"**, allow scripts for your user once, then activate again:
  `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`
- **Skip `pip freeze > requirements.txt`.** It replaces the short package list with a long pinned one.
- **Optional:** in VS Code, run *Python: Select Interpreter* (Ctrl+Shift+P) and pick `.\tutorials\Scripts\python.exe` so the imports aren't underlined as errors.

Check the install:
```powershell
python -c "import yaml, azure.identity, azure.ai.projects; from agent_framework.orchestrations import ConcurrentBuilder; from agent_framework.foundry import FoundryAgent; print('ALL IMPORTS OK')"
```

---

## Step 2: Sign in to Azure

```powershell
az login                                                             # browser sign-in
az account show --query "{name:name, user:user.name}" -o table       # check you're on the right subscription
# az account set --subscription "<name or id>"                       # switch if you need to
az bicep install                                                     # or: az bicep upgrade
```

Optional checks before deploying:
```powershell
# Is the AI services provider registered on your subscription? Expect: Registered
az provider show -n Microsoft.CognitiveServices --query registrationState -o tsv

# Is there quota for gpt-5-mini in your region? You need at least 50 free (limit minus used)
az cognitiveservices usage list -l $location -o json | ConvertFrom-Json |
  Where-Object { $_.name.value -like '*gpt-5-mini*' } |
  ForEach-Object { "{0} used={1} limit={2}" -f $_.name.value, $_.currentValue, $_.limit }
```

---

## Step 3: Deploy the Azure resources (`scripts/01_resource_deployment.sh`)

```powershell
# 3a. Resource group: one container for everything, so one delete removes it all
az group create --name $rg --location $location

# 3b. Deploy the Bicep file (2–5 minutes)
az deployment group create `
  --resource-group $rg `
  --template-file resource_deployment.bicep `
  --parameters coursePrefix=$prefix

# 3c. Show the outputs
az deployment group show --resource-group $rg --name resource_deployment --query properties.outputs
```

Expected outputs:
- `project_endpoint` = `https://<prefix>.services.ai.azure.com/api/projects/<prefix>-proj`
- `llm_model_deployment_name` = `<prefix>-llm-deploy`

These must match the values in `.vscode/launch.json`.

Resources created:

| Resource | Name |
|---|---|
| Foundry account (kind `AIServices`) | `<prefix>` |
| Foundry project | `<prefix>-proj` |
| Model deployment (`gpt-5-mini`, GlobalStandard, capacity 50) | `<prefix>-llm-deploy` |

Idle resources cost nothing. You pay per token when the agents run.

---

## Step 4: Give yourself access inside Foundry (role assignment)

Subscription **Owner** lets you create Azure resources, but not create agents or call the model.
You need the **Foundry User** role on the Foundry resource. Older guides call it *"Azure AI User"*; that name
no longer exists, so the command below uses the role's ID, which hasn't changed.

```powershell
$me = az ad signed-in-user show --query id -o tsv
$id = az cognitiveservices account show -g $rg -n $prefix --query id -o tsv
az role assignment create --assignee $me --role 53ca6127-db72-4b80-b1b0-d745d6d5456d --scope $id
```

**Wait 2–5 minutes** for the role to take effect. If the next step fails with 401, 403 or `PermissionDenied`,
wait a little longer and try again. (In this run, the role was active within a few minutes.)

---

## Step 5: Register the three agents (`agent_deployment.py` × 3)

`agent_deployment.py` registers **only the active (uncommented) block** in `config.yaml`, so you run it three times.

Set the environment variables once in the terminal:
```powershell
$env:PROJECT_ENDPOINT          = "https://$prefix.services.ai.azure.com/api/projects/$prefix-proj"
$env:LLM_MODEL_DEPLOYMENT_NAME = "$prefix-llm-deploy"
```

Then, for each agent:

| Run | Active block in `config.yaml` | Command | Expected output |
|---|---|---|---|
| 1 | `summarizer-agent` | `python agent_deployment.py` | `✅ ... (ID: summarizer-agent:1)` |
| 2 | `researcher-agent` | `python agent_deployment.py` | `✅ ... (ID: researcher-agent:1)` |
| 3 | `critic-agent` | `python agent_deployment.py` | `✅ ... (ID: critic-agent:1)` |

Between runs, edit `config.yaml`: put `# ` in front of each line of the block you just deployed, and remove `# `
from the next block. Exactly one `agent_name:` / `system_prompt:` pair may be uncommented at a time.

- **Using VS Code instead of the terminal:** choose **Python Debugger: Agent Deployment** in Run and Debug and press F5. The environment variables come from `launch.json`.
- **Changing an agent:** edit its prompt and run the script again. Foundry keeps the name and creates a new version (`:2`, `:3`, …). The pipeline always uses the latest version.
- **Seeing your agents:** they appear in the [Foundry portal](https://ai.azure.com), under your project's **Agents** page.

---

## Step 6: Run the pipeline (`agent_execution.py`)

```powershell
$env:PROJECT_ENDPOINT      = "https://$prefix.services.ai.azure.com/api/projects/$prefix-proj"
$env:SUMMARIZER_AGENT_NAME = "summarizer-agent"
$env:RESEARCHER_AGENT_NAME = "researcher-agent"
$env:CRITIC_AGENT_NAME     = "critic-agent"
python agent_execution.py
```

Or use F5 with **Python Debugger: Agent Execution**.

Type a request at `You:`, for example *"Our team plans to migrate our job portal to Azure next quarter."*, and type `exit` to stop.
All three agents receive the same input and answer in parallel. Each response starts by naming the agent that wrote it.

The model's answers vary from run to run. If an agent's answer is off, make its prompt in `config.yaml` more specific and redeploy that agent (Step 5).

---

## Step 7: Delete everything (`scripts/02_resource_deletion.sh`)

Do this when you finish, so the resources stop counting against your subscription.

```powershell
# Delete the resource group and everything in it (a few minutes)
az group delete --name $rg --yes

# The Foundry account is only soft-deleted. List soft-deleted accounts:
az cognitiveservices account list-deleted --output table

# Purge it permanently. Without this, you can't reuse the same name for about 48 hours.
az cognitiveservices account purge --location $location --resource-group $rg --name $prefix
```

To start again later, go back to **Step 2**. You can skip Step 1 because `tutorials\` already exists; just activate it.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `running scripts is disabled on this system` | PowerShell blocks `activate` | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |
| Bicep deploy fails: subdomain or name already in use | `$prefix` is taken, or your own soft-deleted account still holds it | Pick a new prefix, or purge the old account (Step 7) |
| Bicep deploy fails: quota | No `gpt-5-mini` GlobalStandard capacity in that region | Use another `$location`, or lower `capacity` in the Bicep file |
| `Role 'Azure AI User' doesn't exist` | The role was renamed to **Foundry User** | Use the role ID `53ca6127-db72-4b80-b1b0-d745d6d5456d` (Step 4) |
| 401, 403 or `PermissionDenied` from the Python scripts | Role missing or not applied yet | Do Step 4, then wait 2–5 minutes |
| `DefaultAzureCredential failed to retrieve a token` | Not signed in, or the sign-in expired | `az login` |
| Pipeline error: agent not found | One of the three agents isn't registered | Do Step 5 for the missing agent |
| `ModuleNotFoundError` | The `tutorials` environment isn't active, or the install didn't finish | `tutorials\Scripts\activate`, then `pip install -r requirements.txt` |

---

## Quick reference: full rebuild after a cleanup

```powershell
cd E:\VSTS\AI103\unit_code_maf_concurrent
tutorials\Scripts\activate
$prefix = "anujunit621"; $rg = "aiagent-course-rg"; $location = "australiaeast"

az login
az group create --name $rg --location $location
az deployment group create --resource-group $rg --template-file resource_deployment.bicep --parameters coursePrefix=$prefix

$me = az ad signed-in-user show --query id -o tsv
$id = az cognitiveservices account show -g $rg -n $prefix --query id -o tsv
az role assignment create --assignee $me --role 53ca6127-db72-4b80-b1b0-d745d6d5456d --scope $id
# wait 2–5 minutes

$env:PROJECT_ENDPOINT          = "https://$prefix.services.ai.azure.com/api/projects/$prefix-proj"
$env:LLM_MODEL_DEPLOYMENT_NAME = "$prefix-llm-deploy"
python agent_deployment.py   # summarizer active in config.yaml
# switch config.yaml to researcher, then:
python agent_deployment.py
# switch config.yaml to critic, then:
python agent_deployment.py
# switch config.yaml back to summarizer

$env:SUMMARIZER_AGENT_NAME = "summarizer-agent"; $env:RESEARCHER_AGENT_NAME = "researcher-agent"; $env:CRITIC_AGENT_NAME = "critic-agent"
python agent_execution.py

# when finished:
az group delete --name $rg --yes
az cognitiveservices account purge --location $location --resource-group $rg --name $prefix
```
