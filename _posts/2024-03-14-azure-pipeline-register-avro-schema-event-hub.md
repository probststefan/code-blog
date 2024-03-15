---
layout: post
title: Streamlining Azure Pipelines - Automating Avro Schema Publication to Event Hubs Schema Registry
---
At present, the Azure CLI does not provide a means to upload schemas into the Event Hubs registry. Currently, the only options available are through the JavaScript SDK and the Azure Portal. I have been exploring a straightforward method to publish schemas seamlessly through a pipeline. This approach ensures that schemas are automatically versioned, released with each deployment, and mitigates the risk of oversight.

```bash
- task: AzureCLI@2
  displayName: Register Avro Schemas
  inputs:
    azureSubscription: "Service Connection"
    scriptType: "bash"
    scriptLocation: "inlineScript"
    inlineScript: |
      response=$(az account get-access-token --resource https://<Namespace_Name>.servicebus.windows.net)
        
      token="Bearer `echo $response | jq ."accessToken" | tr -d '"'`"

      avro='{"namespace": "com.azure.schemaregistry.samples","type": "record","name": "Order","fields": [{"name": "id","type": "string"},{"name": "amount","type": "double"}]}'
        
      curl -X PUT -d $avro -H "Content-Type:application/json" -H "Authorization:$token" -H "Serialization-Type:Avro" 'https://<Namespace_Name>.servicebus.windows.net/$schemagroups/<SchemaGroup_Name>/schemas/<Schema_Name>?api-version=2020-09-01-preview'
```

Using the AzureCLI@2 Tasks along with a Service Connection, we have the capability to execute `az` commands and obtain an access token for the *.servicebus.windows.net resource. With this token, it becomes feasible to pump Avro files into the registry.

In a further step, I expanded the script to include the capability of reading Avro files from a directory.

```bash
response=$(az account get-access-token --resource https://<Namespace_Name>.servicebus.windows.net)

token="Bearer `echo $response | jq ."accessToken" | tr -d '"'`"

for file in *.avro; do
  avro_data=$(cat "$file")     
  schema_name="${file%.*}"

  echo "Publishing $file to schema registry <Namespace_Name>"
  echo "Content: $avro_data"

  curl -X PUT -d "$avro_data" -H "Content-Type:application/json" -H "Authorization:$token" -H "Serialization-Type:Avro" "https://<Namespace_Name>.servicebus.windows.net/\$schemagroups/<SchemaGroup_Name>/schemas/$schema_name?api-version=2020-09-01-preview"
done
```
