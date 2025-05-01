
# AWS CloudWatch Metrics for EKS: A Comprehensive Guide

## Introduction

Amazon EKS (Elastic Kubernetes Service) exposes various metrics to CloudWatch that enable you to monitor the health, performance, and operational state of your Kubernetes clusters. This document provides a detailed overview of the metric types available for EKS monitoring through CloudWatch, how they work, and best practices for utilizing them effectively.

## Container Insights Metrics Framework

Container Insights collects, aggregates, and summarizes metrics from your EKS clusters. These metrics are organized into several dimensions and categories to provide a comprehensive view of your cluster's health.

### Metric Sources

1. **Built-in EKS Platform Metrics**: Metrics that AWS collects automatically from the EKS control plane.
2. **Container Insights Metrics**: Metrics collected by the CloudWatch agent running as a DaemonSet in your cluster.
3. **Custom Application Metrics**: Metrics that your applications can emit using the CloudWatch agent or other monitoring tools.

### Metric Dimensions

Metrics in CloudWatch are organized by dimensions that allow you to filter and aggregate data:

- **ClusterName**: The name of your EKS cluster
- **Namespace**: The Kubernetes namespace
- **PodName**: The name of the Kubernetes pod
- **ServiceName**: The name of the Kubernetes service
- **ContainerName**: The name of the container
- **InstanceId**: The EC2 instance ID (for self-managed nodes)
- **NodeName**: The Kubernetes node name

## Core Metric Categories

### Cluster-Level Metrics

| Metric Name                        | Description                           | Unit  | Common Use Cases              |
| ---------------------------------- | ------------------------------------- | ----- | ----------------------------- |
| `cluster_failed_node_count`        | Number of nodes that failed           | Count | Cluster health monitoring     |
| `cluster_node_count`               | Total number of nodes in the cluster  | Count | Capacity planning             |
| `namespace_number_of_running_pods` | Number of pods running in a namespace | Count | Namespace capacity monitoring |

### Node-Level Metrics

|Metric Name|Description|Unit|Common Use Cases|
|---|---|---|---|
|`node_cpu_utilization`|Percentage of CPU used by the node|Percent|Resource utilization monitoring|
|`node_memory_utilization`|Percentage of memory used by the node|Percent|Memory pressure detection|
|`node_network_total_bytes`|Total bytes transmitted and received by the node|Bytes|Network usage analysis|
|`node_filesystem_utilization`|Percentage of filesystem used|Percent|Storage monitoring|
|`node_status_condition_ready`|Node readiness status (1=Ready, 0=Not Ready)|Count|Health monitoring|

### Pod-Level Metrics

|Metric Name|Description|Unit|Common Use Cases|
|---|---|---|---|
|`pod_cpu_utilization`|Percentage of CPU used by the pod|Percent|Resource optimization|
|`pod_memory_utilization`|Percentage of memory used by the pod|Percent|OOM prevention|
|`pod_network_rx_bytes`|Bytes received by the pod|Bytes|Network troubleshooting|
|`pod_network_tx_bytes`|Bytes transmitted by the pod|Bytes|Network troubleshooting|
|`pod_cpu_utilization_over_pod_limit`|Pod CPU utilization as a percentage of its limit|Percent|Resource limit tuning|
|`pod_memory_utilization_over_pod_limit`|Pod memory utilization as a percentage of its limit|Percent|Memory limit tuning|
|`pod_number_of_container_restarts`|Number of container restarts in a pod|Count|**CrashLoopBackOff detection**|
|`pod_status_ready`|Pod readiness status (1=Ready, 0=Not Ready)|Count|Health monitoring|
|`pod_status_phase`|Numeric representation of pod phase|Count|Lifecycle monitoring|

### Container-Level Metrics

|Metric Name|Description|Unit|Common Use Cases|
|---|---|---|---|
|`container_cpu_utilization`|Percentage of CPU used by the container|Percent|Performance monitoring|
|`container_memory_utilization`|Percentage of memory used by the container|Percent|Memory leak detection|
|`container_filesystem_usage`|Filesystem bytes used by the container|Bytes|Storage troubleshooting|
|`container_cpu_limit`|The CPU limit for the container|Millicores|Resource configuration validation|
|`container_memory_limit`|The memory limit for the container|Bytes|Resource configuration validation|
|`container_status`|Container status (Running=1, Otherwise=0)|Count|Health monitoring|

### Service-Level Metrics

|Metric Name|Description|Unit|Common Use Cases|
|---|---|---|---|
|`service_number_of_running_pods`|Number of pods running for a service|Count|Service capacity monitoring|
|`service_pod_network_rx_bytes`|Bytes received by all pods in a service|Bytes|Service traffic analysis|
|`service_pod_network_tx_bytes`|Bytes transmitted by all pods in a service|Bytes|Service traffic analysis|

## Enhanced Observability with Log-Based Metrics

In addition to the built-in metrics, you can create custom metrics based on log patterns to monitor specific events:

### Common Log-Based Metrics for EKS

1. **Pod Status Changes**: Metrics created from log patterns indicating pod state transitions
    
    - Pattern: `"previous state.*\[.*\].*new state.*\[.*\]"`
2. **Container OOM Events**: Metrics for Out of Memory events
    
    - Pattern: `"OOMKilled" OR "Out of memory"`
3. **Scheduling Failures**: Metrics for pod scheduling issues
    
    - Pattern: `"FailedScheduling" OR "cannot schedule"`
4. **CrashLoopBackOff Events**: Metrics for container restart cycles
    
    - Pattern: `"BackOff" "CrashLoopBackOff"`

## Metric Collection Architecture

CloudWatch Container Insights uses the following components to collect metrics:

1. **CloudWatch Agent**: Runs as a DaemonSet on each node to collect metrics
2. **Fluent Bit**: Processes and forwards logs to CloudWatch Logs
3. **AWS SDK**: Sends metrics from the agents to CloudWatch Metrics service

### Data Flow

1. Container/node metrics are collected by the CloudWatch agent
2. Logs are collected by Fluent Bit
3. Both are formatted in the CloudWatch Embedded Metric Format (EMF)
4. Data is sent to CloudWatch in near real-time
5. CloudWatch processes the data and makes it available for querying and alerting

## Metric Math and Advanced Monitoring

CloudWatch allows you to perform calculations on metrics to derive more meaningful information:

### Common Metric Math Expressions for EKS

1. **Average CPU utilization across all pods in a namespace**:
    
    ```
    AVG(SEARCH('Namespace="default" MetricName="pod_cpu_utilization"', 'Average', 300))
    ```
    
2. **Container restart rate per hour**:
    
    ```
    RATE(m1) * 3600
    ```
    
    Where m1 is `pod_number_of_container_restarts`
    
3. **Ratio of memory usage to limit**:
    
    ```
    (m1 / m2) * 100
    ```
    
    Where m1 is `container_memory_usage` and m2 is `container_memory_limit`
    

## Best Practices for Metric Alarms

### Critical EKS Metrics to Monitor

|Metric|Threshold Guidance|Impact|Alarm Priority|
|---|---|---|---|
|`pod_number_of_container_restarts`|>5 in 5 minutes|Indicates application stability issues or CrashLoopBackOff|High|
|`pod_memory_utilization_over_pod_limit`|>85%|Early indicator of potential OOM|Medium|
|`node_cpu_utilization`|>80% sustained|Node performance degradation|Medium|
|`node_filesystem_utilization`|>85%|Risk of disk space issues|Medium|
|`pod_status_ready`|=0 for >5 minutes|Pod is unhealthy|High|
|`cluster_failed_node_count`|>0|Node failure affecting cluster capacity|High|

### Effective Alarm Configuration

1. **Use composite alarms** to reduce noise and correlate related issues
2. **Implement different thresholds** for development and production environments
3. **Configure appropriate evaluation periods** to avoid false alarms due to brief spikes
4. **Set up different notification actions** based on alarm severity
5. **Create dashboards** with visual alarm indicators

## Integration with AWS Services

CloudWatch metrics from EKS can integrate with other AWS services:

- **EventBridge**: Trigger automated remediation based on metric alarms
- **Lambda**: Run functions in response to specific metric conditions
- **AWS Systems Manager**: Execute automation documents for troubleshooting
- **AWS Health Dashboard**: Correlate EKS issues with AWS service health
- **X-Ray**: Link metrics to distributed traces for deeper analysis

## Cost Optimization

CloudWatch metrics have associated costs. Consider these strategies to optimize:

1. **Focus on actionable metrics** rather than collecting everything
2. **Adjust collection frequency** based on metric importance
3. **Set appropriate retention periods** for different metric types
4. **Use log filters carefully** to avoid excessive metric generation
5. **Create metric math expressions** instead of storing computed metrics

## Advanced Use Cases

### Kubernetes Events as Metrics

Convert Kubernetes events to CloudWatch metrics for better observability:

1. Deploy `k8s-event-exporter` to your cluster
2. Configure it to send events to CloudWatch Logs
3. Create metric filters from these logs
4. Set up alarms on specific event patterns

### Custom Application Metrics

Instrument your applications to send custom metrics via:

1. **StatsD with CloudWatch agent**
2. **Prometheus with CloudWatch agent**
3. **Direct integration** with CloudWatch API
4. **Application Signals for EKS**

## Conclusion

Effective monitoring of EKS clusters requires a comprehensive understanding of available metrics, appropriate alarm thresholds, and integration with other monitoring tools. By leveraging CloudWatch's rich set of metrics and features, you can build a robust monitoring solution that provides visibility into the health and performance of your Kubernetes workloads.