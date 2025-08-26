Perfect 👍 — let’s go deep and walk through **exactly** how you would set this up in Octopus Deploy, step by step, using the values you listed.

We’ll use the **Step Template** approach so you can reuse it in many projects.

---

# 🔹 Part 1 – Create the Step Template

1. **Navigate:**
   Octopus → **Library** → **Step Templates** → **Add Step Template**.
   Choose **Run a Script**.

2. **General Info:**

   * **Name:** `Pre-Deployment: Trigger GitHub Workflow`
   * **Description:** “Triggers a GitHub Actions workflow and waits until it succeeds before continuing deployment.”
   * **Execution Location:** Run once on a worker (Windows worker if using PowerShell).

3. **Script Section:**
   Paste this (simplified but complete) PowerShell:

   ```powershell
   # Read inputs from step template parameters
   $owner            = $OctopusParameters['StepTemplate.GitHub.Owner']
   $repo             = $OctopusParameters['StepTemplate.GitHub.Repo']
   $workflowId       = $OctopusParameters['StepTemplate.GitHub.WorkflowId']
   $ref              = $OctopusParameters['StepTemplate.GitHub.Ref']
   $token            = $OctopusParameters['StepTemplate.GitHub.ApiToken']
   $pollSeconds      = [int]$OctopusParameters['StepTemplate.GitHub.PollIntervalSeconds']
   $maxWaitMinutes   = [int]$OctopusParameters['StepTemplate.GitHub.MaxWaitMinutes']
   $failOnCancelled  = ($OctopusParameters['StepTemplate.GitHub.FailOnCancelled'] -eq 'True')

   $baseUrl = "https://api.github.com"
   $headers = @{
     Authorization = "token $token"
     Accept        = "application/vnd.github+json"
     "User-Agent"  = "octopus-predeploy"
   }

   Write-Host "Triggering GitHub workflow $workflowId on $owner/$repo (ref: $ref)..."

   # 1. Trigger workflow
   $dispatchUri = "$baseUrl/repos/$owner/$repo/actions/workflows/$workflowId/dispatches"
   $body = @{ ref = $ref } | ConvertTo-Json
   Invoke-RestMethod -Method POST -Uri $dispatchUri -Headers $headers -Body $body
   $dispatchAt = Get-Date

   # 2. Find the workflow run
   $timeoutAt = (Get-Date).AddMinutes($maxWaitMinutes)
   $runId = $null
   while (-not $runId -and (Get-Date) -lt $timeoutAt) {
     $runs = Invoke-RestMethod -Uri "$baseUrl/repos/$owner/$repo/actions/runs?event=workflow_dispatch&branch=$ref&per_page=1" -Headers $headers
     if ($runs.workflow_runs.Count -gt 0) {
       $candidate = $runs.workflow_runs[0]
       if ((Get-Date $candidate.created_at) -ge $dispatchAt.AddMinutes(-1)) {
         $runId = $candidate.id
         $runUrl = $candidate.html_url
         Set-OctopusVariable -name "ExternalWorkflow.Url" -value $runUrl
         Write-Host "Workflow run found: $runUrl"
       }
     }
     Start-Sleep -Seconds $pollSeconds
   }

   if (-not $runId) {
     throw "Could not locate workflow run within $maxWaitMinutes minutes."
   }

   # 3. Poll until completion
   while ((Get-Date) -lt $timeoutAt) {
     $run = Invoke-RestMethod -Uri "$baseUrl/repos/$owner/$repo/actions/runs/$runId" -Headers $headers
     Write-Host "Status: $($run.status), Conclusion: $($run.conclusion)"

     if ($run.status -eq "completed") {
       if ($run.conclusion -eq "success") {
         Write-Host "Workflow succeeded ✅"
         exit 0
       }
       elseif ($run.conclusion -eq "cancelled" -and -not $failOnCancelled) {
         Write-Warning "Workflow cancelled, but FailOnCancelled=False; continuing."
         exit 0
       }
       else {
         throw "Workflow failed with conclusion: $($run.conclusion)"
       }
     }
     Start-Sleep -Seconds $pollSeconds
   }

   throw "Workflow did not complete in $maxWaitMinutes minutes."
   ```

---

# 🔹 Part 2 – Define Template Parameters

In the **Parameters** tab of the template, create these (all **string type**, with **Sensitive** only for tokens):

| Name                         | Label                            | Default | Sensitive? |
| ---------------------------- | -------------------------------- | ------- | ---------- |
| `GitHub.Owner`               | GitHub Owner                     | –       | No         |
| `GitHub.Repo`                | GitHub Repo                      | –       | No         |
| `GitHub.WorkflowId`          | Workflow File (e.g., deploy.yml) | –       | No         |
| `GitHub.Ref`                 | Branch (e.g., main)              | `main`  | No         |
| `GitHub.ApiToken`            | GitHub PAT                       | –       | ✅ Yes      |
| `GitHub.PollIntervalSeconds` | Poll Interval (sec)              | `15`    | No         |
| `GitHub.MaxWaitMinutes`      | Max Wait (minutes)               | `30`    | No         |
| `GitHub.FailOnCancelled`     | Fail if Cancelled (True/False)   | `True`  | No         |

---

# 🔹 Part 3 – Add to Your Project

1. Go to your **Project → Process → Add Step → Add Step Template**.
2. Select `Pre-Deployment: Trigger GitHub Workflow`.
3. Drag it to the **first step** of your process.

---

# 🔹 Part 4 – Fill in Parameters for Your Case

When you add the template, you’ll see parameter fields. Fill them like this:

* **Owner:** `my-org`
* **Repo:** `my-repo`
* **WorkflowId:** `deploy.yml`
* **Ref:** `main`
* **ApiToken:** choose the sensitive variable you created (PAT with `repo` + `workflow` scopes)
* **PollInterval:** `20`
* **MaxWait:** `60`
* **FailOnCancelled:** `True`

---

# 🔹 Part 5 – Run It

* When you deploy, Octopus will:

  1. Trigger the GitHub Actions workflow (`deploy.yml` on `main`).
  2. Wait up to **60 minutes**, polling every **20 seconds**.
  3. If workflow **succeeds → continue** to the rest of deployment.
  4. If workflow **fails/cancelled (with FailOnCancelled=True) → stop** the deployment.
  5. Output variable `ExternalWorkflow.Url` contains the GitHub Actions run link (you can reference it later as `#{Octopus.Action[Pre-Deployment: Trigger GitHub Workflow].Output.ExternalWorkflow.Url}`).

---

✅ With this setup:

* It’s reusable (step template).
* Secure (PAT is Sensitive).
* Configurable (parameters).
* Scoped (you can scope PATs to specific environments).

---

Would you like me to also prepare an **exportable JSON of this Step Template** (so you can import directly into Octopus instead of creating it manually)?
