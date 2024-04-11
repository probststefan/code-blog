---
layout: post
title: Set up Azure Event Hub with log compaction in terraform
---

Enabling log compaction via the Terraform resource "[azurerm_eventhub](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/eventhub)" is currently not feasible. Additionally, a [bug](https://github.com/hashicorp/terraform-provider-azurerm/issues/25563) has been identified within "azurerm_eventhub_authorization_rule". Attempting to create an authorization rule encounters an issue when log compaction is enabled.

That's why I transitioned to Azure ARM templates for deploying an Event Hub with both log compaction and authorization rules.

```bash
resource "azurerm_resource_group" "resource_group" {
  name     = var.application_name
  location = var.location
}

resource "azurerm_resource_group_template_deployment" "eventhub" {
  name                = "eventhub-template"
  resource_group_name = azurerm_resource_group.resource_group.name
  deployment_mode     = "Incremental"
  template_content    = <<TEMPLATE
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.EventHub/namespaces",
      "apiVersion": "2023-01-01-preview",
      "name": "${var.application_name}",
      "location": "${azurerm_resource_group.resource_group.location}",
      "sku": {
          "name": "Premium",
          "tier": "Premium"
      },
      "properties": {
        "zoneRedundant": true
      },
      "resources": [
        {
          "apiVersion": "2023-01-01-preview",
          "name": "${var.eventhub_name}",
          "type": "eventhubs",
          "dependsOn": [
              "Microsoft.EventHub/namespaces/${var.application_name}"
          ],
          "properties": {
            "partitionCount": "1",
            "retentionDescription": {
              "cleanupPolicy": "Compact",
              "tombstoneRetentionTimeInHours": 96
            }
          },
          "resources": [
            {
              "type": "authorizationRules",
              "apiVersion": "2023-01-01-preview",
              "name": "default",
              "dependsOn": [
                "[resourceId('Microsoft.EventHub/namespaces/eventhubs/', '${var.application_name}', '${var.eventhub_name}')]"
              ],
              "properties": {
                "rights": ["Send", "Listen"]
              }
            }
          ]
        }
      ]
    }
  ],
  "outputs": {
    "RootManageSharedAccessKeyConnectionString": {
      "type": "string",
      "value": "[listkeys(resourceId('Microsoft.EventHub/namespaces/AuthorizationRules', '${var.application_name}', 'RootManageSharedAccessKey'), '2017-04-01').primaryConnectionString]"
    },
    "defaultConnectionString": {
      "type": "string",
      "value": "[listkeys(resourceId('Microsoft.EventHub/namespaces/eventhubs/AuthorizationRules', '${var.application_name}', '${var.eventhub_name}', 'default'), '2017-04-01').primaryConnectionString]"
    },
    "eventHubNamespaceId": {
      "type": "string",
      "value": "[resourceId('Microsoft.EventHub/namespaces', '${var.application_name}')]"
    }
  }
}
  TEMPLATE
}
```
