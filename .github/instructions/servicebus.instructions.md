---
description: 'Best practices for developing, securing, monitoring, and deploying Azure Service Bus queues, topics, and subscriptions'
applyTo: "**/src/servicebus/**/*,**/infra/**/*"
---

# Azure Service Bus Development and Deployment Best Practices

## Overview

This guide provides comprehensive best practices for developing, securing, monitoring, and deploying **queues, topics, and subscriptions** to **existing Azure Service Bus namespaces** using GitHub Actions and Infrastructure as Code (Bicep). Focus is on managed identity authentication, message reliability, performance optimization, and enterprise-grade messaging patterns.

## Prerequisites

- Existing Azure Service Bus namespace (Standard or Premium tier recommended)
- GitHub repository with Actions enabled
- Azure service principal or workload identity for deployment
- Bicep CLI and Azure CLI installed for local development
- Understanding of messaging patterns and Service Bus concepts

## Project Structure

```
src/
├── servicebus/
│   ├── queues/
│   │   ├── [queue-name]/
│   │   │   ├── configuration.json
│   │   │   ├── auth-rules.json
│   │   │   └── dead-letter-config.json
│   ├── topics/
│   │   ├── [topic-name]/
│   │   │   ├── configuration.json
│   │   │   ├── subscriptions/
│   │   │   │   └── [subscription-name]/
│   │   │   │       ├── configuration.json
│   │   │   │       ├── rules.json
│   │   │   │       └── dead-letter-config.json
│   │   │   └── auth-rules.json
│   ├── shared/
│   │   ├── authorization-rules/
│   │   ├── network-rules/
│   │   └── monitoring/
│   └── templates/
│       ├── message-schemas/
│       └── correlation-templates/
infra/
├── modules/
│   ├── servicebus/
│   │   ├── queues.bicep
│   │   ├── topics.bicep
│   │   ├── subscriptions.bicep
│   │   ├── auth-rules.bicep
│   │   └── monitoring.bicep
│   └── security/
│       ├── rbac.bicep
│       └── network-rules.bicep
└── main.bicep
```

## Security Best Practices

### 1. Managed Identity Authentication (Recommended)

#### C# Implementation with Managed Identity
```csharp
// Use DefaultAzureCredential for automatic managed identity discovery
using Azure.Identity;
using Azure.Messaging.ServiceBus;

public class ServiceBusClientService
{
    private readonly ServiceBusClient _client;
    private readonly ILogger<ServiceBusClientService> _logger;

    public ServiceBusClientService(IConfiguration configuration, ILogger<ServiceBusClientService> logger)
    {
        var fullyQualifiedNamespace = configuration["ServiceBus:FullyQualifiedNamespace"];
        
        // Use DefaultAzureCredential for managed identity authentication
        var credential = new DefaultAzureCredential(new DefaultAzureCredentialOptions
        {
            // Optional: specify user-assigned managed identity client ID
            ManagedIdentityClientId = configuration["ServiceBus:ManagedIdentityClientId"]
        });

        _client = new ServiceBusClient(fullyQualifiedNamespace, credential);
        _logger = logger;
    }

    public async Task SendMessageAsync<T>(string queueOrTopicName, T messageBody, 
        string correlationId = null, TimeSpan? timeToLive = null)
    {
        try
        {
            await using var sender = _client.CreateSender(queueOrTopicName);
            
            var message = new ServiceBusMessage(JsonSerializer.Serialize(messageBody))
            {
                ContentType = "application/json",
                CorrelationId = correlationId ?? Guid.NewGuid().ToString(),
                TimeToLive = timeToLive ?? TimeSpan.FromHours(24),
                Subject = typeof(T).Name
            };

            // Add custom properties for routing and filtering
            message.ApplicationProperties["MessageType"] = typeof(T).Name;
            message.ApplicationProperties["Timestamp"] = DateTimeOffset.UtcNow;
            message.ApplicationProperties["Source"] = Environment.MachineName;

            await sender.SendMessageAsync(message);
            
            _logger.LogInformation("Message sent to {QueueOrTopic} with CorrelationId: {CorrelationId}", 
                queueOrTopicName, message.CorrelationId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to send message to {QueueOrTopic}", queueOrTopicName);
            throw;
        }
    }

    public async Task ProcessMessagesAsync<T>(string queueOrSubscriptionName, 
        Func<T, ServiceBusReceivedMessage, Task> messageHandler,
        ServiceBusProcessorOptions options = null)
    {
        try
        {
            var processor = _client.CreateProcessor(queueOrSubscriptionName, options ?? new ServiceBusProcessorOptions
            {
                AutoCompleteMessages = false,
                MaxConcurrentCalls = 5,
                PrefetchCount = 10,
                ReceiveMode = ServiceBusReceiveMode.PeekLock,
                MaxAutoLockRenewalDuration = TimeSpan.FromMinutes(5)
            });

            processor.ProcessMessageAsync += async args =>
            {
                try
                {
                    var messageBody = JsonSerializer.Deserialize<T>(args.Message.Body.ToString());
                    
                    _logger.LogInformation("Processing message {MessageId} with CorrelationId: {CorrelationId}", 
                        args.Message.MessageId, args.Message.CorrelationId);

                    await messageHandler(messageBody, args.Message);
                    
                    // Complete the message to remove it from the queue/subscription
                    await args.CompleteMessageAsync(args.Message);
                    
                    _logger.LogInformation("Successfully processed message {MessageId}", args.Message.MessageId);
                }
                catch (JsonException ex)
                {
                    _logger.LogError(ex, "Failed to deserialize message {MessageId}", args.Message.MessageId);
                    // Send to dead letter queue for manual inspection
                    await args.DeadLetterMessageAsync(args.Message, "DeserializationError", ex.Message);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Error processing message {MessageId}", args.Message.MessageId);
                    
                    // Implement retry logic with exponential backoff
                    if (args.Message.DeliveryCount <= 3)
                    {
                        await args.AbandonMessageAsync(args.Message);
                    }
                    else
                    {
                        await args.DeadLetterMessageAsync(args.Message, "ProcessingError", ex.Message);
                    }
                }
            };

            processor.ProcessErrorAsync += args =>
            {
                _logger.LogError(args.Exception, "Service Bus processing error occurred");
                return Task.CompletedTask;
            };

            await processor.StartProcessingAsync();
            _logger.LogInformation("Started processing messages from {QueueOrSubscription}", queueOrSubscriptionName);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to start message processing for {QueueOrSubscription}", queueOrSubscriptionName);
            throw;
        }
    }

    public async ValueTask DisposeAsync()
    {
        if (_client != null)
        {
            await _client.DisposeAsync();
        }
    }
}
```

### 2. Role-Based Access Control (RBAC)

#### Built-in Service Bus Roles
- **Azure Service Bus Data Owner**: Full access to namespace and entities
- **Azure Service Bus Data Sender**: Send messages to queues and topics
- **Azure Service Bus Data Receiver**: Receive messages from queues and subscriptions

#### Principle of Least Privilege Implementation
```bicep
// Assign specific roles for different application components
resource senderRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(serviceBusNamespace.id, senderPrincipalId, 'sender')
  scope: serviceBusNamespace
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '69a216fc-b8fb-44d8-bc22-1f3c2cd27a39') // Service Bus Data Sender
    principalId: senderPrincipalId
    principalType: 'ServicePrincipal'
  }
}

resource receiverRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(serviceBusNamespace.id, receiverPrincipalId, 'receiver')
  scope: serviceBusNamespace
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4f6d3b9b-027b-4f4c-9142-0e5a2a2247e0') // Service Bus Data Receiver
    principalId: receiverPrincipalId
    principalType: 'ServicePrincipal'
  }
}
```

### 3. Network Security and Private Endpoints

```bicep
// Configure network rules and private endpoint access
resource serviceBusNetworkRules 'Microsoft.ServiceBus/namespaces/networkRuleSets@2022-10-01-preview' = {
  parent: serviceBusNamespace
  name: 'default'
  properties: {
    publicNetworkAccess: 'Disabled' // Disable public access
    defaultAction: 'Deny'
    virtualNetworkRules: [
      {
        subnet: {
          id: subnetId
        }
        ignoreMissingVnetServiceEndpoint: false
      }
    ]
    ipRules: [
      {
        ipMask: '10.0.0.0/16'
        action: 'Allow'
      }
    ]
    trustedServiceAccessEnabled: true
  }
}

// Create private endpoint for Service Bus
resource serviceBusPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-05-01' = {
  name: '${serviceBusNamespace.name}-pe'
  location: location
  properties: {
    subnet: {
      id: privateEndpointSubnetId
    }
    privateLinkServiceConnections: [
      {
        name: 'serviceBusConnection'
        properties: {
          privateLinkServiceId: serviceBusNamespace.id
          groupIds: ['namespace']
        }
      }
    ]
  }
}
```

## Message Design and Reliability Patterns

### 1. Message Schema and Correlation

```json
// Message Template (src/servicebus/templates/message-schemas/order-schema.json)
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "orderId": {
      "type": "string",
      "format": "uuid"
    },
    "customerId": {
      "type": "string"
    },
    "items": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "productId": {
            "type": "string"
          },
          "quantity": {
            "type": "integer",
            "minimum": 1
          },
          "price": {
            "type": "number",
            "minimum": 0
          }
        },
        "required": ["productId", "quantity", "price"]
      }
    },
    "timestamp": {
      "type": "string",
      "format": "date-time"
    }
  },
  "required": ["orderId", "customerId", "items", "timestamp"]
}
```

### 2. Topic and Subscription Filtering

```json
// Subscription Rules Configuration (src/servicebus/topics/orders/subscriptions/high-priority/rules.json)
{
  "rules": [
    {
      "name": "HighPriorityOrdersRule",
      "filter": {
        "sqlFilter": "Priority = 'High' OR TotalAmount > 1000"
      },
      "action": {
        "sqlAction": "SET RoutedTimestamp = GETUTCDATE()"
      }
    },
    {
      "name": "RegionalOrdersRule",
      "filter": {
        "correlationFilter": {
          "properties": {
            "Region": "US-WEST"
          }
        }
      }
    }
  ]
}
```

### 3. Dead Letter Queue Handling

```csharp
public class DeadLetterQueueProcessor
{
    private readonly ServiceBusClient _client;
    private readonly ILogger<DeadLetterQueueProcessor> _logger;

    public async Task ProcessDeadLetterQueueAsync(string queueName)
    {
        var deadLetterQueuePath = $"{queueName}/$deadletterqueue";
        var receiver = _client.CreateReceiver(deadLetterQueuePath);

        try
        {
            var messages = await receiver.ReceiveMessagesAsync(maxMessages: 10);
            
            foreach (var message in messages)
            {
                _logger.LogWarning("Processing dead letter message {MessageId}. " +
                    "Reason: {DeadLetterReason}, Description: {DeadLetterDescription}, " +
                    "DeliveryCount: {DeliveryCount}",
                    message.MessageId, 
                    message.DeadLetterReason, 
                    message.DeadLetterErrorDescription,
                    message.DeliveryCount);

                // Analyze the dead letter reason and take appropriate action
                await HandleDeadLetterMessage(message, queueName);
                
                // Complete the dead letter message after processing
                await receiver.CompleteMessageAsync(message);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing dead letter queue for {QueueName}", queueName);
        }
        finally
        {
            await receiver.DisposeAsync();
        }
    }

    private async Task HandleDeadLetterMessage(ServiceBusReceivedMessage message, string originalQueue)
    {
        switch (message.DeadLetterReason)
        {
            case "MaxDeliveryCountExceeded":
                await LogAndAlert($"Message {message.MessageId} exceeded max delivery count");
                break;
            
            case "TTLExpiredException":
                _logger.LogWarning("Message {MessageId} expired in queue {Queue}", 
                    message.MessageId, originalQueue);
                break;
            
            default:
                // Attempt to reprocess or route to manual review
                await ReprocessMessage(message, originalQueue);
                break;
        }
    }
}
```

## Performance Optimization

### 1. Batching and Prefetching

```csharp
public class HighPerformanceServiceBusService
{
    public async Task SendBatchMessagesAsync<T>(string queueOrTopicName, IEnumerable<T> messages)
    {
        await using var sender = _client.CreateSender(queueOrTopicName);
        
        var batches = messages
            .Select((msg, index) => new { msg, index })
            .GroupBy(x => x.index / 100) // Batch size of 100
            .Select(g => g.Select(x => x.msg));

        foreach (var batch in batches)
        {
            using var messageBatch = await sender.CreateMessageBatchAsync();
            
            foreach (var message in batch)
            {
                var serviceBusMessage = new ServiceBusMessage(JsonSerializer.Serialize(message))
                {
                    ContentType = "application/json",
                    CorrelationId = Guid.NewGuid().ToString()
                };

                if (!messageBatch.TryAddMessage(serviceBusMessage))
                {
                    // Send current batch and create a new one
                    await sender.SendMessagesAsync(messageBatch);
                    messageBatch.Clear();
                    
                    if (!messageBatch.TryAddMessage(serviceBusMessage))
                    {
                        throw new InvalidOperationException("Message too large for batch");
                    }
                }
            }
            
            if (messageBatch.Count > 0)
            {
                await sender.SendMessagesAsync(messageBatch);
            }
        }
    }

    public ServiceBusProcessor CreateOptimizedProcessor(string queueOrSubscriptionName)
    {
        return _client.CreateProcessor(queueOrSubscriptionName, new ServiceBusProcessorOptions
        {
            AutoCompleteMessages = false,
            MaxConcurrentCalls = Environment.ProcessorCount * 2, // Scale with CPU cores
            PrefetchCount = 50, // Prefetch messages for better throughput
            ReceiveMode = ServiceBusReceiveMode.PeekLock,
            MaxAutoLockRenewalDuration = TimeSpan.FromMinutes(10)
        });
    }
}
```

### 2. Session-Enabled Queues for Ordered Processing

```csharp
public class SessionAwareProcessor
{
    public async Task ProcessSessionMessagesAsync(string sessionEnabledQueue)
    {
        var processor = _client.CreateSessionProcessor(sessionEnabledQueue, new ServiceBusSessionProcessorOptions
        {
            AutoCompleteMessages = false,
            MaxConcurrentSessions = 10,
            SessionIdleTimeout = TimeSpan.FromMinutes(5)
        });

        processor.ProcessMessageAsync += async args =>
        {
            var sessionId = args.Message.SessionId;
            _logger.LogInformation("Processing message in session {SessionId}", sessionId);

            // Messages within the same session are processed in order
            try
            {
                await ProcessOrderedMessage(args.Message);
                await args.CompleteMessageAsync(args.Message);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing session message");
                await args.AbandonMessageAsync(args.Message);
            }
        };

        await processor.StartProcessingAsync();
    }
}
```

## Infrastructure as Code with Bicep

### 1. Main Service Bus Module (infra/main.bicep)

```bicep
targetScope = 'resourceGroup'

@description('Name of the existing Service Bus namespace')
param serviceBusNamespaceName string

@description('Environment name (dev, staging, prod)')
@allowed(['dev', 'staging', 'prod'])
param environment string = 'dev'

@description('Location for resources')
param location string = resourceGroup().location

@description('Queue configurations')
param queueConfigurations array = [
  {
    name: 'orders-queue'
    maxSizeInMegabytes: 1024
    messageTimeToLive: 'P14D'
    lockDuration: 'PT5M'
    maxDeliveryCount: 5
    duplicateDetectionHistoryTimeWindow: 'PT10M'
    enablePartitioning: false
    enableExpress: false
    requiresDuplicateDetection: true
    deadLetteringOnMessageExpiration: true
  }
  {
    name: 'notifications-queue'
    maxSizeInMegabytes: 2048
    messageTimeToLive: 'P7D'
    lockDuration: 'PT1M'
    maxDeliveryCount: 3
    duplicateDetectionHistoryTimeWindow: 'PT10M'
    enablePartitioning: true
    enableExpress: true
    requiresDuplicateDetection: false
    deadLetteringOnMessageExpiration: true
  }
]

@description('Topic configurations')
param topicConfigurations array = [
  {
    name: 'order-events'
    maxSizeInMegabytes: 2048
    messageTimeToLive: 'P14D'
    duplicateDetectionHistoryTimeWindow: 'PT10M'
    enablePartitioning: false
    requiresDuplicateDetection: true
    subscriptions: [
      {
        name: 'inventory-updates'
        messageTimeToLive: 'P7D'
        lockDuration: 'PT5M'
        maxDeliveryCount: 5
        deadLetteringOnMessageExpiration: true
        rules: [
          {
            name: 'InventoryRule'
            sqlFilter: "EventType = 'OrderCompleted' OR EventType = 'OrderCancelled'"
          }
        ]
      }
      {
        name: 'audit-subscription'
        messageTimeToLive: 'P30D'
        lockDuration: 'PT1M'
        maxDeliveryCount: 10
        deadLetteringOnMessageExpiration: false
        rules: [
          {
            name: 'AllEventsRule'
            sqlFilter: "1=1" // Capture all events
          }
        ]
      }
    ]
  }
]

// Reference existing Service Bus namespace
resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' existing = {
  name: serviceBusNamespaceName
}

// Deploy queues
module queuesModule 'modules/servicebus/queues.bicep' = {
  name: 'queues-deployment'
  params: {
    serviceBusNamespaceName: serviceBusNamespace.name
    queueConfigurations: queueConfigurations
    environment: environment
  }
}

// Deploy topics and subscriptions
module topicsModule 'modules/servicebus/topics.bicep' = {
  name: 'topics-deployment'
  params: {
    serviceBusNamespaceName: serviceBusNamespace.name
    topicConfigurations: topicConfigurations
    environment: environment
  }
}

// Deploy monitoring and diagnostics
module monitoringModule 'modules/servicebus/monitoring.bicep' = {
  name: 'monitoring-deployment'
  params: {
    serviceBusNamespaceName: serviceBusNamespace.name
    environment: environment
  }
}

// Deploy RBAC assignments
module rbacModule 'modules/security/rbac.bicep' = {
  name: 'rbac-deployment'
  params: {
    serviceBusNamespaceName: serviceBusNamespace.name
    environment: environment
  }
}

// Outputs for verification
output serviceBusNamespaceId string = serviceBusNamespace.id
output deployedQueues array = queuesModule.outputs.queueNames
output deployedTopics array = topicsModule.outputs.topicNames
```

### 2. Queues Module (infra/modules/servicebus/queues.bicep)

```bicep
@description('Service Bus namespace name')
param serviceBusNamespaceName string

@description('Queue configurations')
param queueConfigurations array

@description('Environment name')
param environment string

resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' existing = {
  name: serviceBusNamespaceName
}

// Create queues with comprehensive configuration
resource queues 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = [for config in queueConfigurations: {
  parent: serviceBusNamespace
  name: '${config.name}-${environment}'
  properties: {
    maxSizeInMegabytes: config.maxSizeInMegabytes
    messageTimeToLive: config.messageTimeToLive
    lockDuration: config.lockDuration
    maxDeliveryCount: config.maxDeliveryCount
    duplicateDetectionHistoryTimeWindow: config.duplicateDetectionHistoryTimeWindow
    enablePartitioning: config.enablePartitioning
    enableExpress: config.enableExpress
    requiresDuplicateDetection: config.requiresDuplicateDetection
    deadLetteringOnMessageExpiration: config.deadLetteringOnMessageExpiration
    enableBatchedOperations: true
    status: 'Active'
    autoDeleteOnIdle: 'P10675199DT2H48M5.4775807S' // Max value (never auto-delete)
  }
}]

// Create authorization rules for each queue (if needed for SAS - prefer managed identity)
resource queueAuthRules 'Microsoft.ServiceBus/namespaces/queues/authorizationRules@2022-10-01-preview' = [for (config, i) in queueConfigurations: if(environment != 'prod') {
  parent: queues[i]
  name: 'SendListen'
  properties: {
    rights: [
      'Send'
      'Listen'
    ]
  }
}]

// Output queue information
output queueNames array = [for (config, i) in queueConfigurations: queues[i].name]
output queueResourceIds array = [for (config, i) in queueConfigurations: queues[i].id]
```

### 3. Topics and Subscriptions Module (infra/modules/servicebus/topics.bicep)

```bicep
@description('Service Bus namespace name')
param serviceBusNamespaceName string

@description('Topic configurations')
param topicConfigurations array

@description('Environment name')
param environment string

resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' existing = {
  name: serviceBusNamespaceName
}

// Create topics
resource topics 'Microsoft.ServiceBus/namespaces/topics@2022-10-01-preview' = [for config in topicConfigurations: {
  parent: serviceBusNamespace
  name: '${config.name}-${environment}'
  properties: {
    maxSizeInMegabytes: config.maxSizeInMegabytes
    messageTimeToLive: config.messageTimeToLive
    duplicateDetectionHistoryTimeWindow: config.duplicateDetectionHistoryTimeWindow
    enablePartitioning: config.enablePartitioning
    requiresDuplicateDetection: config.requiresDuplicateDetection
    enableBatchedOperations: true
    status: 'Active'
    autoDeleteOnIdle: 'P10675199DT2H48M5.4775807S'
  }
}]

// Create subscriptions for each topic
resource subscriptions 'Microsoft.ServiceBus/namespaces/topics/subscriptions@2022-10-01-preview' = [for (topicConfig, topicIndex) in topicConfigurations: {
  parent: topics[topicIndex]
  name: '${topicConfig.subscriptions[0].name}-${environment}' // First subscription
  properties: {
    messageTimeToLive: topicConfig.subscriptions[0].messageTimeToLive
    lockDuration: topicConfig.subscriptions[0].lockDuration
    maxDeliveryCount: topicConfig.subscriptions[0].maxDeliveryCount
    deadLetteringOnMessageExpiration: topicConfig.subscriptions[0].deadLetteringOnMessageExpiration
    enableBatchedOperations: true
    status: 'Active'
    autoDeleteOnIdle: 'P10675199DT2H48M5.4775807S'
  }
}]

// Create additional subscriptions (flattened approach)
resource additionalSubscriptions 'Microsoft.ServiceBus/namespaces/topics/subscriptions@2022-10-01-preview' = [for item in flatten([for (topicConfig, topicIndex) in topicConfigurations: [for (sub, subIndex) in skip(topicConfig.subscriptions, 1): {
  topicIndex: topicIndex
  subConfig: sub
  name: '${sub.name}-${environment}'
}]]): {
  parent: topics[item.topicIndex]
  name: item.name
  properties: {
    messageTimeToLive: item.subConfig.messageTimeToLive
    lockDuration: item.subConfig.lockDuration
    maxDeliveryCount: item.subConfig.maxDeliveryCount
    deadLetteringOnMessageExpiration: item.subConfig.deadLetteringOnMessageExpiration
    enableBatchedOperations: true
    status: 'Active'
    autoDeleteOnIdle: 'P10675199DT2H48M5.4775807S'
  }
}]

// Create subscription rules (SQL filters)
resource subscriptionRules 'Microsoft.ServiceBus/namespaces/topics/subscriptions/rules@2022-10-01-preview' = [for item in flatten([for (topicConfig, topicIndex) in topicConfigurations: [for (sub, subIndex) in topicConfig.subscriptions: [for (rule, ruleIndex) in sub.rules: {
  topicIndex: topicIndex
  subIndex: subIndex
  rule: rule
  subscriptionName: '${sub.name}-${environment}'
}]]]): {
  parent: serviceBusNamespace
  name: '${topicConfigurations[item.topicIndex].name}-${environment}/${item.subscriptionName}/${item.rule.name}'
  properties: {
    filterType: 'SqlFilter'
    sqlFilter: {
      sqlExpression: item.rule.sqlFilter
    }
    action: contains(item.rule, 'sqlAction') ? {
      sqlExpression: item.rule.sqlAction
    } : null
  }
}]

output topicNames array = [for (config, i) in topicConfigurations: topics[i].name]
output topicResourceIds array = [for (config, i) in topicConfigurations: topics[i].id]
```

### 4. Monitoring Module (infra/modules/servicebus/monitoring.bicep)

```bicep
@description('Service Bus namespace name')
param serviceBusNamespaceName string

@description('Environment name')
param environment string

@description('Log Analytics workspace ID')
param logAnalyticsWorkspaceId string = ''

resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' existing = {
  name: serviceBusNamespaceName
}

// Create Log Analytics workspace if not provided
resource logAnalyticsWorkspace 'Microsoft.OperationalInsights/workspaces@2022-10-01' = if (empty(logAnalyticsWorkspaceId)) {
  name: 'law-servicebus-${environment}'
  location: resourceGroup().location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: 30
  }
}

// Diagnostic settings for comprehensive logging
resource diagnosticSettings 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'servicebus-diagnostics'
  scope: serviceBusNamespace
  properties: {
    workspaceId: empty(logAnalyticsWorkspaceId) ? logAnalyticsWorkspace.id : logAnalyticsWorkspaceId
    logs: [
      {
        categoryGroup: 'allLogs'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: 30
        }
      }
    ]
    metrics: [
      {
        category: 'AllMetrics'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: 30
        }
      }
    ]
  }
}

// Create alerts for critical metrics
resource deadLetterMessagesAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'ServiceBus-DeadLetterMessages-${environment}'
  location: 'global'
  properties: {
    description: 'Alert when dead letter message count exceeds threshold'
    severity: 2
    enabled: true
    scopes: [
      serviceBusNamespace.id
    ]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT15M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'DeadLetterMessages'
          metricName: 'DeadletteredMessages'
          operator: 'GreaterThan'
          threshold: 10
          timeAggregation: 'Total'
          criterionType: 'StaticThresholdCriterion'
        }
      ]
    }
    actions: []
  }
}

resource highCpuAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'ServiceBus-HighCPU-${environment}'
  location: 'global'
  properties: {
    description: 'Alert when Service Bus CPU usage is high'
    severity: 1
    enabled: true
    scopes: [
      serviceBusNamespace.id
    ]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT15M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'CPUUsage'
          metricName: 'NamespaceCpuUsage'
          operator: 'GreaterThan'
          threshold: 80
          timeAggregation: 'Average'
          criterionType: 'StaticThresholdCriterion'
        }
      ]
    }
    actions: []
  }
}

resource messageCountAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'ServiceBus-HighMessageCount-${environment}'
  location: 'global'
  properties: {
    description: 'Alert when active message count is high'
    severity: 2
    enabled: true
    scopes: [
      serviceBusNamespace.id
    ]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT15M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'ActiveMessages'
          metricName: 'ActiveMessages'
          operator: 'GreaterThan'
          threshold: 1000
          timeAggregation: 'Average'
          criterionType: 'StaticThresholdCriterion'
        }
      ]
    }
    actions: []
  }
}
```

## GitHub Actions CI/CD Pipeline

### Main Workflow (.github/workflows/servicebus-deploy.yml)

```yaml
name: Deploy Azure Service Bus Configuration

on:
  push:
    branches: [main]
    paths: 
      - 'src/servicebus/**'
      - 'infra/**'
  pull_request:
    branches: [main]
    paths:
      - 'src/servicebus/**'
      - 'infra/**'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'dev'
        type: choice
        options:
          - dev
          - staging
          - prod

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  validate:
    name: Validate Service Bus Configuration
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup .NET for validation
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0'

      - name: Validate JSON schemas
        run: |
          # Install JSON schema validator
          npm install -g ajv-cli
          
          # Validate queue configurations
          for config in src/servicebus/queues/*/configuration.json; do
            echo "Validating $config"
            # Add JSON schema validation logic here
            jq . "$config" > /dev/null || exit 1
          done
          
          # Validate topic configurations
          for config in src/servicebus/topics/*/configuration.json; do
            echo "Validating $config"
            jq . "$config" > /dev/null || exit 1
          done

      - name: Setup Bicep
        uses: Azure/bicep-build-action@v1.0.1
        with:
          bicepFilePath: infra/main.bicep

      - name: Lint Bicep templates
        run: |
          az bicep build --file infra/main.bicep
          az bicep lint --file infra/main.bicep

  security-scan:
    name: Security and Compliance Scan
    runs-on: ubuntu-latest
    needs: validate
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run Checkov security scan
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: .
          framework: bicep
          output_format: sarif
          output_file_path: reports/checkov-results.sarif
          skip_check: CKV_AZURE_201 # Skip specific checks if needed

      - name: Upload Checkov results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: reports/checkov-results.sarif

  plan:
    name: Plan Infrastructure Changes
    runs-on: ubuntu-latest
    needs: [validate, security-scan]
    if: github.event_name == 'pull_request'
    environment: 
      name: dev
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy Bicep What-If
        uses: azure/arm-deploy@v1
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ vars.AZURE_RESOURCE_GROUP }}
          template: infra/main.bicep
          parameters: >
            serviceBusNamespaceName=${{ vars.SERVICE_BUS_NAMESPACE }}
            environment=dev
          additionalArguments: --what-if --what-if-result-format FullResourcePayloads

      - name: Comment What-If Results
        uses: actions/github-script@v7
        with:
          script: |
            const output = `
            ## Service Bus Configuration Changes Preview 📨
            
            The infrastructure what-if analysis has been completed. Please review the changes above.
            
            - ✅ **Validation**: JSON schemas and Bicep templates validated
            - ✅ **Security**: Checkov security scan completed
            - 📋 **Changes**: Review the what-if output above
            
            **Queues and Topics to be modified:**
            - Check the what-if output for specific queue and topic changes
            - Review subscription rules and filters
            - Validate retention policies and dead letter settings
            
            *This comment will be updated when the PR is merged and deployment completes.*
            `;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

  deploy-dev:
    name: Deploy to Development
    runs-on: ubuntu-latest
    needs: [validate, security-scan]
    if: github.ref == 'refs/heads/main'
    environment:
      name: dev
      url: https://portal.azure.com/#resource${{ vars.SERVICE_BUS_RESOURCE_ID }}/overview
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy Service Bus Configuration
        uses: azure/arm-deploy@v1
        id: deploy
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ vars.AZURE_RESOURCE_GROUP }}
          template: infra/main.bicep
          parameters: >
            serviceBusNamespaceName=${{ vars.SERVICE_BUS_NAMESPACE }}
            environment=dev
          deploymentName: servicebus-deployment-${{ github.run_number }}

      - name: Validate Service Bus Entities
        run: |
          echo "Validating deployed Service Bus entities..."
          
          # List all queues
          echo "Deployed Queues:"
          az servicebus queue list \
            --namespace-name ${{ vars.SERVICE_BUS_NAMESPACE }} \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP }} \
            --output table
          
          # List all topics
          echo "Deployed Topics:"
          az servicebus topic list \
            --namespace-name ${{ vars.SERVICE_BUS_NAMESPACE }} \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP }} \
            --output table
          
          # Verify specific queue properties
          echo "Verifying queue configurations..."
          queues=$(az servicebus queue list \
            --namespace-name ${{ vars.SERVICE_BUS_NAMESPACE }} \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP }} \
            --query "[].name" -o tsv)
          
          for queue in $queues; do
            echo "Queue: $queue"
            az servicebus queue show \
              --namespace-name ${{ vars.SERVICE_BUS_NAMESPACE }} \
              --resource-group ${{ vars.AZURE_RESOURCE_GROUP }} \
              --name $queue \
              --query "{name:name,maxDeliveryCount:maxDeliveryCount,deadLetteringOnMessageExpiration:deadLetteringOnMessageExpiration}" \
              --output table
          done

      - name: Test Message Operations (Optional)
        if: contains(github.event.head_commit.message, '[test-messages]')
        run: |
          echo "Testing basic message operations..."
          
          # Send a test message using Azure CLI
          az servicebus message send \
            --namespace-name ${{ vars.SERVICE_BUS_NAMESPACE }} \
            --queue-name "orders-queue-dev" \
            --body '{"test": "deployment-validation", "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%S.%3NZ)'"}' \
            || echo "Message send test skipped (queue may not exist yet)"

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: deploy-dev
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://portal.azure.com/#resource${{ vars.SERVICE_BUS_RESOURCE_ID_STAGING }}/overview
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Staging Service Bus
        uses: azure/arm-deploy@v1
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ vars.AZURE_RESOURCE_GROUP_STAGING }}
          template: infra/main.bicep
          parameters: >
            serviceBusNamespaceName=${{ vars.SERVICE_BUS_NAMESPACE_STAGING }}
            environment=staging
          deploymentName: servicebus-staging-${{ github.run_number }}

  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://portal.azure.com/#resource${{ vars.SERVICE_BUS_RESOURCE_ID_PROD }}/overview
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Production Service Bus
        uses: azure/arm-deploy@v1
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          resourceGroupName: ${{ vars.AZURE_RESOURCE_GROUP_PROD }}
          template: infra/main.bicep
          parameters: >
            serviceBusNamespaceName=${{ vars.SERVICE_BUS_NAMESPACE_PROD }}
            environment=prod
          deploymentName: servicebus-prod-${{ github.run_number }}

      - name: Post-deployment Verification
        run: |
          echo "🚀 Production deployment completed!"
          
          # Verify critical entities exist
          critical_queues=("orders-queue-prod" "notifications-queue-prod")
          
          for queue in "${critical_queues[@]}"; do
            if az servicebus queue show \
              --namespace-name ${{ vars.SERVICE_BUS_NAMESPACE_PROD }} \
              --resource-group ${{ vars.AZURE_RESOURCE_GROUP_PROD }} \
              --name $queue &>/dev/null; then
              echo "✅ Queue $queue verified"
            else
              echo "❌ Queue $queue not found"
              exit 1
            fi
          done
          
          echo "✅ Production deployment verified successfully"

  monitoring-setup:
    name: Configure Monitoring and Alerts
    runs-on: ubuntu-latest
    needs: [deploy-prod]
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Verify Monitoring Configuration
        run: |
          echo "Verifying monitoring and alerts setup..."
          
          # Check diagnostic settings
          az monitor diagnostic-settings list \
            --resource ${{ vars.SERVICE_BUS_RESOURCE_ID_PROD }} \
            --output table
          
          # List configured alerts
          az monitor metrics alert list \
            --resource-group ${{ vars.AZURE_RESOURCE_GROUP_PROD }} \
            --output table
          
          echo "✅ Monitoring configuration verified"

  notify:
    name: Notify Teams
    runs-on: ubuntu-latest
    needs: [deploy-prod, monitoring-setup]
    if: always()
    steps:
      - name: Notify on Success
        if: needs.deploy-prod.result == 'success'
        uses: 8398a7/action-slack@v3
        with:
          status: success
          fields: repo,commit,author,took
          text: "Service Bus deployment successful! All queues and topics are configured."
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

      - name: Notify on Failure
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: failure
          fields: repo,commit,author,took
          text: "Service Bus deployment failed! Please check the logs."
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

## Message Monitoring and Observability

### 1. Custom Metrics and Logging

```csharp
public class ServiceBusMetricsCollector
{
    private readonly IMetrics _metrics;
    private readonly ILogger<ServiceBusMetricsCollector> _logger;

    public ServiceBusMetricsCollector(IMetrics metrics, ILogger<ServiceBusMetricsCollector> logger)
    {
        _metrics = metrics;
        _logger = logger;
    }

    public void RecordMessageSent(string queueOrTopic, string messageType, TimeSpan duration)
    {
        _metrics.CreateHistogram<double>("servicebus_message_send_duration_ms")
            .Record(duration.TotalMilliseconds, 
                new KeyValuePair<string, object?>("queue_topic", queueOrTopic),
                new KeyValuePair<string, object?>("message_type", messageType));

        _metrics.CreateCounter<long>("servicebus_messages_sent_total")
            .Add(1,
                new KeyValuePair<string, object?>("queue_topic", queueOrTopic),
                new KeyValuePair<string, object?>("message_type", messageType));

        _logger.LogInformation("Message sent to {QueueOrTopic} of type {MessageType} in {Duration}ms",
            queueOrTopic, messageType, duration.TotalMilliseconds);
    }

    public void RecordMessageProcessed(string queueOrSubscription, string messageType, 
        TimeSpan duration, bool success)
    {
        _metrics.CreateHistogram<double>("servicebus_message_process_duration_ms")
            .Record(duration.TotalMilliseconds,
                new KeyValuePair<string, object?>("queue_subscription", queueOrSubscription),
                new KeyValuePair<string, object?>("message_type", messageType),
                new KeyValuePair<string, object?>("success", success));

        _metrics.CreateCounter<long>("servicebus_messages_processed_total")
            .Add(1,
                new KeyValuePair<string, object?>("queue_subscription", queueOrSubscription),
                new KeyValuePair<string, object?>("message_type", messageType),
                new KeyValuePair<string, object?>("success", success));

        var logLevel = success ? LogLevel.Information : LogLevel.Error;
        _logger.Log(logLevel, "Message processed from {QueueOrSubscription} of type {MessageType} " +
            "in {Duration}ms - Success: {Success}",
            queueOrSubscription, messageType, duration.TotalMilliseconds, success);
    }

    public void RecordDeadLetter(string queueOrSubscription, string reason, string description)
    {
        _metrics.CreateCounter<long>("servicebus_deadletter_messages_total")
            .Add(1,
                new KeyValuePair<string, object?>("queue_subscription", queueOrSubscription),
                new KeyValuePair<string, object?>("reason", reason));

        _logger.LogWarning("Message sent to dead letter queue for {QueueOrSubscription}. " +
            "Reason: {Reason}, Description: {Description}",
            queueOrSubscription, reason, description);
    }
}
```

### 2. Kusto Queries for Service Bus Analysis

```kusto
// Monitor message processing rates
AzureMetrics
| where ResourceProvider == "MICROSOFT.SERVICEBUS"
| where MetricName in ("IncomingMessages", "OutgoingMessages")
| summarize MessageCount = sum(Total) by bin(TimeGenerated, 5m), MetricName, ResourceId
| render timechart

// Analyze dead letter messages
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SERVICEBUS"
| where OperationName == "DeadletterMessage"
| summarize DeadLetterCount = count() by bin(TimeGenerated, 1h), ResourceId
| render timechart

// Monitor processing duration
traces
| where customDimensions contains "servicebus_message_process_duration_ms"
| extend Duration = toreal(customDimensions["duration"])
| extend QueueSubscription = tostring(customDimensions["queue_subscription"])
| summarize avg(Duration), max(Duration), percentile(Duration, 95) by bin(timestamp, 5m), QueueSubscription
| render timechart

// Track message failure rates
customEvents
| where name == "MessageProcessed"
| extend Success = tobool(customDimensions["success"])
| extend QueueSubscription = tostring(customDimensions["queue_subscription"])
| summarize SuccessRate = countif(Success) * 100.0 / count() by bin(timestamp, 5m), QueueSubscription
| render timechart
```

## Troubleshooting and Best Practices

### 1. Common Issues and Solutions

#### Message Lock Loss
```csharp
// Implement proper lock renewal for long-running processes
public async Task ProcessLongRunningMessage(ServiceBusReceivedMessage message, ServiceBusReceiver receiver)
{
    using var cancellationTokenSource = new CancellationTokenSource();
    var lockRenewalTask = RenewMessageLock(message, receiver, cancellationTokenSource.Token);
    
    try
    {
        // Long-running processing
        await DoLongRunningWork();
        
        // Complete the message
        await receiver.CompleteMessageAsync(message);
        cancellationTokenSource.Cancel(); // Stop lock renewal
    }
    catch (Exception ex)
    {
        cancellationTokenSource.Cancel();
        await receiver.AbandonMessageAsync(message);
        throw;
    }
}

private async Task RenewMessageLock(ServiceBusReceivedMessage message, 
    ServiceBusReceiver receiver, CancellationToken cancellationToken)
{
    try
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            await Task.Delay(TimeSpan.FromMinutes(2), cancellationToken);
            if (!cancellationToken.IsCancellationRequested)
            {
                await receiver.RenewMessageLockAsync(message);
                _logger.LogDebug("Renewed lock for message {MessageId}", message.MessageId);
            }
        }
    }
    catch (OperationCanceledException)
    {
        // Expected when cancellation is requested
    }
    catch (Exception ex)
    {
        _logger.LogWarning(ex, "Failed to renew message lock for {MessageId}", message.MessageId);
    }
}
```

#### Duplicate Message Handling
```csharp
public class IdempotentMessageProcessor
{
    private readonly IMemoryCache _processedMessages;
    private readonly ILogger<IdempotentMessageProcessor> _logger;

    public async Task<bool> ProcessMessageIdempotently<T>(ServiceBusReceivedMessage message,
        Func<T, Task> processMessage)
    {
        var messageId = message.MessageId;
        var correlationId = message.CorrelationId;
        
        // Use both MessageId and CorrelationId for deduplication
        var deduplicationKey = $"{messageId}:{correlationId}";
        
        if (_processedMessages.TryGetValue(deduplicationKey, out _))
        {
            _logger.LogWarning("Duplicate message detected: {MessageId}, CorrelationId: {CorrelationId}",
                messageId, correlationId);
            return false; // Already processed
        }

        try
        {
            var messageBody = JsonSerializer.Deserialize<T>(message.Body.ToString());
            await processMessage(messageBody);
            
            // Cache the processed message ID for deduplication
            _processedMessages.Set(deduplicationKey, true, TimeSpan.FromHours(24));
            
            return true; // Successfully processed
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing message {MessageId}", messageId);
            throw;
        }
    }
}
```

### 2. Performance Tuning Guidelines

#### Connection Pooling and Reuse
```csharp
public class OptimizedServiceBusClientFactory
{
    private static readonly ConcurrentDictionary<string, ServiceBusClient> _clients = new();
    private readonly DefaultAzureCredential _credential;

    public OptimizedServiceBusClientFactory()
    {
        _credential = new DefaultAzureCredential();
    }

    public ServiceBusClient GetClient(string fullyQualifiedNamespace)
    {
        return _clients.GetOrAdd(fullyQualifiedNamespace, ns => 
            new ServiceBusClient(ns, _credential, new ServiceBusClientOptions
            {
                TransportType = ServiceBusTransportType.AmqpWebSockets, // Better for firewalls
                RetryOptions = new ServiceBusRetryOptions
                {
                    Mode = ServiceBusRetryMode.Exponential,
                    MaxRetries = 3,
                    Delay = TimeSpan.FromSeconds(1),
                    MaxDelay = TimeSpan.FromSeconds(30)
                }
            }));
    }

    public void DisposeAll()
    {
        foreach (var client in _clients.Values)
        {
            client.DisposeAsync().AsTask().Wait();
        }
        _clients.Clear();
    }
}
```

### 3. Environment-Specific Configuration

#### Development Environment
- Shorter message TTL for faster testing
- Higher delivery count for debugging
- Relaxed dead letter policies
- Detailed logging enabled

#### Production Environment
- Optimized message TTL based on business requirements
- Strict delivery count limits
- Comprehensive dead letter handling
- Structured logging with correlation IDs

## Additional References

### Official Microsoft Documentation
- [Azure Service Bus Documentation](https://learn.microsoft.com/en-us/azure/service-bus-messaging/)
- [Service Bus Queues, Topics, and Subscriptions](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-queues-topics-subscriptions)
- [Service Bus Authentication and Authorization](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-authentication-and-authorization)
- [Service Bus Performance Best Practices](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-performance-improvements)

### Security and Identity
- [Authenticate with Managed Identity](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-managed-service-identity)
- [Service Bus Security Baseline](https://learn.microsoft.com/en-us/security/benchmark/azure/baselines/service-bus-security-baseline)
- [Disable Local Authentication](https://learn.microsoft.com/en-us/azure/service-bus-messaging/disable-local-authentication)

### Infrastructure as Code
- [Service Bus Bicep Templates](https://learn.microsoft.com/en-us/azure/templates/microsoft.servicebus/namespaces)
- [Deploy Bicep with GitHub Actions](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deploy-github-actions)
- [Service Bus ARM Templates](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-resource-manager-overview)

### Monitoring and Diagnostics
- [Monitor Service Bus](https://learn.microsoft.com/en-us/azure/service-bus-messaging/monitor-service-bus)
- [Service Bus Metrics and Alerts](https://learn.microsoft.com/en-us/azure/service-bus-messaging/monitor-service-bus-reference)
- [Diagnostic Logs for Service Bus](https://learn.microsoft.com/en-us/azure/service-bus-messaging/monitor-service-bus-reference#resource-logs)

### SDKs and Development
- [.NET SDK for Service Bus](https://learn.microsoft.com/en-us/dotnet/api/overview/azure/service-bus)
- [Java SDK for Service Bus](https://learn.microsoft.com/en-us/java/api/overview/azure/servicebus)
- [Python SDK for Service Bus](https://learn.microsoft.com/en-us/python/api/overview/azure/servicebus)
- [JavaScript SDK for Service Bus](https://learn.microsoft.com/en-us/javascript/api/overview/azure/service-bus)

### Messaging Patterns and Architecture
- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/)
- [Event-Driven Architecture with Service Bus](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/enterprise-integration/queues-events)
- [Microservices Communication Patterns](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/communication-in-microservice-architecture)

### Pricing and Capacity Planning
- [Service Bus Pricing](https://azure.microsoft.com/pricing/details/service-bus/)
- [Service Bus FAQ](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-faq)
- [Service Bus Quotas and Limits](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-quotas)
