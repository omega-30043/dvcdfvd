Awesome — here’s a **clear, end-to-end recipe** you can drop into Octopus so a *pre-deployment* step kicks an external workflow, waits for success, and only then continues the deployment.

---

# What you’ll build

* **Step 1 (pre-deployment):** a **Run a Script** step that **triggers** your external workflow (example below uses **GitHub Actions**) and **waits/polls** until it **succeeds** (or fails/times out).
* **Remaining steps:** run **only if** Step 1 exits `0` (success). If the external workflow fails, the deployment stops right there.

> If you use **Jenkins** or **Azure DevOps** instead of GitHub Actions, I included mini scripts for those at the end.

---

# Step-by-step in Octopus

1. **Open your Project → Process → Add Step → Run a Script**

   * **Step name:** `Pre-Deployment: Trigger External Workflow`
   * **Execution location:** **Run once on a worker** (recommended).

     * Use a Windows worker if you’ll paste the PowerShell script below.

2. **Script type:** **PowerShell**

3. **Paste the script** from the “Drop-in PowerShell” section below.

4. **Failure behavior:** leave default (“**Fail the deployment**”) so the process stops if this step exits non-zero.

5. **Run conditions (optional):** scope this step to specific **Environments**, **Channels**, or **Tenants** if needed.

6. **Variables → Project variables (create these):**

   | Variable name                | Type            | Example                                                   | Purpose                                                           |
   | ---------------------------- | --------------- | --------------------------------------------------------- | ----------------------------------------------------------------- |
   | `GitHub.Owner`               | Text            | `my-org`                                                  | GitHub org/user that owns the repo                                |
   | `GitHub.Repo`                | Text            | `my-repo`                                                 | Repository name                                                   |
   | `GitHub.WorkflowId`          | Text            | `deploy.yml`                                              | Workflow filename (or numeric ID)                                 |
   | `GitHub.Ref`                 | Text            | `main`                                                    | Branch to dispatch (e.g., `main`)                                 |
   | `GitHub.WorkflowInputs`      | Text (optional) | `{"env":"staging","version":"#{Octopus.Release.Number}"}` | **JSON string** passed as workflow `inputs`                       |
   | `GitHub.PollIntervalSeconds` | Text/Number     | `15`                                                      | Seconds between status checks                                     |
   | `GitHub.MaxWaitMinutes`      | Text/Number     | `30`                                                      | Max time to wait before timeout                                   |
   | `GitHub.FailOnCancelled`     | Text/Bool       | `True`                                                    | Treat “cancelled” as failure (`True`) or allow continue (`False`) |
   | `GitHub.Api.Token`           | **Sensitive**   | (PAT)                                                     | **GitHub token** with `repo` + `workflow` scopes                  |

   > Mark `GitHub.Api.Token` as **Sensitive** in Octopus.
   > You can **scope** any of these variables by Environment if values differ per env.

7. **Save**, create a **Release**, and **Deploy**. Step 1 will gate the rest.

---

# Drop-in PowerShell (GitHub Actions)

> Works on an Octopus **Windows worker** with outbound HTTPS to `api.github.com`.
> This script: **dispatches** the workflow, **finds the exact run** created by this dispatch, **polls** until it completes, and **fails** the deployment on non-success results.

```powershell
# ====== Read inputs from Octopus variables ======
$owner            = $OctopusParameters['GitHub.Owner']
$repo             = $OctopusParameters['GitHub.Repo']
$workflowId       = $OctopusParameters['GitHub.WorkflowId'] # e.g., deploy.yml or numeric ID
$ref              = $OctopusParameters['GitHub.Ref']        # e.g., main
$inputsJson       = $OctopusParameters['GitHub.WorkflowInputs'] # optional JSON
$token            = $OctopusParameters['GitHub.Api.Token']  # Sensitive

$pollSeconds      = [int]($OctopusParameters['GitHub.PollIntervalSeconds'] ?? 15)
$maxWaitMinutes   = [int]($OctopusParameters['GitHub.MaxWaitMinutes'] ?? 30)
$failOnCancelled  = ($OctopusParameters['GitHub.FailOnCancelled'] ?? 'True') -eq 'True'

# ====== Constants & helpers ======
$baseUrl = "https://api.github.com"
$headers = @{
  Authorization = "token $token"
  Accept        = "application/vnd.github+json"
  "User-Agent"  = "octopus-predeploy"
}

function Invoke-Gh {
  param([string]$Method,[string]$Uri,[object]$Body=$null)
  try {
    if ($Body -ne $null) {
      $json = $Body | ConvertTo-Json -Depth 10
      return Invoke-RestMethod -Method $Method -Uri $Uri -Headers $headers -Body $json
    } else {
      return Invoke-RestMethod -Method $Method -Uri $Uri -Headers $headers
    }
  }
  catch {
    Write-Error "GitHub API call failed: $Method $Uri`n$($_.Exception.Message)"
    throw
  }
}

# ====== 1) Trigger (dispatch) ======
$dispatchUri = "$baseUrl/repos/$owner/$repo/actions/workflows/$workflowId/dispatches"
$payload = @{ ref = $ref }
if ($inputsJson) {
  try { $payload.inputs = ($inputsJson | ConvertFrom-Json) }
  catch {
    Write-Warning "GitHub.WorkflowInputs is not valid JSON; sending as raw string."
    $payload.inputs = @{ _raw = $inputsJson }
  }
}

$dispatchAt = Get-Date
Invoke-Gh -Method POST -Uri $dispatchUri -Body $payload | Out-Null
Write-Host "Triggered workflow dispatch at $dispatchAt (ref: $ref)."

# ====== 2) Identify the run created by this dispatch ======
$timeoutAt = (Get-Date).AddMinutes($maxWaitMinutes)
$runId = $null
$runHtml = $null

while (-not $runId -and (Get-Date) -lt $timeoutAt) {
  $runsUri = "$baseUrl/repos/$owner/$repo/actions/workflows/$workflowId/runs?event=workflow_dispatch&branch=$ref&per_page=10"
  $list = Invoke-Gh -Method GET -Uri $runsUri

  # Find the newest run created at/after we dispatched
  $candidate = $list.workflow_runs | Sort-Object {[datetime]$_.created_at} -Descending |
    Where-Object { (Get-Date $_.created_at) -ge $dispatchAt.AddMinutes(-1) } | Select-Object -First 1

  if ($candidate) {
    $runId  = $candidate.id
    $runHtml = $candidate.html_url
  } else {
    Start-Sleep -Seconds $pollSeconds
  }
}

if (-not $runId) {
  throw "Timed out before locating the workflow run created by this dispatch."
}

# Expose a handy link to this run for later steps / task summary
Set-OctopusVariable -name "ExternalWorkflow.Url" -value $runHtml
Write-Host "External workflow run: $runHtml"

# ====== 3) Poll until completion ======
while ((Get-Date) -lt $timeoutAt) {
  $run = Invoke-Gh -Method GET -Uri "$baseUrl/repos/$owner/$repo/actions/runs/$runId"
  $status = $run.status        # queued | in_progress | completed
  $conclusion = $run.conclusion # success | failure | neutral | cancelled | skipped | timed_out | action_required

  Write-Host "Status: $status, Conclusion: $conclusion"

  if ($status -eq 'completed') {
    if ($conclusion -eq 'success') {
      Write-Host "External workflow succeeded."
      exit 0
    }
    elseif ($conclusion -eq 'cancelled' -and -not $failOnCancelled) {
      Write-Warning "Workflow cancelled, but FailOnCancelled=False; allowing deployment to continue."
      exit 0
    }
    else {
      throw "External workflow concluded as '$conclusion'. Failing deployment."
    }
  }

  Start-Sleep -Seconds $pollSeconds
}

throw "Timed out after $maxWaitMinutes minute(s) waiting for external workflow."
```

### What each variable does (quick reference)

* **`GitHub.Owner` / `GitHub.Repo`**: points the script at the correct repository.
* **`GitHub.WorkflowId`**: the **workflow file name** (e.g., `deploy.yml`) or its **numeric ID**.
* **`GitHub.Ref`**: the branch to dispatch (e.g., `main`).
* **`GitHub.WorkflowInputs`**: optional JSON to populate your workflow `inputs`. Example:

  ```json
  {"environment":"staging","version":"#{Octopus.Release.Number}"}
  ```

  (Notice you can inject Octopus variables!)
* **`GitHub.Api.Token`** (**Sensitive**): a PAT with **`repo` + `workflow`** scopes.
* **`GitHub.PollIntervalSeconds`**: how frequently to check run status.
* **`GitHub.MaxWaitMinutes`**: how long to wait before failing with timeout.
* **`GitHub.FailOnCancelled`**: if `True`, a **cancelled** run fails the deployment; if `False`, Octopus will continue.

### Using the output later

* The script exposes `ExternalWorkflow.Url` as an output variable.
  In later steps you can reference:

  ```
  #{Octopus.Action[Pre-Deployment: Trigger External Workflow].Output.ExternalWorkflow.Url}
  ```

  to print or link to the workflow run page.

---

## Optional: Bash version (Linux worker)

If you prefer a Linux worker, I can provide a curl/jq Bash version—just say the word.

---

# Alternatives (quick recipes)

## A) Jenkins

**Variables to add:**

* `Jenkins.BaseUrl` (e.g., `https://jenkins.acme.local`)
* `Jenkins.JobPath` (e.g., `my-folder/my-job`)
* `Jenkins.User` (or use a token-only user)
* `Jenkins.ApiToken` (**Sensitive**)
* `Jenkins.Params` (optional query string like `BRANCH=main&ENV=staging`)
* `Jenkins.InsecureSkipTls` (`True/False`) for self-signed certs
* `PollIntervalSeconds`, `MaxWaitMinutes`

**PowerShell gist (handles queue → build → result):**

```powershell
$base   = $OctopusParameters['Jenkins.BaseUrl'].TrimEnd('/')
$job    = $OctopusParameters['Jenkins.JobPath']
$user   = $OctopusParameters['Jenkins.User']
$token  = $OctopusParameters['Jenkins.ApiToken']
$params = $OctopusParameters['Jenkins.Params']
$poll   = [int]($OctopusParameters['PollIntervalSeconds'] ?? 10)
$maxMin = [int]($OctopusParameters['MaxWaitMinutes'] ?? 30)
$insec  = ($OctopusParameters['Jenkins.InsecureSkipTls'] ?? 'False') -eq 'True'
$timeoutAt = (Get-Date).AddMinutes($maxMin)

if ($insec) { add-type @"
using System.Net;
using System.Security.Cryptography.X509Certificates;
public class TrustAllCertsPolicy : ICertificatePolicy {
  public bool CheckValidationResult(ServicePoint srvPoint, X509Certificate certificate, WebRequest request, int certificateProblem) { return true; }
}
"@; [System.Net.ServicePointManager]::CertificatePolicy = New-Object TrustAllCertsPolicy }

$triggerUrl = "$base/job/$job/build" + ($(if($params){"WithParameters?$params"}else{""}))
$cred = ("{0}:{1}" -f $user,$token)
$bytes = [System.Text.Encoding]::UTF8.GetBytes($cred)
$basic = "Basic " + [Convert]::ToBase64String($bytes)

# Trigger
$resp = Invoke-WebRequest -Method POST -Uri $triggerUrl -Headers @{Authorization=$basic} -AllowUnencryptedAuthentication -ErrorAction Stop
$queueUrl = $resp.Headers['Location']

# Wait for executable (build number)
$buildUrl = $null
while (-not $buildUrl -and (Get-Date) -lt $timeoutAt) {
  $q = Invoke-RestMethod -Method GET -Uri ($queueUrl + "api/json") -Headers @{Authorization=$basic}
  if ($q.executable) { $buildUrl = $q.executable.url; break }
  Start-Sleep -Seconds $poll
}

if (-not $buildUrl) { throw "Timed out waiting for Jenkins build to start." }
Write-Host "Jenkins build: $buildUrl"
Set-OctopusVariable -name "ExternalWorkflow.Url" -value $buildUrl

# Wait for result
while ((Get-Date) -lt $timeoutAt) {
  $b = Invoke-RestMethod -Method GET -Uri ($buildUrl + "api/json") -Headers @{Authorization=$basic}
  if (-not $b.building) {
    if ($b.result -eq "SUCCESS") { exit 0 } else { throw "Jenkins result: $($b.result)" }
  }
  Start-Sleep -Seconds $poll
}
throw "Timed out waiting for Jenkins result."
```

> If you must skip TLS verification, the snippet above shows one (risky) way. Prefer fixing certs/proxy if possible.

## B) Azure DevOps Pipelines

**Variables:**

* `AzDo.Org` (`myorg`)
* `AzDo.Project` (`myproject`)
* `AzDo.PipelineId` (numeric)
* `AzDo.Branch` (`refs/heads/main`)
* `AzDo.PAT` (**Sensitive**)
* `PollIntervalSeconds`, `MaxWaitMinutes`

**PowerShell gist:**

```powershell
$org   = $OctopusParameters['AzDo.Org']
$proj  = $OctopusParameters['AzDo.Project']
$pid   = $OctopusParameters['AzDo.PipelineId']
$branch= $OctopusParameters['AzDo.Branch']
$pat   = $OctopusParameters['AzDo.PAT']
$poll  = [int]($OctopusParameters['PollIntervalSeconds'] ?? 15)
$max   = [int]($OctopusParameters['MaxWaitMinutes'] ?? 30)
$timeoutAt = (Get-Date).AddMinutes($max)

$base = "https://dev.azure.com/$org/$proj/_apis"
$auth = "Basic " + [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$pat"))
$h = @{ Authorization = $auth; "Content-Type"="application/json" }

# Trigger
$run = Invoke-RestMethod -Method POST -Uri "$base/pipelines/$pid/runs?api-version=7.1-preview.1" -Headers $h -Body (@{
  resources = @{ repositories = @{ self = @{ refName = $branch } } }
} | ConvertTo-Json -Depth 10)

$runId = $run.id
$runUrl = $run._links.web.href
Set-OctopusVariable -name "ExternalWorkflow.Url" -value $runUrl
Write-Host "ADO run: $runUrl"

# Wait
while ((Get-Date) -lt $timeoutAt) {
  $r = Invoke-RestMethod -Method GET -Uri "$base/pipelines/$pid/runs/$runId?api-version=7.1-preview.1" -Headers $h
  if ($r.state -eq 'completed') {
    if ($r.result -eq 'succeeded') { exit 0 } else { throw "ADO result: $($r.result)" }
  }
  Start-Sleep -Seconds $poll
}
throw "Timed out waiting for ADO pipeline."
```

---

## Tips & gotchas

* **Worker networking:** the Octopus worker must reach GitHub/Jenkins/Azure DevOps through your proxy/firewall.
* **Permissions:** your token must have rights to **trigger** and **read** run status.
* **Time boxing:** adjust `MaxWaitMinutes` to fit your workflow’s typical duration.
* **Visibility:** the script publishes a link (`ExternalWorkflow.Url`) you can echo in later steps/logs.
* **Branch/inputs matching:** ensure your external workflow is configured to accept `workflow_dispatch` with any required `inputs`.

If you tell me which system you’re using (GitHub/Jenkins/Azure DevOps) and any constraints (proxy, self-signed cert, Linux vs Windows worker), I can tailor the exact script and variables to your environment.
# dvcdfvd
