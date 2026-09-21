# Deploy-Prod Workflow Troubleshooting Log — 2026-08-22

Context: Working through the "Deploy the model to a real-time endpoint from a pull
request comment" section of the mslearn-mlops lab
(`docs/07-deploy-monitor.md`). The `/deploy-prod` comment trigger runs
`.github/workflows/deploy-prod.yml`, which executes
`src/deploy_to_online_endpoint.py` against a managed online endpoint named
`diabetes-endpoint-8ab6a8cf`.

Three issues were hit and diagnosed in this session, in order. Two are fixed.
One (null predictions) is still open — see "Open issue" at the bottom.

---

## Issue 1 (ruled out): VM SKU / quota hypothesis

**Initial hypothesis (incorrect):** `create_or_update_deployment()` requests
`instance_type="Standard_D2as_v4"`, and `infra/setup.sh` only provisions
`STANDARD_DS11_V2` compute, so I first suspected a quota mismatch between the
Dasv4 family and what the lab subscription actually has quota for.

**Why it was ruled out:** The actual traceback (see Issue 2) showed the
failure happening earlier, in `ensure_endpoint()` during endpoint creation —
before the deployment/SKU step was ever reached. No SKU or quota error was
ever produced. This hypothesis was abandoned once the real traceback was
retrieved from the GitHub Actions step logs.

**Lesson:** Always get the actual step log/traceback from Actions before
trusting a plausible-looking hypothesis based on code inspection alone.
How to get it: Actions tab → failed run → `deploy-prod` job → expand the
"Deploy model to managed online endpoint" step (not just the run summary).

---

## Issue 2 (fixed): `SubscriptionNotRegistered`

**Error:**
```
azure.core.exceptions.HttpResponseError: (SubscriptionNotRegistered) Resource provider [N/A] isn't registered with Subscription [N/A]. Please see troubleshooting guide, available here: https://aka.ms/register-resource-provider
Code: SubscriptionNotRegistered
```
Raised from `ensure_endpoint()` →
`ml_client.online_endpoints.begin_create_or_update(endpoint).result()`.

**Root cause:** Creating a managed online endpoint for the first time in a
subscription requires several Azure resource providers beyond
`Microsoft.MachineLearningServices` (notably `Microsoft.PolicyInsights` and
`Microsoft.Cdn`). Registering a resource provider is a **subscription-level**
operation. The GitHub Actions service principal (`AZURE_CREDENTIALS` secret)
was created with Contributor scoped only to the resource group:
```bash
az ad sp create-for-rbac --name "<name>" --role contributor \
    --scopes /subscriptions/<subscription-id>/resourceGroups/<your-resource-group-name>
```
That RG-scoped identity can't trigger provider auto-registration at the
subscription level, so Azure returned `SubscriptionNotRegistered` instead of
silently registering the provider.

**Fix applied** — registered the missing providers using an admin/Owner
account (not the GitHub Actions SP) via Cloud Shell / Azure CLI:
```azurecli
az account set --subscription <your-subscription-id>
az provider register --namespace Microsoft.PolicyInsights
az provider register --namespace Microsoft.Cdn
```
Verify registration completed before retrying (can take a few minutes):
```azurecli
az provider show -n Microsoft.PolicyInsights --query registrationState -o tsv
az provider show -n Microsoft.Cdn --query registrationState -o tsv
```
Both must show `Registered`. If you hit this again with provider still
showing `[N/A]`, list every unregistered provider at once:
```azurecli
az provider list --query "[?registrationState!='Registered'].{Provider:namespace, State:registrationState}" -o table
```

After registration completed, re-posting `/deploy-prod` on the PR triggered
the workflow again.

---

## Issue 3 (fixed): `BadRequest` — endpoint not created successfully

**Error (after Issue 2 was fixed and the workflow re-ran):**
```
azure.core.exceptions.HttpResponseError: (BadRequest) The request is invalid.
Code: BadRequest
...
"errors":{"":["This endpoint has not been created successfully or is in deleting provisioning state. Please recreate endpoint and then try again to create deployment."]}
...
Code: InferencingClientCallFailed
```
Raised from `create_or_update_deployment()` →
`ml_client.online_deployments.begin_create_or_update(deployment).result()`,
*after* the model upload step succeeded.

**Root cause:** The endpoint `diabetes-endpoint-8ab6a8cf` already existed as
an Azure resource — it was created (or partially created) during the earlier
run that failed with `SubscriptionNotRegistered` — but it was left in a
non-`Succeeded` provisioning state (failed/stuck).

The bug: `ensure_endpoint()` in `src/deploy_to_online_endpoint.py` only checks
whether the endpoint *exists* via `.get()`, and never checks
`provisioning_state`:
```python
def ensure_endpoint(ml_client: MLClient, endpoint_name: str) -> ManagedOnlineEndpoint:
    try:
        return ml_client.online_endpoints.get(name=endpoint_name)
    except ResourceNotFoundError:
        ...
```
So the script found the broken endpoint, assumed it was healthy, and
proceeded straight to deployment creation, which Azure rejected.

**Fix applied (command-line, no code change yet):** deleted the broken
endpoint so the next `/deploy-prod` run would create a fresh, healthy one:
```azurecli
az ml online-endpoint delete --name diabetes-endpoint-8ab6a8cf \
  --resource-group <your-rg> --workspace-name <your-ws> --yes --no-wait
```
Waited for deletion to fully complete (confirmed via
`az ml online-endpoint show -n diabetes-endpoint-8ab6a8cf ...` returning a
404/not-found), then re-commented `/deploy-prod` on the PR. The workflow
recreated the endpoint from scratch and the deployment succeeded this time.

**Not yet applied — recommended code fix for next session**, to make the
script self-healing instead of relying on manual CLI cleanup:
```python
def ensure_endpoint(ml_client: MLClient, endpoint_name: str) -> ManagedOnlineEndpoint:
    try:
        endpoint = ml_client.online_endpoints.get(name=endpoint_name)
        if endpoint.provisioning_state != "Succeeded":
            ml_client.online_endpoints.begin_delete(name=endpoint_name).result()
            raise ResourceNotFoundError("Endpoint exists but is not in a usable state")
    except ResourceNotFoundError:
        endpoint = ManagedOnlineEndpoint(
            name=endpoint_name,
            description="Online endpoint for MLflow diabetes model",
            auth_mode="key",
        )
        return ml_client.online_endpoints.begin_create_or_update(endpoint).result()
    return endpoint
```
This was proposed but **not applied to the file** during this session —
still a TODO if it recurs.

---

## Current state

- `/deploy-prod` workflow now completes successfully.
- Endpoint `diabetes-endpoint-8ab6a8cf` and deployment `blue` are both in a
  `Succeeded` state.
- Traffic is routed 100% to `blue`.

## Open issue (unresolved — pick up next session)

**Symptom:** Testing the endpoint in Azure Machine Learning studio (Endpoints
→ Real-time endpoints → the endpoint → Test tab) with the sample payload from
the lab returns **null** for the prediction instead of a boolean
classification result.

**Not yet investigated.** Starting points for next session:

1. **Check `MLmodel` signature vs. request format.** `model/MLmodel` declares:
   ```yaml
   signature:
     inputs: '[{"name": "Pregnancies", "type": "integer"}, ...]'
     outputs: '[{"type": "boolean"}]'
   ```
   A `null` response (rather than an error) suggests the request was
   accepted and scored, but the output serialization produced `null`. This
   is worth checking against known MLflow issues with boolean-typed outputs
   in older MLflow/pyfunc versions.
2. **Pinned environment is old.** `model/conda.yaml` pins
   `mlflow==1.30.0`, `scikit-learn==0.24.1`, `python=3.8`. Worth checking
   Microsoft/MLflow docs or release notes for known scoring-serialization
   bugs with boolean outputs in that MLflow version.
3. **Check the deployment container logs**, not just the Test tab response:
   ```azurecli
   az ml online-deployment get-logs --name blue --endpoint-name diabetes-endpoint-8ab6a8cf \
     --resource-group <your-rg> --workspace-name <your-ws> --lines 100
   ```
   or via the Python SDK:
   ```python
   ml_client.online_deployments.get_logs(name="blue", endpoint_name="diabetes-endpoint-8ab6a8cf", lines=100)
   ```
   This should show whether the scoring script itself errored/warned during
   inference, or returned `null` cleanly.
4. **Verify the request payload column order/types exactly match the
   signature** (integer vs. double for `BMI`/`DiabetesPedigree`, which are
   declared as `double` in the signature but the sample payload passes them
   as decimal literals — should be fine, but worth double-checking after
   ruling out the above).

## Issue 4 (diagnosed, fix planned — pick up next session): `az ml job stream` crashes with `binascii.Error`, breaking both `train-dev.yml` and `train-prod.yml`

**Context:** Working through the CI training workflows
(`.github/workflows/train-dev.yml`, triggered on PR events, and
`.github/workflows/train-prod.yml`, triggered by an `/train-prod` PR
comment). Both call `az ml job create` against `src/job.yml`, then rely on
the CLI's log-streaming feature to know when the job is done. In both
workflows, the underlying Azure ML training job (`src/train-model-parameters.py`)
completes successfully every time — confirmed in Azure ML Studio (job
`diabetes-train-prod-35620956263`: Completed, 21.84s, Accuracy 0.774, AUC
0.8483233; job `diabetes-train-prod-35625216701`: also completed, after a
~5 min cold-started run). The CI workflow still reports failure/bad output.

### Symptom A — prod: hard failure, no PR comment

`train-prod.yml`'s "Stream prod training job" step:
```yaml
- name: Stream prod training job
  run: az ml job stream --name "${{ steps.train.outputs.job_name }}"
```
fails with exit code 1 and prints:
```
ERROR: Met error <class 'binascii.Error'>:Invalid base64-encoded string: number of data characters (97) cannot be 1 more than a multiple of 4
Please check log by running the command with '--debug' for more details.
```
(A second run instead printed `ERROR: Met error <class 'binascii.Error'>:Incorrect padding` — same exception class, same underlying cause, different malformed-string shape.)
Because this step has no `if:`/`continue-on-error:`, its failure causes
GitHub Actions to skip every subsequent step (`Download prod training
outputs`, `Extract prod metrics from output artifact`, `Comment prod metrics
on pull request` all show as skipped ⊘). Result: no PR comment at all, job
marked failed. The `Post Sign in to Azure` / `Post Check out repository`
steps still show green — these are auto-registered cleanup hooks from
`azure/login@v2` / `actions/checkout@v4` that run on `if: always()`
regardless of job outcome, so their success does not imply the pipeline
continued.

### Symptom B — dev: silent bad output, misleading PR comment

`train-dev.yml`'s training step pipes the same streaming call into `tee`:
```yaml
az ml job create -f src/job.yml --name "$JOB_NAME" \
  --set inputs.training_data.path=azureml:diabetes-dev-folder@latest \
  --stream | tee training_output.log
```
A bash pipeline's exit code is the *last* command's (`tee`, which always
succeeds), so the same `binascii.Error` crash is silently swallowed — the
step is reported green. But because the crash cuts the stream short,
`training_output.log` never receives the script's `Accuracy: ...` / `AUC:
...` print lines. The next step then greps that log:
```bash
ACC=$(grep -o "Accuracy: .*" training_output.log | tail -n 1 | awk '{print $2}')
AUC=$(grep -o "AUC: .*" training_output.log | tail -n 1 | awk '{print $2}')
```
finds nothing, and the PR comment step falls back to posting:
> ✅ Dev training workflow completed.
> Metrics could not be parsed from logs. Check the Azure Machine Learning job run for details.

which looks like a "successful but slightly broken" run, masking that the
real cause is the same crash as Symptom A.

### Root cause

**Verified (Microsoft Learn, [az ml job reference](https://learn.microsoft.com/en-us/cli/azure/ml/job)):**
- `--stream` on `az ml job create` is documented as *"Indicates whether to
  stream the job's logs to the console"* with **default value `False`**.
  Streaming is therefore an optional display convenience, not part of job
  submission or execution. This is the key fact: the job runs server-side
  and completes regardless of whether anything is watching it, which is
  exactly what the Studio evidence shows.
- `az ml job stream` is documented as *"Stream job logs to the console"* —
  a read-only observer command.
- Polling with `az ml job show --query status` is a documented, supported
  pattern (official example in the same reference).

**Verified (Microsoft Learn, [Use Azure Pipelines with Azure Machine Learning](https://learn.microsoft.com/azure/machine-learning/how-to-devops-machine-learning?view=azureml-api-2#step-5-create-a-yaml-pipeline-to-submit-the-azure-machine-learning-job)):**
Microsoft explicitly documents that their job-completion **notification path
is unreliable** and prescribes status polling as the workaround:
> "Both wait mechanisms in this step ... depend on Azure Machine Learning
> sending a `RunTerminated` notification back to Azure DevOps when the job
> finishes. **This notification path is currently under investigation and
> might not complete as expected** ... If you encounter this behavior,
> **poll the job status from an agent job by using `az ml job show --query
> status` until a terminal state (`Completed`, `Failed`, or `Canceled`) is
> returned**, and exit the task with a matching status."

That guidance is for Azure DevOps rather than GitHub Actions, but it is the
same underlying problem class — a CI system trying to find out when an
Azure ML job finished — and Microsoft's own recommended answer is polling,
not streaming/notifications. This is why the polling approach below is now
the **primary** recommendation rather than a fallback.

**Verified job status values** ([JobStatus type](https://learn.microsoft.com/javascript/api/@azure/arm-machinelearning/jobstatus?view=azure-node-latest)):
`NotStarted`, `Starting`, `Provisioning`, `Preparing`, `Queued`, `Running`,
`Finalizing`, `CancelRequested`, `Completed`, `Failed`, `Canceled`,
`NotResponding`, `Paused`, `Unknown`. Only `Completed` / `Failed` /
`Canceled` are terminal — note that `NotResponding` and `Paused` are *not*
terminal, so a polling loop should keep waiting through them rather than
treating them as failures.

**Still unverified — the `binascii.Error` itself.** Best-effort
explanation: `az ml job stream` polls the service in a loop for new log
content, using an opaque continuation token to resume from where it left
off. That token (or related stream metadata) appears to be base64-encoded,
and the CLI calls Python's `base64.b64decode()` (via the stdlib `binascii`
module) on a value that isn't validly formed base64 — wrong length / bad
padding — raising `binascii.Error`. A targeted Microsoft Learn search for
this error in `az ml job stream` returned nothing relevant, and GitHub code
search against `Azure/azure-sdk-for-python` returned no matches. Treat the
"why" as reasonable inference, not a confirmed citation. **It does not need
to be confirmed for the fix below to work**, because the fix removes the
dependency on streaming entirely rather than working around the internal
bug.

### Fix plan

All tasks lean on the fact that prod's download-and-parse-JSON-artifact
steps (`Download prod training outputs`, `Extract prod metrics from output
artifact`) are already correct — they only ever failed because they were
skipped. `src/job.yml` already declares `outputs.metrics_output`, and
`save_metrics()` in `src/train-model-parameters.py` already writes
`metrics.json` there for every run, dev included.

The strategy is: **submit the job, wait by polling status, read metrics from
the artifact.** Never call `az ml job stream`, and never parse streamed log
text. Every command used in this approach is documented and supported (see
Root cause above).

**Task 1 — prod: replace the stream step with a status-polling wait**
File: `.github/workflows/train-prod.yml`
Delete the "Stream prod training job" step and replace it with a
wait-for-completion step that polls `az ml job show --query status`. This
is the pattern Microsoft recommends in the Azure Pipelines doc cited above.
A timeout cap matters: without one, a stuck job hangs the runner until
GitHub's 6-hour job limit. 30 minutes is generous given the observed
cold-start run took ~5 minutes.

> **Developer prompt (CORE):**
> **C — Context:** `.github/workflows/train-prod.yml` has a step named "Stream prod training job" that runs `az ml job stream --name "${{ steps.train.outputs.job_name }}"`. This step reliably crashes with `binascii.Error` (reproduced on two separate runs), exits 1, and causes GitHub Actions to skip every following step. It is also currently the only thing making the workflow wait for the Azure ML job to finish, because the preceding "Run training job in prod" step submits without `--stream` and returns immediately. Microsoft's own documentation ([Use Azure Pipelines with Azure Machine Learning](https://learn.microsoft.com/azure/machine-learning/how-to-devops-machine-learning?view=azureml-api-2)) recommends polling `az ml job show --query status` until a terminal state is reached, because job-completion notification paths are unreliable.
> **O — Objective:** Replace the "Stream prod training job" step with a step named "Wait for prod training job" that blocks until the Azure ML job reaches a terminal state, using status polling instead of log streaming, and fails the step if the job ends in `Failed` or `Canceled`.
> **R — Requirements:** Use `az ml job show --name "$JOB_NAME" --query status -o tsv` in a loop with `sleep 15` between polls. Treat only `Completed`, `Failed`, and `Canceled` as terminal — all other statuses (including `NotResponding` and `Paused`) must continue waiting. Include a 30-minute timeout that exits 1 with a clear message. Echo the current status each iteration so the Actions log shows progress. Do not call `az ml job stream` anywhere. Do not modify any other step in the file.
> **E — Expected output:** Show the full new step, and confirm that `az ml job stream` no longer appears anywhere in `train-prod.yml`.

**Task 2 — dev: submit without `--stream`, then poll**
File: `.github/workflows/train-dev.yml`
The "Run training job in dev and capture logs" step currently does
`az ml job create ... --stream | tee training_output.log` and never exposes
`$JOB_NAME` to later steps. Change it to submit without `--stream` (matching
prod's submit step), write `job_name` to `$GITHUB_OUTPUT`, and then add the
same polling wait step from Task 1. `training_output.log` goes away
entirely.

> **Developer prompt (CORE):**
> **C — Context:** `.github/workflows/train-dev.yml`'s "Run training job in dev and capture logs" step builds `JOB_NAME="diabetes-train-dev-${{ github.run_id }}"` and runs `az ml job create -f src/job.yml --name "$JOB_NAME" --set inputs.training_data.path=azureml:diabetes-dev-folder@latest --stream | tee training_output.log`. Two problems: (1) `--stream` triggers the same `binascii.Error` crash seen in prod, but the `| tee` pipe swallows the non-zero exit code so it fails silently; (2) `$JOB_NAME` is never written to `$GITHUB_OUTPUT`, so later steps can't reference the job. Compare with `.github/workflows/train-prod.yml`'s "Run training job in prod" step, which submits without `--stream`, uses `--query name -o tsv`, and writes `echo "job_name=$JOB_NAME" >> "$GITHUB_OUTPUT"`.
> **O — Objective:** Rewrite dev's training step to submit the job without `--stream` (following prod's submit-step pattern, keeping the dev-specific `diabetes-dev-folder` input override and the `diabetes-train-dev-` name prefix) and expose `job_name` as a step output. Then add a "Wait for dev training job" step identical in logic to the polling step created in Task 1.
> **R — Requirements:** Remove `--stream` and the `| tee training_output.log` pipe entirely — `training_output.log` must no longer be created or referenced. Keep the step `id: train`. Rename the step to "Run training job in dev" since it no longer captures logs. Do not modify the "Comment dev metrics on pull request" step in this task.
> **E — Expected output:** Show the updated training step and the new wait step, and confirm neither `--stream` nor `training_output.log` appears anywhere in `train-dev.yml`.

**Task 3 — dev: replace log-grep metrics extraction with artifact download**
File: `.github/workflows/train-dev.yml`
Delete the "Extract dev metrics from logs" step (the `grep -o "Accuracy:
.*"` / `grep -o "AUC: .*"` block). Replace it with two steps copied from
`train-prod.yml` — "Download prod training outputs" and "Extract prod
metrics from output artifact" — renamed for dev and using
`steps.train.outputs.job_name` (now available after Task 2).

> **Developer prompt (CORE):**
> **C — Context:** `.github/workflows/train-dev.yml` currently has a step named "Extract dev metrics from logs" that greps `training_output.log` for `"Accuracy: .*"` and `"AUC: .*"` text. After Task 2 that log file no longer exists at all, so this step is now broken as well as unreliable. `.github/workflows/train-prod.yml` already solves this correctly with two steps: "Download prod training outputs" (downloads the `metrics_output` artifact via `az ml job download --name "$JOB_NAME" --output-name metrics_output --download-path downloaded-job`, locates `metrics.json` with `find`, and also fetches the Studio URL) and "Extract prod metrics from output artifact" (parses `accuracy`/`auc` out of that JSON file with `python -c "import json; ..."` and writes them to `$GITHUB_OUTPUT` as `prod_accuracy`/`prod_auc`). Note that `az ml job download` places files in a folder named after the job, which is why prod uses `find` to locate `metrics.json` rather than a hardcoded path.
> **O — Objective:** In `train-dev.yml`, delete the "Extract dev metrics from logs" step and replace it with equivalent copies of prod's two steps, adapted for dev: use `steps.train.outputs.job_name` (added in Task 2), name the steps "Download dev training outputs" and "Extract dev metrics from output artifact", and write outputs as `dev_accuracy` / `dev_auc` (matching what the existing "Comment dev metrics on pull request" step already expects via `steps.parse-metrics.outputs.dev_accuracy` / `dev_auc`).
> **R — Requirements:** Keep the step `id: parse-metrics` on the metrics-extraction step so the existing "Comment dev metrics on pull request" step continues to work with zero changes to that step. Do not modify the "Comment dev metrics on pull request" step. Do not change any prod file in this task.
> **E — Expected output:** Show the new two steps in full, placed where "Extract dev metrics from logs" used to be, and confirm `training_output.log` is no longer read anywhere in the file.

**Task 4 — verify end-to-end**
1. Confirm job `diabetes-train-prod-35625216701` shows `Completed` in Azure
   ML Studio (Web View link from the failed run's log).
2. Optionally sanity-check the artifact path locally/Cloud Shell before
   relying on it in CI:
   ```azurecli
   az ml job download --name diabetes-train-prod-35620956263 \
     --output-name metrics_output --download-path ./downloaded-job
   ```
   Confirm `metrics.json` contains `accuracy`/`auc` matching Studio (0.774 /
   0.8483233).
3. Push a change to `src/train-model-parameters.py` or `src/job.yml` (or use
   `workflow_dispatch`) to trigger `train-dev.yml`; confirm the PR comment
   now shows real Accuracy/AUC values instead of "Metrics could not be
   parsed from logs."
4. Post a **new** `/train-prod` comment on the PR (do not use "Re-run failed
   jobs" — `github.run_id` is documented as staying the same across re-runs
   while `github.run_attempt` increments, and the Azure ML job name embeds
   `run_id`, so a re-run would collide with the already-existing job name
   and fail immediately). Confirm the workflow now posts a "✅ Prod training
   workflow completed" comment with real metrics, and that no step logs a
   `binascii.Error` (the streaming code path is no longer invoked at all).

> **Developer prompt (CORE):**
> **C — Context:** Tasks 1–3 have been applied to `train-dev.yml` and `train-prod.yml`, replacing all log-streaming and log-grepping with status polling plus artifact download. Job `diabetes-train-prod-35625216701` is already confirmed `Completed` in Azure ML Studio with real metrics, so it can be used to validate the artifact-download path without spending compute on a new run.
> **O — Objective:** Verify both workflows now produce PR comments with real Accuracy/AUC values, and confirm the `binascii.Error` no longer appears anywhere because `az ml job stream` / `--stream` are no longer used.
> **R — Requirements:** Do not re-run the existing failed prod run via the GitHub UI "Re-run jobs" button (job name collision, will fail for an unrelated reason). Trigger dev via a real file change or `workflow_dispatch`; trigger prod via a brand-new `/train-prod` PR comment.
> **E — Expected output:** Report back: (a) dev PR comment content, (b) prod PR comment content, (c) confirmation that the "Wait for ... training job" step in each workflow logged status transitions and ended on `Completed`, (d) confirmation that `binascii.Error` appears in neither run's logs, (e) confirmation that both comments contain non-empty Accuracy/AUC values.

### Simpler alternative (lower confidence — only if you want a one-line change)

Instead of Task 1, adding `continue-on-error: true` to prod's existing
"Stream prod training job" step would also let the pipeline proceed to the
download/extract/comment steps:
```yaml
- name: Stream prod training job
  continue-on-error: true
  run: |
    az ml job stream --name "${{ steps.train.outputs.job_name }}"
```
`continue-on-error` at step level is documented as *"prevents a job from
failing when a specific step fails"*
([GitHub Actions workflow syntax](https://docs.github.com/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idstepscontinue-on-error)),
so this does work mechanically. The observed timing supports it too: the
crashing stream step ran 41s against a 21.84s job, and 5m16s against a
cold-started run, implying the crash happens at or after job completion.

**Why this is not the primary recommendation:** it keeps a known-broken,
undiagnosed code path as the workflow's only wait mechanism. If the crash
ever happens *early* rather than at end-of-stream, the download step fires
against a still-running job and the workflow fails with "No metrics.json
found" — a confusing, intermittent failure mode. Polling has no such
failure mode and is what Microsoft recommends. Prefer Task 1.

### Additional hardening (optional, not required for the fix to work)

- Change job names to include `${{ github.run_attempt }}` (e.g.
  `diabetes-train-prod-${{ github.run_id }}-${{ github.run_attempt }}`) in
  both workflows, so the GitHub "Re-run failed jobs" button becomes usable
  instead of colliding with an existing Azure ML job name.
- Prod's "Comment prod metrics on pull request" step interpolates
  `prod_accuracy`/`prod_auc` with no emptiness check, unlike dev's `if (acc
  || auc)` guard. Consider mirroring dev's fallback message for consistency.
- Pin the CLI extension version (`az extension add -n ml --version <x.y.z>`)
  instead of the current unpinned `az extension add -n ml -y`, so a future
  CLI regression can't silently change workflow behaviour.
- Add `set -o pipefail` to the top of any step that pipes a command into
  another, so a crash in a piped command can never again be silently
  swallowed. (After Tasks 1–3 there are no such pipes left in these two
  workflows, but this is worth adopting as a convention.)

---

## Reference links used this session

- [Unable to Create Online Endpoint in Azure ML — Resource Provider Error (Microsoft Q&A)](https://learn.microsoft.com/answers/a/2063782)
- [How to fix SubscriptionNotRegistered (Microsoft Q&A)](https://learn.microsoft.com/answers/a/2290713)
- [Azure resource providers and types — Register resource provider](https://learn.microsoft.com/azure/azure-resource-manager/management/resource-providers-and-types#register-resource-provider)
- [Troubleshoot online endpoint deployment and scoring](https://learn.microsoft.com/azure/machine-learning/how-to-troubleshoot-online-endpoints?view=azureml-api-2)
- [Managed online endpoints SKU list](https://learn.microsoft.com/azure/machine-learning/reference-managed-online-endpoints-vm-sku-list?view=azureml-api-2)
- Local files consulted: `src/deploy_to_online_endpoint.py`,
  `.github/workflows/deploy-prod.yml`, `infra/setup.sh`,
  `model/MLmodel`, `model/conda.yaml`, `docs/07-deploy-monitor.md`
- Issue 4 — sources that verify the fix plan:
  - [az ml job (Azure CLI reference)](https://learn.microsoft.com/en-us/cli/azure/ml/job) —
    confirms `--stream` defaults to `False`, `az ml job stream` is a
    console-only observer, `--output-name` downloads a named job output, and
    downloads land in a folder named after the job (hence the `find`).
    Also gives the official `az ml job show --query status` example.
  - [Use Azure Pipelines with Azure Machine Learning](https://learn.microsoft.com/azure/machine-learning/how-to-devops-machine-learning?view=azureml-api-2#step-5-create-a-yaml-pipeline-to-submit-the-azure-machine-learning-job) —
    Microsoft states the job-completion notification path is "currently
    under investigation and might not complete as expected" and prescribes
    polling `az ml job show --query status` until `Completed`/`Failed`/
    `Canceled`. This is the basis for making polling the primary fix.
  - [JobStatus type (@azure/arm-machinelearning)](https://learn.microsoft.com/javascript/api/@azure/arm-machinelearning/jobstatus?view=azure-node-latest) —
    full enum of job statuses; confirms which are terminal.
  - [GitHub Actions workflow syntax — `continue-on-error`](https://docs.github.com/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idstepscontinue-on-error)
    and the `github` context (`run_id` stable across re-runs,
    `run_attempt` increments) — retrieved via Context7
    `/websites/github_en_actions`, because `docs.github.com` is intercepted
    by a corporate Zscaler proxy on this network.
- Issue 4 — local files consulted: `.github/workflows/train-dev.yml`,
  `.github/workflows/train-prod.yml`, `src/job.yml`,
  `src/train-model-parameters.py`.
- Issue 4 — **what remains unverified:** the `binascii.Error` itself. A
  targeted Microsoft Learn search for this error in `az ml job stream`
  returned nothing relevant, and GitHub code search against
  `Azure/azure-sdk-for-python` returned no matches. The root-cause
  explanation above is inference from observed CI logs, not a confirmed
  citation. The fix does not depend on it.
