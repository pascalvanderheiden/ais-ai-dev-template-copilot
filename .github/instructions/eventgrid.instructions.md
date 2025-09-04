---
description: 'Best practices for developing, securing, monitoring, and deploying Azure Event Grid topics, queues, and subscriptions with CloudEvents 1.0+'
applyTo: "**/src/eventgrid/**/*,**/infra/**/*"
---

# Azure Event Grid Development and Deployment Best Practices

## Overview

This guide provides best practices and ready-to-use deployment patterns for building with Azure Event Grid, including custom topics, system topics, Event Grid subscriptions, and queue-based destinations. It emphasizes CloudEvents 1.0+ compliance by default, security hardening (AAD, private networking, RBAC), robust filtering and resilience (dead-letter and retries), and end-to-end automation with Bicep and GitHub Actions.

## Prerequisites

- Azure subscription with permissions to deploy resources
- GitHub repository with Actions enabled (OIDC recommended)
- Azure service principal or federated credentials for deployment
- Azure CLI and Bicep CLI locally for validation
- Log Analytics workspace and Application Insights for monitoring (optional but recommended)

## Project Structure (recommended)

```
src/
├── eventgrid/
│   ├── publishers/              # Producer samples, schema docs (CloudEvents 1.0+)
│   └── subscribers/             # Subscriber samples, validation, handlers
infra/
├── modules/
│   ├── eventgrid/
│   │   ├── topic.bicep
│   │   ├── subscription-queue.bicep
│   │   ├── subscription-webhook.bicep
│   │   ├── system-topic.bicep
│   │   ├── namespace-queue.bicep     # Optional: Event Grid Namespace + Queues
│   │   └── role-assignments.bicep
│   ├── monitoring/
│   │   ├── diagnostics.bicep
│   │   └── alerts.bicep
│   └── network/
│       └── private-endpoint.bicep
└── eventgrid.main.bicep
```

## Design Principles

- CloudEvents-first: enforce CloudEvents 1.0+ for both publishing and delivery.
- Zero trust: disable local auth, require AAD, and prefer private endpoints.
- Observability: enable diagnostics and alerts for publish and delivery failures.
- Resilience: configure retries, dead-lettering, and idempotent consumers.
- Principle of least privilege: scoped RBAC for publishers and administrators.

## Security Best Practices

- Authentication/Authorization
  - Disable local auth on custom topics (`disableLocalAuth: true`).
  - Require AAD for publishing (assign role `EventGrid Data Sender` to publishers on topic scope).
  - Prefer delivery with managed identity to supported destinations (e.g., Storage Queue, Service Bus).
- Network
  - Set `publicNetworkAccess: 'Disabled'` and expose topics via Private Endpoints when feasible.
  - Use firewall/IP rules (`inboundIpRules`) only if private endpoints are not viable.
- Data Protection
  - Enforce CloudEvents 1.0+ (`inputSchema: 'CloudEventSchemaV1_0'` on topics and `eventDeliverySchema: 'CloudEventSchemaV1_0'` on subscriptions).
  - Use dead-letter destinations to a secured Blob container with managed identity.

## Filtering and Schema

- Use advanced filters to minimize unnecessary deliveries and consumer cost.
- Include CloudEvents attributes consistently:
  - `type` (domain/event), `source` (producer), `id` (unique), `time`, `subject`, `dataschema` (versioned schema URL).
- Adopt a self-describing payload with explicit `specversion` (1.0+).

## Reliability

- Configure retries (`maxDeliveryAttempts`) and TTL (`eventTimeToLiveInMinutes`).
- Always set a dead-letter destination (Blob container) and monitor it.
- Make consumers idempotent: deduplicate using CloudEvents `id` and `time`.

---

## Infrastructure as Code with Bicep

Below are detailed Bicep samples for deploying Event Grid resources with CloudEvents 1.0+ enforced.

### Main Template (infra/eventgrid.main.bicep)
```bicep
targetScope = 'resourceGroup'

@description('Region for Event Grid and dependent resources')
param location string = resourceGroup().location

@description('Custom topic name')
@minLength(3)
@maxLength(50)
param topicName string

@description('Storage account name used for dead-lettering and (optionally) Storage Queue destination')
@minLength(3)
@maxLength(24)
param storageAccountName string

@description('Event subscription name for queue destination')
@minLength(3)
@maxLength(64)
param subscriptionName string

@description('Queue name (Storage Queue) for event delivery')
@minLength(1)
@maxLength(63)
param queueName string = 'eg-events'

@description('Whether to deploy a private endpoint for the topic')
param enablePrivateEndpoint bool = false

@description('Log Analytics workspace resource ID for diagnostics (optional)')
param logAnalyticsWorkspaceId string = ''

// Storage account for dead-letter and queue (if desired)
resource sa 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    encryption: {
      services: {
        blob: { enabled: true }
        file: { enabled: true }
      }
      keySource: 'Microsoft.Storage'
    }
  }
}

// Create blob container for dead-lettering
resource dlc 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' = {
  name: '${sa.name}/default/deadletter'
  properties: {
    publicAccess: 'None'
  }
}

// Create Storage Queue for delivery
resource queue 'Microsoft.Storage/storageAccounts/queueServices/queues@2023-01-01' = {
  name: '${sa.name}/default/${queueName}'
}

// Custom Topic with CloudEvents input schema
resource topic 'Microsoft.EventGrid/topics@2024-06-01' = {
  name: topicName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    inputSchema: 'CloudEventSchemaV1_0' // Enforce CloudEvents 1.0+ for publishers
    publicNetworkAccess: 'Enabled' // Consider 'Disabled' with Private Endpoint in production
    disableLocalAuth: true // Require AAD only (no SAS)
    inboundIpRules: []
  }
}

// Event Subscription delivering to Storage Queue via Managed Identity
resource subQueue 'Microsoft.EventGrid/eventSubscriptions@2024-06-01' = {
  name: subscriptionName
  scope: topic
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    destination: {
      endpointType: 'StorageQueue'
      properties: {
        queueName: queueName
        storageAccountId: sa.id
      }
    }
    eventDeliverySchema: 'CloudEventSchemaV1_0' // Enforce CloudEvents on delivery
    retryPolicy: {
      maxDeliveryAttempts: 30
      eventTimeToLiveInMinutes: 1440 // 24h
    }
    deadLetterDestination: {
      endpointType: 'StorageBlob'
      properties: {
        resourceId: sa.id
        blobContainerName: 'deadletter'
      }
    }
    filter: {
      subjectBeginsWith: ''
      subjectEndsWith: ''
      enableAdvancedFilteringOnArrays: true
      advancedFilters: [
        // Example: deliver only specific CloudEvents types
        {
          operatorType: 'StringIn'
          key: 'type'
          values: [ 'com.example.order.created', 'com.example.order.updated' ]
        }
      ]
    }
    deliveryWithResourceIdentity: {
      // Use the topic's system assigned identity to authenticate to Storage
      identity: {
        type: 'SystemAssigned'
      }
      deliveryAttributeMappings: []
    }
  }
}

// Optional: diagnostics to Log Analytics
resource diag 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = if (!empty(logAnalyticsWorkspaceId)) {
  name: 'eg-topic-diagnostics'
  scope: topic
  properties: {
    workspaceId: logAnalyticsWorkspaceId
    logs: [
      { category: 'PublishFailures', enabled: true }
      { category: 'PublishSuccesses', enabled: true }
      { category: 'DeliveryFailures', enabled: true }
      { category: 'DeliverySuccesses', enabled: true }
      { category: 'SubscriptionDeletion', enabled: true }
    ]
    metrics: [ { category: 'AllMetrics', enabled: true } ]
  }
}

// Example role assignment: allow a producer to send events (AAD principalId required)
@description('Object ID of the AAD principal that will publish to the topic (EventGrid Data Sender)')
param publisherObjectId string = ''

var eventGridDataSenderRoleId = '4282d2a6-0d6e-4c41-b0d3-7f1a2e0a6c5b' // built-in role id

resource ra 'Microsoft.Authorization/roleAssignments@2022-04-01' = if (!empty(publisherObjectId)) {
  name: guid(topic.id, publisherObjectId, eventGridDataSenderRoleId)
  scope: topic
  properties: {
    principalId: publisherObjectId
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', eventGridDataSenderRoleId)
    principalType: 'ServicePrincipal'
  }
}

// Optional: Private Endpoint
@description('Subnet resource ID for Private Endpoint (required when enablePrivateEndpoint=true)')
param privateEndpointSubnetId string = ''

resource pe 'Microsoft.Network/privateEndpoints@2023-09-01' = if (enablePrivateEndpoint) {
  name: '${topic.name}-pe'
  location: location
  properties: {
    subnet: { id: privateEndpointSubnetId }
    privateLinkServiceConnections: [
      {
        name: 'eg-topic-pls'
        properties: {
          privateLinkServiceId: topic.id
          groupIds: ['topic']
          requestMessage: 'Private endpoint for Event Grid topic'
        }
      }
    ]
  }
}

output topicEndpoint string = topic.properties.endpoint
output topicId string = topic.id
```

### System Topic Subscription (example)
```bicep
// Create a system topic on a source resource (e.g., a Storage Account) and deliver to a Service Bus queue

@description('Existing Storage Account that will emit events')
param storageAccountId string

@description('System topic name')
param systemTopicName string

@description('Service Bus queue resource ID for delivery')
param serviceBusQueueId string

resource sysTopic 'Microsoft.EventGrid/systemTopics@2024-06-01' = {
  name: systemTopicName
  location: location
  properties: {
    source: storageAccountId
    topicType: 'microsoft.storage.storageaccounts'
  }
}

resource subSb 'Microsoft.EventGrid/systemTopics/eventSubscriptions@2024-06-01' = {
  name: '${sysTopic.name}/sb-delivery'
  properties: {
    destination: {
      endpointType: 'ServiceBusQueue'
      properties: {
        resourceId: serviceBusQueueId
      }
    }
    eventDeliverySchema: 'CloudEventSchemaV1_0'
    retryPolicy: {
      maxDeliveryAttempts: 30
      eventTimeToLiveInMinutes: 1440
    }
    deadLetterDestination: {
      endpointType: 'StorageBlob'
      properties: {
        resourceId: sa.id
        blobContainerName: 'deadletter'
      }
    }
  }
}
```

### Optional: Event Grid Namespace + Queues (check GA status and API version)
```bicep
// Note: Event Grid Namespaces and Queues may require specific API versions/regions.
// Verify availability and adjust apiVersion accordingly based on documentation.

@description('Event Grid namespace name')
param namespaceName string = 'eg-namespace'

resource egNs 'Microsoft.EventGrid/namespaces@2024-06-01-preview' = {
  name: namespaceName
  location: location
  properties: {
    publicNetworkAccess: 'Enabled'
  }
}

// Create an Event Grid Queue (pull delivery semantics)
resource egQueue 'Microsoft.EventGrid/namespaces/queues@2024-06-01-preview' = {
  name: '${egNs.name}/orders-queue'
  properties: {
    maxDeliveryCount: 30
    lockDurationInSeconds: 30
    deadLetterDestination: {
      endpointType: 'StorageBlob'
      properties: {
        resourceId: sa.id
        blobContainerName: 'deadletter'
      }
    }
  }
}
```

---

## GitHub Actions CI/CD Pipeline

The following workflow validates and deploys Event Grid resources and ensures queues (Storage Queue or Event Grid Queue) are available. It enforces CloudEvents 1.0+ via Bicep configuration.

### Workflow: .github/workflows/eventgrid-deploy.yml
```yaml
name: Deploy Event Grid

on:
  push:
    branches: [main]
    paths:
      - 'infra/**'
      - '.github/workflows/eventgrid-deploy.yml'
  pull_request:
    branches: [main]
    paths:
      - 'infra/**'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'dev'
        type: choice
        options: [dev, staging, prod]

permissions:
  id-token: write
  contents: read

env:
  BICEP_FILE: infra/eventgrid.main.bicep

jobs:
  validate:
    name: Validate Bicep
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Bicep
        uses: Azure/bicep-build-action@v1.0.1
        with:
          bicepFilePath: ${{ env.BICEP_FILE }}

  plan:
    name: What-If Plan
    runs-on: ubuntu-latest
    needs: validate
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: What-If
        uses: azure/arm-deploy@v1
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ vars.AZURE_RESOURCE_GROUP }}
          template: ${{ env.BICEP_FILE }}
          parameters: >
            location=${{ vars.AZURE_REGION }}
            topicName=${{ vars.EG_TOPIC_NAME }}
            storageAccountName=${{ vars.EG_STORAGE_ACCOUNT }}
            subscriptionName=${{ vars.EG_SUBSCRIPTION_NAME }}
            queueName=${{ vars.EG_QUEUE_NAME }}
          additionalArguments: --what-if --what-if-result-format FullResourcePayloads

  deploy:
    name: Deploy Event Grid
    runs-on: ubuntu-latest
    needs: validate
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Deploy Bicep
        uses: azure/arm-deploy@v1
        id: deploy
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ vars.AZURE_RESOURCE_GROUP }}
          template: ${{ env.BICEP_FILE }}
          parameters: >
            location=${{ vars.AZURE_REGION }}
            topicName=${{ vars.EG_TOPIC_NAME }}
            storageAccountName=${{ vars.EG_STORAGE_ACCOUNT }}
            subscriptionName=${{ vars.EG_SUBSCRIPTION_NAME }}
            queueName=${{ vars.EG_QUEUE_NAME }}
          deploymentName: eg-${{ github.run_number }}
      - name: Output Topic Endpoint
        run: echo "Topic Endpoint => ${{ steps.deploy.outputs.topicEndpoint }}"

  ensure-queue:
    name: Ensure Queue Exists (Storage Queue)
    runs-on: ubuntu-latest
    needs: deploy
    steps:
      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Create Queue if missing
        run: |
          az storage queue create \
            --name ${{ vars.EG_QUEUE_NAME }} \
            --account-name ${{ vars.EG_STORAGE_ACCOUNT }} \
            --auth-mode login
```

Notes:
- The Bicep template enforces CloudEvents 1.0+ for both input and delivery.
- The workflow uses OIDC (id-token) for secure deployments.
- `--auth-mode login` ensures MSI/AAD auth for queue creation.

---

## Monitoring and Alerting

- Enable Diagnostic Settings on the topic for categories: PublishSuccesses, PublishFailures, DeliverySuccesses, DeliveryFailures.
- Send diagnostics to Log Analytics and set up Azure Monitor alerts:

```bicep
@description('Alert when DeliveryFailures exceed threshold')
param alertEmails array = ['alerts@contoso.com']

resource topicRef 'Microsoft.EventGrid/topics@2024-06-01' existing = {
  name: topicName
}

resource actionGroup 'Microsoft.Insights/actionGroups@2023-01-01' = {
  name: '${topicName}-ag'
  location: 'global'
  properties: {
    groupShortName: 'EGAlerts'
    enabled: true
    emailReceivers: [for e in alertEmails: {
      name: replace(e, '@', '-')
      emailAddress: e
      useCommonAlertSchema: true
    }]
  }
}

resource deliveryFailuresAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: '${topicName}-delivery-failures'
  location: 'global'
  properties: {
    description: 'Delivery failures > 0 in 5 minutes'
    severity: 2
    enabled: true
    scopes: [topicRef.id]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT5M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'DeliveryFailures'
          metricName: 'DeliveryFailedCount'
          operator: 'GreaterThan'
          threshold: 0
          timeAggregation: 'Total'
        }
      ]
    }
    actions: [ { actionGroupId: actionGroup.id } ]
  }
}
```

---

## Development Guidance (CloudEvents 1.0+)

- Producers must publish CloudEvents-compliant messages. Example (HTTP binary mode):
```http
POST {topic-endpoint} HTTP/1.1
Content-Type: application/json; charset=utf-8
ce-specversion: 1.0
ce-type: com.example.order.created
ce-source: /sales/orders
ce-id: 6f1e6a0d-5143-4c6a-9f34-4ef0c2a3f54d
ce-time: 2025-09-03T10:00:00Z

{ "orderId": "12345", "total": 99.50 }
```

- Subscribers should validate `specversion`, `type`, `id`, and `time`, and deduplicate using `id`.
- Use advanced filters on `type`, `subject`, or custom attributes to reduce noise.

---

## Additional References

- Azure Event Grid overview: https://learn.microsoft.com/azure/event-grid/overview
- CloudEvents in Event Grid: https://learn.microsoft.com/azure/event-grid/cloudevents-schema
- Event Grid event schemas: https://learn.microsoft.com/azure/event-grid/event-schema
- Create custom topics and subscriptions: https://learn.microsoft.com/azure/event-grid/custom-topics
- Advanced filtering: https://learn.microsoft.com/azure/event-grid/event-filtering
- Retry and dead-letter: https://learn.microsoft.com/azure/event-grid/delivery-and-retry
- Private endpoints for Event Grid: https://learn.microsoft.com/azure/event-grid/private-endpoints
- Monitor Event Grid: https://learn.microsoft.com/azure/event-grid/monitor-azure-event-grid
- Bicep templates for Event Grid: https://learn.microsoft.com/azure/templates/microsoft.eventgrid/allversions
- System topics: https://learn.microsoft.com/azure/event-grid/system-topics
- Security and authentication: https://learn.microsoft.com/azure/event-grid/security-authentication
- Event Grid namespaces and queues: https://learn.microsoft.com/azure/event-grid/namespaces-overview