# Azure-Policy-Automatic-remediation

Great question 👍. In **Logic Apps**, you have a few ways to display or pass outputs depending on what you want to see:

---

## 1. **Use a `Response` action (for HTTP-triggered apps)**

If your Logic App starts with an **HTTP trigger**, you can add a **Response** action at the end to return values (JSON, text, etc.) back to the caller (Postman, cURL, API, another app).

Example:

```json
{
  "type": "Response",
  "inputs": {
    "statusCode": 200,
    "headers": { "Content-Type": "application/json" },
    "body": {
      "message": "Remediation workflow complete",
      "nonCompliantCount": "@outputs('Compose_NonCompliant_Count')",
      "remediationResult": "@actions('Create_Remediation_Subscription')?['outputs']"
    }
  }
}
```

👉 This way, when you call the Logic App URL, you’ll see the JSON response with your remediation results.

---

## 2. **Use `Compose` actions**

* `Compose` is handy for showing intermediate results (counts, JSON, or text).
* These show up in the **Run History** in the portal.
* You already have `Compose_NonCompliant_Count` and `Compose_Skip_Message` in the workflow I shared — those values will be visible in the run logs.

---

## 3. **Send outputs elsewhere**

You can add actions like:

* **Send an email** (Office 365 Outlook, SMTP, etc.)
* **Post a message to Teams / Slack**
* **Insert a row in a SQL table**
* **Write to Blob Storage / Log Analytics**

Example (send an email with results):

```json
{
  "type": "ApiConnection",
  "inputs": {
    "host": {
      "connection": { "name": "@parameters('$connections')['office365']['connectionId']" }
    },
    "method": "post",
    "path": "/v2/Mail",
    "body": {
      "To": "ops-team@contoso.com",
      "Subject": "Azure Policy Remediation Results",
      "Body": "Non-compliant count: @{outputs('Compose_NonCompliant_Count')}"
    }
  }
}
```

---

## 4. **Check Run History**

Even without a Response action:

* Go to your Logic App in the Azure Portal.
* Open **Runs history** → select a run → expand actions → see inputs/outputs.
* This is often how you debug and inspect outputs like compliance counts or remediation API responses.

---

✅ **Best practice**:
If you expect to trigger the Logic App manually or from another app → use a **Response** action.
If you just want to monitor → rely on **Compose** and the **Run History**, or push results to monitoring (Log Analytics, Storage, etc.).

---

Would you like me to **add a `Response` action** to the workflow I gave you so it automatically returns the remediation outcome to Postman/cURL when you test it?



Here’s a **Logic App (Consumption) Code View** workflow that:

1. Accepts a POST with your `policyAssignmentId` and `remediationName`
2. Calls **Policy Insights – policyStates/latest/summarize** to get the non-compliant count
3. **If count > 0**, creates a remediation (subscription scope by default, easy toggle for RG)

---

### Logic App — Code View JSON

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
        "metadata": { "description": "Use 'subscription' or 'resourceGroup'." }
      },
      "policyApiVersion": {
        "type": "string",
        "defaultValue": "2022-09-01"
      },
      "remediationApiVersion": {
        "type": "string",
        "defaultValue": "2021-10-01"
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
              "resourceGroupName": { "type": "string", "description": "Required when defaultScope = resourceGroup" },
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
      "Summarize_NonCompliance": {
        "type": "Http",
        "inputs": {
          "method": "POST",
          "uri": "@{concat('https://management.azure.com/subscriptions/', parameters('subscriptionId'), '/providers/Microsoft.PolicyInsights/policyStates/latest/summarize?api-version=', parameters('policyApiVersion'))}",
          "headers": {
            "Content-Type": "application/json"
          },
          "body": {
            "filter": "@{concat(\"PolicyAssignmentId eq '\", triggerBody()?['policyAssignmentId'], \"'\")}"
          },
          "authentication": {
            "type": "ManagedServiceIdentity",
            "audience": "https://management.azure.com"
          }
        }
      },

      "Parse_Summary": {
        "type": "ParseJson",
        "runAfter": { "Summarize_NonCompliance": [ "Succeeded" ] },
        "inputs": {
          "content": "@body('Summarize_NonCompliance')",
          "schema": {
            "type": "object",
            "properties": {
              "value": {
                "type": "array",
                "items": {
                  "type": "object",
                  "properties": {
                    "results": {
                      "type": "array",
                      "items": {
                        "type": "object",
                        "properties": {
                          "nonCompliantResources": { "type": "integer" },
                          "compliantResources": { "type": "integer" }
                        },
                        "required": [ "nonCompliantResources" ]
                      }
                    }
                  },
                  "required": [ "results" ]
                }
              }
            },
            "required": [ "value" ]
          }
        }
      },

      "Compose_NonCompliant_Count": {
        "type": "Compose",
        "runAfter": { "Parse_Summary": [ "Succeeded" ] },
        "inputs": "@coalesce(first(first(body('Parse_Summary')?['value'])?['results'])?['nonCompliantResources'], 0)"
      },

      "Condition_Create_Remediation": {
        "type": "If",
        "runAfter": { "Compose_NonCompliant_Count": [ "Succeeded" ] },
        "expression": {
          "greater": [
            "@outputs('Compose_NonCompliant_Count')",
            0
          ]
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
            }
          },
          "Create_Remediation_Subscription": {
            "type": "Http",
            "runAfter": { "Compose_RemediationBody": [ "Succeeded" ] },
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
              "uri": "@{concat('https://management.azure.com/subscriptions/', parameters('subscriptionId'), '/providers/Microsoft.PolicyInsights/remediations/', encodeURIComponent(triggerBody()?['remediationName']), '?api-version=', parameters('remediationApiVersion'))}",
              "headers": { "Content-Type": "application/json" },
              "body": "@{outputs('Compose_RemediationBody')}",
              "authentication": {
                "type": "ManagedServiceIdentity",
                "audience": "https://management.azure.com"
              }
            }
          },
          "Create_Remediation_ResourceGroup": {
            "type": "Http",
            "runAfter": { "Create_Remediation_Subscription": [ "Skipped" ] },
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
              "uri": "@{concat('https://management.azure.com/subscriptions/', parameters('subscriptionId'), '/resourceGroups/', encodeURIComponent(triggerBody()?['resourceGroupName']), '/providers/Microsoft.PolicyInsights/remediations/', encodeURIComponent(triggerBody()?['remediationName']), '?api-version=', parameters('remediationApiVersion'))}",
              "headers": { "Content-Type": "application/json" },
              "body": "@{outputs('Compose_RemediationBody')}",
              "authentication": {
                "type": "ManagedServiceIdentity",
                "audience": "https://management.azure.com"
              }
            }
          }
        },
        "else": {
          "actions": {
            "Compose_Skip_Message": {
              "type": "Compose",
              "inputs": {
                "message": "No non-compliant resources found for the supplied policyAssignmentId. Remediation not created.",
                "nonCompliantCount": "@outputs('Compose_NonCompliant_Count')"
              }
            }
          }
        }
      }
    },
    "outputs": {
      "result": {
        "type": "Object",
        "value": {
          "nonCompliantCount": "@outputs('Compose_NonCompliant_Count')",
          "subscriptionCall": "@actions('Create_Remediation_Subscription')?['outputs']",
          "resourceGroupCall": "@actions('Create_Remediation_ResourceGroup')?['outputs']"
        }
      }
    }
  }
}
```

---

### How to use

1. **Enable Managed Identity** on the Logic App.
2. Grant it **Resource Policy Contributor** at the scope where you’ll create remediations (subscription or RG).
3. Set `subscriptionId` in Parameters (Code View), and optionally set `defaultScope` to `resourceGroup`.
4. POST one of your earlier payloads (remediation name + policy assignment ID, etc.).
5. The workflow will summarize compliance, then **only create a remediation if the non-compliant count > 0**. Otherwise it returns a skip message with the count.

If you’d like, I can also provide a **Standard (single-tenant) Logic App** version or an **ARM/Bicep template** that deploys this workflow end-to-end.

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
Here’s a simple **sample JSON payload** you can post to your Logic App’s HTTP trigger to create a remediation task:

### Example – subscription scope

```json
{
  "remediationName": "enable-diagnostics-remediation",
  "policyAssignmentId": "/subscriptions/11111111-2222-3333-4444-555555555555/providers/Microsoft.Authorization/policyAssignments/enable-diags-assignment",
  "resourceDiscoveryMode": "ReEvaluateCompliance",
  "locations": [ "eastus", "westus" ],
  "resourceCount": 50,
  "parallelDeployments": 3,
  "failureThresholdPercent": 5
}
```

### Example – resource group scope

```json
{
  "remediationName": "rg-remediation-task",
  "policyAssignmentId": "/subscriptions/11111111-2222-3333-4444-555555555555/resourceGroups/MyResourceGroup/providers/Microsoft.Authorization/policyAssignments/storage-policy-assignment",
  "resourceGroupName": "MyResourceGroup",
  "policyDefinitionReferenceId": "8c8fa9e4",
  "resourceDiscoveryMode": "ExistingNonCompliant"
}
```

---

🔑 **Notes:**

* Replace the GUID with your actual subscription ID.
* `remediationName` can be any unique string.
* `policyAssignmentId` must point to the **policy assignment** you want to remediate.
* `policyDefinitionReferenceId` is required if the assignment is an **initiative** and you want to remediate a single policy inside it.
* `resourceDiscoveryMode`:

  * `ExistingNonCompliant` = only remediate currently flagged resources.
  * `ReEvaluateCompliance` = re-check compliance and then remediate.

Do you want me to also give you a **ready-to-run `curl` example** that calls your Logic App trigger URL with one of these payloads, so you can test it immediately?


## Extras & tuning

* Use `locations` to restrict remediation to certain regions; `parallelDeployments`, `resourceCount`, and `failureThreshold.percentage` let you tune pace and stop conditions. ([Microsoft Learn][4])
* You can also target **management group** or **individual resource** scopes with the analogous REST paths if needed. ([Microsoft Learn][7])


[1]: https://learn.microsoft.com/en-us/azure/governance/policy/overview?utm_source=chatgpt.com "Overview of Azure Policy"
[2]: https://www.azadvertizer.net/azrolesadvertizer/36243c78-bf99-498c-9df9-86d9f8d28608.html?utm_source=chatgpt.com "Resource Policy Contributor - 36243c78-bf99-498c-9df9- ..."
[3]: https://learn.microsoft.com/en-us/azure/governance/policy/concepts/remediation-structure?utm_source=chatgpt.com "Azure Policy remediation task structure"
[4]: https://learn.microsoft.com/en-us/rest/api/policy/remediations/create-or-update-at-subscription?view=rest-policy-2021-10-01 "Remediations - Create Or Update At Subscription - REST API (Azure Policy) | Microsoft Learn"
[5]: https://learn.microsoft.com/en-us/rest/api/policy/remediations?view=rest-policy-2021-10-01&utm_source=chatgpt.com "Remediations - REST API (Azure Policy)"
[6]: https://learn.microsoft.com/en-us/azure/logic-apps/authenticate-with-managed-identity?utm_source=chatgpt.com "Authenticate access and connections with managed ..."
[7]: https://learn.microsoft.com/en-us/rest/api/policy/remediations/create-or-update-at-management-group?view=rest-policy-2021-10-01&utm_source=chatgpt.com "Remediations - Create Or Update At Management Group"
