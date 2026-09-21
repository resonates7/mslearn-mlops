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

## Reference links used this session

- [Unable to Create Online Endpoint in Azure ML — Resource Provider Error (Microsoft Q&A)](https://learn.microsoft.com/answers/a/2063782)
- [How to fix SubscriptionNotRegistered (Microsoft Q&A)](https://learn.microsoft.com/answers/a/2290713)
- [Azure resource providers and types — Register resource provider](https://learn.microsoft.com/azure/azure-resource-manager/management/resource-providers-and-types#register-resource-provider)
- [Troubleshoot online endpoint deployment and scoring](https://learn.microsoft.com/azure/machine-learning/how-to-troubleshoot-online-endpoints?view=azureml-api-2)
- [Managed online endpoints SKU list](https://learn.microsoft.com/azure/machine-learning/reference-managed-online-endpoints-vm-sku-list?view=azureml-api-2)
- Local files consulted: `src/deploy_to_online_endpoint.py`,
  `.github/workflows/deploy-prod.yml`, `infra/setup.sh`,
  `model/MLmodel`, `model/conda.yaml`, `docs/07-deploy-monitor.md`
