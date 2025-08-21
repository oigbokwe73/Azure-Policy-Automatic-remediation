# Azure-Policy-Automatic-remediation

Here’s a ready-to-paste **Logic App (Consumption) “Code View”** workflow that creates an **Azure Policy remediation task** via the Azure Resource Manager REST API using the Logic App’s **managed identity**.

It exposes an HTTP trigger so you can post a payload with your `policyAssignmentId`, `remediationName`, and optional tuning knobs (locations filter, discovery mode, parallelism, etc.). It includes two actions so you can target either **subscription scope** (default) or **resource group scope** with a simple switch.

> ⚠️ Prereqs (once per Logic App)
>
> 1. Enable a managed identity on the Logic App (System-assigned or User-assigned).
> 2. Grant that identity **Resource Policy Contributor** at the scope where you’ll create the remediation (subscription or resource group). This role allows creating `Microsoft.PolicyInsights/remediations/*`. ([Microsoft Learn][1], [Az Advertizer][2])
> 3. The **policy assignment’s** own managed identity (used to run the DINE/Modify remediation) must have the minimum RBAC to make the underlying changes (for example, Contributor on the target resources). ([Microsoft Learn][3])

---

# Logic App workflow (Code View JSON)

```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/schemas/2016-06-01/workflows.json",
    "contentVersion": "1.0.0.0",
    "parameters": {
      "subscriptionId": {
        "type": "string",
        "defaultValue": "<SUBSCRIPTION-ID>"
      },
      "defaultScope": {
        "type": "string",
        "defaultValue": "subscription",
        "metadata": { "description": "Use 'subscription' or 'resourceGroup' to choose the API path." }
      }
    },
    "triggers": {
      "http_in": {
        "type": "Request",
        "kind": "Http",
        "inputs": {
          "method": "POST",
          "schema": {
            "type": "object",
            "required": [ "remediationName", "policyAssignmentId" ],
            "properties": {
              "remediationName": { "type": "string" },
              "policyAssignmentId": { "type": "string" },
              "resourceGroupName": { "type": "string", "description": "Required only when defaultScope is resourceGroup" },
              "policyDefinitionReferenceId": { "type": "string" },
              "resourceDiscoveryMode": {
                "type": "string",
                "enum": [ "ExistingNonCompliant", "ReEvaluateCompliance" ],
                "default": "ExistingNonCompliant"
              },
              "locations": { "type": "array", "items": { "type": "string" } },
              "resourceCount": { "type": "integer" },
              "parallelDeployments": { "type": "integer" },
              "failureThresholdPercent": { "type": "number" }
            }
          }
        }
      }
    },
    "actions": {
      "Compose_RemediationBody": {
        "type": "Compose",
        "inputs": {
          "properties": {
            "policyAssignmentId": "@triggerBody()?['policyAssignmentId']",
            "policyDefinitionReferenceId": "@coalesce(triggerBody()?['policyDefinitionReferenceId'], null)",
            "resourceDiscoveryMode": "@coalesce(triggerBody()?['resourceDiscoveryMode'], 'ExistingNonCompliant')",
            "filters": {
              "locations": "@coalesce(triggerBody()?['locations'], null)"
            },
            "resourceCount": "@coalesce(triggerBody()?['resourceCount'], null)",
            "parallelDeployments": "@coalesce(triggerBody()?['parallelDeployments'], null)",
            "failureThreshold": {
              "percentage": "@coalesce(triggerBody()?['failureThresholdPercent'], null)"
            }
          }
        },
        "runAfter": {}
      },

      "Create_Remediation_Subscription": {
        "type": "Http",
        "runAfter": {
          "Compose_RemediationBody": [ "Succeeded" ]
        },
        "conditions": [
          {
            "expression": {
              "and": [
                { "equals": [ "@parameters('defaultScope')", "subscription" ] }
              ]
            }
          }
        ],
        "inputs": {
          "method": "PUT",
          "uri": "@{concat('https://management.azure.com/subscriptions/', parameters('subscriptionId'), '/providers/Microsoft.PolicyInsights/remediations/', encodeURIComponent(triggerBody()?['remediationName']), '?api-version=2021-10-01')}",
          "headers": {
            "Content-Type": "application/json"
          },
          "body": "@{outputs('Compose_RemediationBody')}",
          "authentication": {
            "type": "ManagedServiceIdentity",
            "audience": "https://management.azure.com"
          }
        }
      },

      "Create_Remediation_ResourceGroup": {
        "type": "Http",
        "runAfter": {
          "Create_Remediation_Subscription": [ "Skipped" ]
        },
        "conditions": [
          {
            "expression": {
              "and": [
                { "equals": [ "@parameters('defaultScope')", "resourceGroup" ] }
              ]
            }
          }
        ],
        "inputs": {
          "method": "PUT",
          "uri": "@{concat('https://management.azure.com/subscriptions/', parameters('subscriptionId'), '/resourceGroups/', encodeURIComponent(triggerBody()?['resourceGroupName']), '/providers/Microsoft.PolicyInsights/remediations/', encodeURIComponent(triggerBody()?['remediationName']), '?api-version=2021-10-01')}",
          "headers": {
            "Content-Type": "application/json"
          },
          "body": "@{outputs('Compose_RemediationBody')}",
          "authentication": {
            "type": "ManagedServiceIdentity",
            "audience": "https://management.azure.com"
          }
        }
      }
    },
    "outputs": {
      "status": {
        "type": "Object",
        "value": {
          "subscriptionCall": "@actions('Create_Remediation_Subscription')?['outputs']",
          "resourceGroupCall": "@actions('Create_Remediation_ResourceGroup')?['outputs']"
        }
      }
    }
  }
}
```

### Example request payloads

**A) Subscription scope (default)**
POST to your Logic App trigger URL with:

```json
{
  "remediationName": "kv-diagnostics-remediation",
  "policyAssignmentId": "/subscriptions/<SUBSCRIPTION-ID>/providers/Microsoft.Authorization/policyAssignments/<ASSIGNMENT-NAME>",
  "resourceDiscoveryMode": "ReEvaluateCompliance",
  "locations": [ "eastus", "westus" ],
  "resourceCount": 100,
  "parallelDeployments": 5,
  "failureThresholdPercent": 10
}
```

**B) Resource Group scope**
Either set the top-level parameter `defaultScope` to `resourceGroup` in Code View, or pass it as a parameter override at deployment time; then send:

```json
{
  "remediationName": "rg-remediation",
  "policyAssignmentId": "/subscriptions/<SUBSCRIPTION-ID>/resourceGroups/<RG>/providers/Microsoft.Authorization/policyAssignments/<ASSIGNMENT>",
  "resourceGroupName": "<RG>",
  "policyDefinitionReferenceId": "8c8fa9e4",
  "resourceDiscoveryMode": "ExistingNonCompliant"
}
```

> Notes on payload fields
>
> * `policyDefinitionReferenceId` is **required** when the policy assignment is an **initiative** and you want to remediate a specific policy within it. ([Microsoft Learn][4])
> * `resourceDiscoveryMode` accepts `ExistingNonCompliant` (default) or `ReEvaluateCompliance`. The latter re-runs evaluation before remediating. ([Microsoft Learn][4])
> * The REST shape and examples come from the Azure Policy **Remediations – Create or Update** API. ([Microsoft Learn][4])

### What this Logic App does

* Builds the remediation **request body** from your POSTed JSON.
* Calls the **subscription** endpoint by default:
  `PUT /subscriptions/{subId}/providers/Microsoft.PolicyInsights/remediations/{remediationName}?api-version=2021-10-01` ([Microsoft Learn][4])
* If `defaultScope` = `resourceGroup`, calls:
  `PUT /subscriptions/{subId}/resourceGroups/{rg}/providers/Microsoft.PolicyInsights/remediations/{remediationName}?api-version=2021-10-01` (same properties). ([Microsoft Learn][5])
* Authenticates with the Logic App’s **Managed Identity** against the **Azure Resource Manager** audience (`https://management.azure.com`). Logic Apps support MSI on the built-in HTTP action. ([Microsoft Learn][6])

---

## Quick test steps

1. In the Logic App blade, enable **Identity** and assign **Resource Policy Contributor** at your target scope. ([Microsoft Learn][1], [Az Advertizer][2])
2. Save the workflow and copy the **HTTP trigger URL**.
3. POST one of the sample payloads (e.g., from `curl`/Postman).
4. On success you’ll get `201 Created`/`200 OK` and the remediation object in the action output; you can also see it under **Policy > Remediation** in the portal. ([Microsoft Learn][4])

---

## Extras & tuning

* Use `locations` to restrict remediation to certain regions; `parallelDeployments`, `resourceCount`, and `failureThreshold.percentage` let you tune pace and stop conditions. ([Microsoft Learn][4])
* You can also target **management group** or **individual resource** scopes with the analogous REST paths if needed. ([Microsoft Learn][7])

If you want this as **Bicep/ARM** to deploy a Logic App with the definition embedded (or a **Standard** Logic App version), say the word and I’ll generate it.

[1]: https://learn.microsoft.com/en-us/azure/governance/policy/overview?utm_source=chatgpt.com "Overview of Azure Policy"
[2]: https://www.azadvertizer.net/azrolesadvertizer/36243c78-bf99-498c-9df9-86d9f8d28608.html?utm_source=chatgpt.com "Resource Policy Contributor - 36243c78-bf99-498c-9df9- ..."
[3]: https://learn.microsoft.com/en-us/azure/governance/policy/concepts/remediation-structure?utm_source=chatgpt.com "Azure Policy remediation task structure"
[4]: https://learn.microsoft.com/en-us/rest/api/policy/remediations/create-or-update-at-subscription?view=rest-policy-2021-10-01 "Remediations - Create Or Update At Subscription - REST API (Azure Policy) | Microsoft Learn"
[5]: https://learn.microsoft.com/en-us/rest/api/policy/remediations?view=rest-policy-2021-10-01&utm_source=chatgpt.com "Remediations - REST API (Azure Policy)"
[6]: https://learn.microsoft.com/en-us/azure/logic-apps/authenticate-with-managed-identity?utm_source=chatgpt.com "Authenticate access and connections with managed ..."
[7]: https://learn.microsoft.com/en-us/rest/api/policy/remediations/create-or-update-at-management-group?view=rest-policy-2021-10-01&utm_source=chatgpt.com "Remediations - Create Or Update At Management Group"
