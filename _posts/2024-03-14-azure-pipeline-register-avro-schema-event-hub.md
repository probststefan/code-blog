---
layout: post
title: Streamlining Azure Pipelines - Automating Avro Schema Publication to Event Hubs Schema Registry
---
```bash
- task: AzureCLI@2
  displayName: Register Avro Schemas
  inputs:
    azureSubscription: "Service Connection"
    scriptType: "bash"
    scriptLocation: "inlineScript"
    inlineScript: |
      response=$(az account get-access-token --resource https://${{ parameters.applicationName }}.servicebus.windows.net)
        
      token="Bearer `echo $response | jq ."accessToken" | tr -d '"'`"
        
      curl -X PUT -d '{"namespace": "com.azure.schemaregistry.samples","type": "record","name": "Order","fields": [{"name": "id","type": "string"},{"name": "amount","type": "double"}]}'  -H "Content-Type:application/json" -H "Authorization:$token" -H "Serialization-Type:Avro" 'https://${{ parameters.applicationName }}.servicebus.windows.net/$schemagroups/avro/schemas/stefan?api-version=2020-09-01-preview'
```
