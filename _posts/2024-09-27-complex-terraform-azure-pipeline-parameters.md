```terraform
variable "email_addresses" {
  type = list(object({
    name          = string
    email_address = string
  }))
  default = []
}

resource "azurerm_monitor_action_group" "monitoring_action_group" {
  name                = "Application Monitoring"
  resource_group_name = azurerm_resource_group.resource_group.name
  short_name          = "EmailAlert"

  dynamic "email_receiver" {
    for_each = var.email_addresses
    content {
      name          = email_receiver.value["name"]
      email_address = email_receiver.value["email_address"]
    }
  }
}
```

```yml
parameters:
  - name: alertEmails
    type: object
    default: []

steps:
  - script: |
      echo "{\"email_addresses\": $ALERT_EMAILS_JSON}" > $(terraform_working_dir)/terraform.tfvars.json
    env:
      ALERT_EMAILS_JSON: ${{ convertToJson(parameters.alertEmails) }}
    displayName: Generate terraform.tfvars.json from parameters
  ...
  - task: TerraformTaskV4@4
    name: terraformPlan
    displayName: Create Terraform Plan
    inputs:
      provider: azurerm
      command: plan
      commandOptions: >-
        -var environment=dev
        -var-file="terraform.tfvars.json"
      workingDirectory: $(terraform_working_dir)
```

```yml
      - template: terraform.yml
        parameters:
          alertEmails:
            - name: "Foo Bar"
              email_address: "foobar@development.de"
            - name: "John Due"
              email_address: "mail@john.com"
```
