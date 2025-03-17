# AWS INFRASTRUCTURE SPECIFICATION
## For Oceanic Wave Dynamics Simulation System

### 1. OVERVIEW & ARCHITECTURE

#### 1.1 Multi-Region Architecture
```
                    ┌─────────────┐          ┌─────────────┐
                    │ US-WEST-2   │          │ US-EAST-1   │
                    │ (PRIMARY)   │◄────────►│ (DISASTER   │
                    └─────────────┘          │  RECOVERY)  │
                          ▲                  └─────────────┘
                          │                        ▲
                          │                        │
                          ▼                        │
┌─────────────────────────────────────────────────┼─────────────┐
│                     GLOBAL EDGE                  │             │
│  ┌─────────────┐   ┌─────────────┐   ┌──────────┴──────────┐  │
│  │ CloudFront  │◄─►│ Route 53    │◄─►│ Global Accelerator  │  │
│  └─────────────┘   └─────────────┘   └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┬─┘
                                                              │
┌─────────────────────────────────────────────────────────────▼─┐
│                        COMPUTE LAYER                           │
│                                                                │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐    │
│  │ EC2 Instances │   │ AWS Batch     │   │ Lambda        │    │
│  │ - Simulation  │   │ - Job Queue   │   │ - Data        │    │
│  │ - Rendering   │   │ - Workers     │   │   Collection  │    │
│  └───────────────┘   └───────────────┘   └───────────────┘    │
└────────────────────────────────┬───────────────────────────────┘
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────┐
│                        STORAGE LAYER                            │
│                                                                 │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐     │
│  │ S3            │   │ EFS           │   │ DynamoDB      │     │
│  │ - Raw Data    │   │ - Simulation  │   │ - Metadata    │     │
│  │ - Results     │   │   Working Dir │   │ - User Data   │     │
│  └───────────────┘   └───────────────┘   └───────────────┘     │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────┐
│                     APPLICATION LAYER                           │
│                                                                 │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐     │
│  │ API Gateway   │   │ AppSync       │   │ ElastiCache   │     │
│  │ - REST API    │   │ - GraphQL     │   │ - Results     │     │
│  │ - Webhooks    │   │ - Realtime    │   │   Caching     │     │
│  └───────────────┘   └───────────────┘   └───────────────┘     │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────┐
│                     MANAGEMENT LAYER                            │
│                                                                 │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐     │
│  │ CloudWatch    │   │ Systems       │   │ Auto Scaling  │     │
│  │ - Monitoring  │   │   Manager     │   │ - Compute     │     │
│  │ - Logging     │   │ - Automation  │   │   Resources   │     │
│  └───────────────┘   └───────────────┘   └───────────────┘     │
└────────────────────────────────────────────────────────────────┘
```

#### 1.2 AWS Services Summary

| Layer            | Services                           | Purpose                                            |
|------------------|------------------------------------|----------------------------------------------------|
| User Interface   | CloudFront, Route 53              | Content delivery, DNS routing                      |
| Compute          | EC2, Batch, Lambda, Fargate       | Simulations, rendering, data processing            |
| Storage          | S3, EFS, DynamoDB, RDS            | Data persistence, metadata, user information       |
| Application      | API Gateway, AppSync, ElastiCache  | APIs, real-time updates, result caching           |
| Management       | CloudWatch, Auto Scaling, SSM     | Monitoring, scaling, automation                    |
| Security         | IAM, WAF, Shield, KMS             | Authentication, protection, encryption             |

### 2. COMPUTE RESOURCES

#### 2.1 EC2 Instance Selection

| Workload               | Instance Type  | vCPUs | Memory  | GPUs          | Storage           | Purpose                                |
|------------------------|----------------|-------|---------|---------------|-------------------|----------------------------------------|
| Wave Simulation        | g5.12xlarge    | 48    | 192 GB  | 4× A10G (24GB)| EBS, EFS          | Physics computation for wave simulation |
| Unreal Rendering       | g5.16xlarge    | 64    | 256 GB  | 1× A10G (24GB)| EBS               | Real-time visualization rendering       |
| API/Web Servers        | c6i.2xlarge    | 8     | 16 GB   | -             | EBS               | Web services, APIs, application servers |
| Database               | r6i.2xlarge    | 8     | 64 GB   | -             | EBS (io2)         | Primary database for user data          |
| Cache/Redis            | r6g.xlarge     | 4     | 32 GB   | -             | EBS               | In-memory cache for fast data access    |

#### 2.2 Batch Processing Configuration

```json
{
  "computeEnvironments": [
    {
      "computeEnvironmentName": "SimulationSpotFleet",
      "type": "MANAGED",
      "state": "ENABLED",
      "computeResources": {
        "type": "SPOT",
        "allocationStrategy": "SPOT_CAPACITY_OPTIMIZED",
        "minvCpus": 0,
        "maxvCpus": 1000,
        "desiredvCpus": 0,
        "instanceTypes": ["g5.4xlarge", "g5.8xlarge", "g5.12xlarge"],
        "subnets": ["subnet-12345", "subnet-67890"],
        "securityGroupIds": ["sg-12345"],
        "instanceRole": "ecsInstanceRole",
        "bidPercentage": 80,
        "spotIamFleetRole": "spotFleetRole"
      }
    },
    {
      "computeEnvironmentName": "RenderingOnDemandFleet",
      "type": "MANAGED",
      "state": "ENABLED",
      "computeResources": {
        "type": "EC2",
        "allocationStrategy": "BEST_FIT_PROGRESSIVE",
        "minvCpus": 0,
        "maxvCpus": 256,
        "desiredvCpus": 0,
        "instanceTypes": ["g5.4xlarge", "g5.8xlarge"],
        "subnets": ["subnet-12345", "subnet-67890"],
        "securityGroupIds": ["sg-12345"],
        "instanceRole": "ecsInstanceRole"
      }
    }
  ],
  "jobQueues": [
    {
      "jobQueueName": "SimulationQueue",
      "state": "ENABLED",
      "priority": 100,
      "computeEnvironmentOrder": [
        {
          "order": 1,
          "computeEnvironment": "SimulationSpotFleet"
        }
      ]
    },
    {
      "jobQueueName": "RenderingQueue",
      "state": "ENABLED",
      "priority": 100,
      "computeEnvironmentOrder": [
        {
          "order": 1,
          "computeEnvironment": "RenderingOnDemandFleet"
        }
      ]
    }
  ]
}
```

### 3. DATA STORAGE ARCHITECTURE

#### 3.1 S3 Bucket Organization

```
aws-wave-simulation/
├── raw-data/
│   ├── buoys/
│   ├── satellite/
│   ├── bathymetry/
│   └── weather/
├── simulation-input/
│   ├── YYYY-MM-DD/
│   │   └── [location-id]/
├── simulation-results/
│   ├── YYYY-MM-DD/
│   │   └── [batch-id]/
│   │       └── [location-id]/
├── visualization-assets/
│   ├── YYYY-MM-DD/
│   │   └── [location-id]/
└── public-content/
    ├── thumbnails/
    ├── videos/
    └── web-assets/
```

#### 3.2 Storage Lifecycle Policy

| Storage Class            | Data Type                           | Transition Timeline                          | Cost Optimization                        |
|--------------------------|-------------------------------------|----------------------------------------------|------------------------------------------|
| S3 Standard              | Active simulation results (<7 days) | -                                            | Short-term, active storage               |
| S3 Intelligent Tiering   | Recent results (7-30 days)          | From Standard after 7 days                   | Automatic optimization based on access   |
| S3 One Zone-IA           | Recent visualization assets         | From Standard after 7 days                   | Less critical, frequently accessed data  |
| S3 Glacier Instant       | Archived results (30-90 days)       | From Intelligent Tiering after 30 days       | Occasional access needed                 |
| S3 Glacier Deep Archive  | Historical data (>90 days)          | From Glacier Instant after 90 days           | Rare access, long-term retention         |

#### 3.3 Database Schema (DynamoDB)

**Simulations Table:**
```json
{
  "TableName": "Simulations",
  "KeySchema": [
    { "AttributeName": "simulation_id", "KeyType": "HASH" }
  ],
  "AttributeDefinitions": [
    { "AttributeName": "simulation_id", "AttributeType": "S" },
    { "AttributeName": "location_id", "AttributeType": "S" },
    { "AttributeName": "date", "AttributeType": "S" }
  ],
  "GlobalSecondaryIndexes": [
    {
      "IndexName": "LocationDate-Index",
      "KeySchema": [
        { "AttributeName": "location_id", "KeyType": "HASH" },
        { "AttributeName": "date", "KeyType": "RANGE" }
      ],
      "Projection": { "ProjectionType": "ALL" }
    }
  ],
  "BillingMode": "PAY_PER_REQUEST"
}
```

### 4. SECURITY ARCHITECTURE

#### 4.1 IAM Role Structure

```
├── SimulationEnvironment
│   ├── EC2SimulationInstanceRole
│   ├── BatchJobExecutionRole
│   └── LambdaDataProcessingRole
├── DataManagement
│   ├── S3ReadWriteRole
│   ├── DynamoDBAccessRole
│   └── KMSEncryptionRole
├── APIServices
│   ├── APIGatewayExecutionRole
│   ├── LambdaAPIHandlerRole
│   └── CloudFrontDistributionRole
└── Monitoring
    ├── CloudWatchLoggingRole
    ├── CloudTrailAuditRole
    └── SNSNotificationRole
```

#### 4.2 Network Security

```
┌───────────────────────────────────────────────────────────────┐
│                      VPC (10.0.0.0/16)                        │
│                                                               │
│  ┌─────────────────┐      ┌─────────────────┐                 │
│  │   Public Subnet │      │  Public Subnet  │                 │
│  │  (10.0.1.0/24)  │      │  (10.0.2.0/24)  │                 │
│  │                 │      │                 │                 │
│  │  ┌───────────┐  │      │  ┌───────────┐  │                 │
│  │  │   NAT     │  │      │  │   NAT     │  │                 │
│  │  │  Gateway  │  │      │  │  Gateway  │  │                 │
│  │  └─────┬─────┘  │      │  └─────┬─────┘  │                 │
│  └────────┼────────┘      └────────┼────────┘                 │
│           │                        │                          │
│  ┌────────┼────────┐      ┌────────┼────────┐                 │
│  │ Private Subnet  │      │ Private Subnet  │                 │
│  │ (10.0.3.0/24)   │      │ (10.0.4.0/24)   │                 │
│  │                 │      │                 │                 │
│  │ ┌─────────────┐ │      │ ┌─────────────┐ │ ┌─────────────┐ │
│  │ │ Simulation  │ │      │ │ Rendering   │ │ │ Application │ │
│  │ │ Instances   │ │      │ │ Instances   │ │ │ Servers     │ │
│  │ └─────────────┘ │      │ └─────────────┘ │ └─────────────┘ │
│  └─────────────────┘      └─────────────────┘                 │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### 5. COST OPTIMIZATION

#### 5.1 EC2 Spot Strategy

| Instance Type | On-Demand Hourly | Spot Market Average | Savings | Interruption Strategy                      |
|---------------|------------------|---------------------|---------|-------------------------------------------|
| g5.12xlarge   | $4.08            | $1.22 - $1.63       | 60-70%  | Checkpointing every 5 minutes             |
| g5.8xlarge    | $2.72            | $0.82 - $1.09       | 60-70%  | Job segmentation, auto-recovery           |
| g5.4xlarge    | $1.36            | $0.41 - $0.54       | 60-70%  | Smaller workloads, faster completion      |

#### 5.2 Auto Scaling Configuration

```json
{
  "AutoScalingGroups": [
    {
      "AutoScalingGroupName": "SimulationASG",
      "MinSize": 0,
      "MaxSize": 20,
      "DesiredCapacity": 0,
      "LaunchTemplate": {
        "LaunchTemplateId": "lt-0123456789abcdef0",
        "Version": "$Latest"
      },
      "VPCZoneIdentifier": "subnet-12345,subnet-67890",
      "TargetGroupARNs": ["arn:aws:elasticloadbalancing:region:account-id:targetgroup/simulation-tg/1234567890"],
      "HealthCheckType": "EC2",
      "HealthCheckGracePeriod": 300,
      "Tags": [
        {
          "Key": "Name",
          "Value": "SimulationNode",
          "PropagateAtLaunch": true
        }
      ]
    }
  ],
  "ScalingPolicies": [
    {
      "AutoScalingGroupName": "SimulationASG",
      "PolicyName": "QueueBasedScaling",
      "PolicyType": "TargetTrackingScaling",
      "TargetTrackingConfiguration": {
        "CustomizedMetricSpecification": {
          "MetricName": "QueueDepth",
          "Namespace": "AWS/SQS",
          "Dimensions": [
            {
              "Name": "QueueName",
              "Value": "SimulationJobQueue"
            }
          ],
          "Statistic": "Average"
        },
        "TargetValue": 10.0
      }
    }
  ]
}
```

### 6. PERFORMANCE OPTIMIZATION

#### 6.1 Placement Groups

```json
{
  "PlacementGroups": [
    {
      "GroupName": "SimulationCluster",
      "Strategy": "cluster"
    },
    {
      "GroupName": "RenderingCluster",
      "Strategy": "cluster"
    }
  ]
}
```

#### 6.2 Storage Performance Configuration

| Storage Type     | Configuration                      | Performance                             | Workload                             |
|------------------|------------------------------------|-----------------------------------------|--------------------------------------|
| EBS gp3          | 10,000 IOPS, 1000 MB/s throughput  | High throughput, moderate IOPS          | OS volumes, general storage          |
| EBS io2          | 50,000 IOPS, 1000 MB/s throughput  | Very high IOPS, consistent performance  | Database, transaction logs           |
| EFS Performance  | Max I/O, Provisioned 250 MiB/s     | High throughput, shared access          | Simulation working directories       |
| FSx for Lustre   | 1000 MB/s                          | Very high throughput, parallel access   | High performance simulation I/O      |
| ElastiCache      | r6g.xlarge, Multi-AZ               | Sub-millisecond latency                 | Real-time data, visualization cache  |

### 7. DISASTER RECOVERY

#### 7.1 RPO/RTO Targets

| Data Category              | RPO (Recovery Point Objective) | RTO (Recovery Time Objective) | Strategy                                     |
|----------------------------|--------------------------------|-------------------------------|----------------------------------------------|
| User accounts/credentials  | ~5 minutes                     | <1 hour                       | Multi-region replication                     |
| Simulation metadata        | ~15 minutes                    | <2 hours                      | DynamoDB global tables                       |
| Active simulation results  | ~1 hour                        | <4 hours                      | S3 cross-region replication                  |
| Historical simulation data | ~24 hours                      | <24 hours                     | S3 cross-region replication (as needed)      |
| Infrastructure config      | Real-time                      | <8 hours                      | Infrastructure as Code in version control    |

#### 7.2 Backup Strategy

```json
{
  "BackupPlan": {
    "BackupPlanName": "SimulationDataBackup",
    "Rules": [
      {
        "RuleName": "DailyBackups",
        "TargetBackupVault": "SimulationBackupVault",
        "ScheduleExpression": "cron(0 0 * * ? *)",
        "StartWindowMinutes": 60,
        "CompletionWindowMinutes": 180,
        "Lifecycle": {
          "DeleteAfterDays": 35
        },
        "RecoveryPointTags": {
          "Type": "Daily"
        }
      },
      {
        "RuleName": "WeeklyBackups",
        "TargetBackupVault": "SimulationBackupVault",
        "ScheduleExpression": "cron(0 0 ? * SUN *)",
        "StartWindowMinutes": 60,
        "CompletionWindowMinutes": 180,
        "Lifecycle": {
          "DeleteAfterDays": 90
        },
        "RecoveryPointTags": {
          "Type": "Weekly"
        }
      }
    ]
  }
}
```

### 8. MONITORING & ALERTING

#### 8.1 CloudWatch Dashboard

**Key Metrics:**
- Simulation queue depth
- Instance resource utilization (CPU, GPU, Memory)
- Batch job success/failure rates
- S3 request rates and latency
- API Gateway request volume and latency
- DynamoDB consumed capacity
- Cost tracking by resource group

#### 8.2 Alert Configuration

| Alert                         | Threshold                        | Action                                    |
|-------------------------------|----------------------------------|-------------------------------------------|
| Simulation Queue High         | >50 jobs for >30 minutes         | Auto-scale up simulation instances        |
| Job Failure Rate High         | >10% failure rate in 1 hour      | Notify SRE team, pause new submissions   |
| API Latency High              | >500ms p95 for >5 minutes        | Scale up API resources, notify SRE team   |
| Storage Throughput Exceeding  | >80% of provisioned for >15 min  | Scale up storage, notify SRE team         |
| Spot Instance Termination     | Any imminent termination         | Checkpoint jobs, redirect to on-demand    |
| Cost Anomaly                  | >30% deviation from forecast     | Notify finance team, pause non-critical   |

### 9. DEPLOYMENT & CI/CD

```
┌───────────────┐   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  Source Code  │──>│  AWS CodeBuild │──>│ AWS CodeDeploy│──>│  Production   │
│  Repository   │   │                │   │               │   │  Environment  │
└───────────────┘   └───────────────┘   └───────────────┘   └───────────────┘
       │                    │                   │                   │
       │                    │                   │                   │
       ▼                    ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ Infrastructure│   │  Container    │   │ Configuration  │   │  Monitoring   │
│     Code      │   │  Registry     │   │   Store        │   │  & Alerting   │
└───────────────┘   └───────────────┘   └───────────────┘   └───────────────┘
```

### 10. COST ESTIMATES

#### 10.1 Monthly Cost Breakdown (500 Surf Spots, 1000 Users)

| Resource Category           | Service                 | Monthly Cost    | Notes                                  |
|-----------------------------|-------------------------|-----------------|----------------------------------------|
| Compute - Simulation        | EC2 (g5 Spot)           | $1,600 - $2,200 | 4-6 hours daily of g5.12xlarge        |
| Compute - Rendering         | EC2 (g5 On-Demand)      | $900 - $1,300   | 2-3 hours daily of g5.8xlarge         |
| Compute - Web/API           | EC2 (c6i)               | $200 - $300     | 2-4 instances running continuously     |
| Storage - S3                | S3 Standard + IA        | $400 - $600     | ~10TB active storage + lifecycle       |
| Storage - Database          | DynamoDB + RDS          | $200 - $300     | On-demand capacity + small RDS         |
| Storage - Cache             | ElastiCache             | $150 - $200     | r6g.xlarge in Multi-AZ                 |
| Networking                  | Data Transfer           | $300 - $500     | ~5-10TB monthly transfer               |
| Networking                  | CloudFront              | $150 - $250     | Distribution to global users           |
| Management                  | CloudWatch, etc.        | $100 - $150     | Logging, monitoring, SNS               |
| **TOTAL**                   |                         | **$4,000 - $5,800** | **$4-5.80 per user monthly**        |

#### 10.2 Scaling Projections

| User Count | Surf Spots | Monthly Cost   | Cost Per User  | Economies of Scale            |
|------------|------------|----------------|----------------|-------------------------------|
| 1,000      | 500        | $4,000-$5,800  | $4.00-$5.80    | Baseline                      |
| 5,000      | 750        | $7,500-$10,000 | $1.50-$2.00    | ~65% reduction per user       |
| 10,000     | 1,000      | $12,000-$16,000| $1.20-$1.60    | ~75% reduction per user       |
| 50,000     | 1,500      | $30,000-$40,000| $0.60-$0.80    | ~85% reduction per user       |

### 11. IMPLEMENTATION ROADMAP

| Phase | Timeline | Focus Areas                                         | Key Milestones                                 |
|-------|----------|-----------------------------------------------------|-----------------------------------------------|
| 1     | Weeks 1-4| Infrastructure foundation, core storage, VPC design | Base VPC, IAM, initial EC2 instances           |
| 2     | Weeks 5-8| Simulation infrastructure, batch processing         | Functional simulation pipeline in staging      |
| 3     | Weeks 9-12| API layer, database, user-facing services         | Complete API endpoints, initial user access    |
| 4     | Weeks 13-16| Optimization, security hardening, monitoring     | Production-ready environment with monitoring   |
| 5     | Weeks 17-20| Scaling, disaster recovery, multi-region         | DR testing, load testing completed            |

---

## APPENDICES

### Appendix A: AWS Service List and Purpose

| Service               | Purpose in OWDSS Architecture                               |
|-----------------------|-------------------------------------------------------------|
| EC2                   | Compute instances for simulation and rendering              |
| S3                    | Object storage for simulation data and results              |
| DynamoDB              | NoSQL database for metadata and user information            |
| Lambda                | Serverless functions for data collection and processing     |
| Batch                 | Job scheduling and management for simulations               |
| EFS                   | Shared file system for simulation data                      |
| CloudFront            | CDN for delivery of visualization assets                    |
| API Gateway           | REST API interface for applications                         |
| AppSync               | GraphQL API for real-time data exchange                     |
| ElastiCache           | In-memory caching for performance                           |
| CloudWatch            | Monitoring and alerting                                     |
| Auto Scaling          | Dynamic resource scaling based on demand                    |
| IAM                   | Identity and access management                              |
| KMS                   | Encryption key management                                   |
| Route 53              | DNS management and global routing                           |
| VPC                   | Network isolation and security                              |
| Systems Manager       | Configuration and automation                                |
| CodeBuild/CodeDeploy  | CI/CD pipeline for application deployment                   |

### Appendix B: Security Controls Checklist

- [ ] VPC Flow Logs enabled for network monitoring
- [ ] GuardDuty enabled for threat detection
- [ ] All S3 buckets encrypted with KMS CMKs
- [ ] WAF configured with rate limiting and geo-restrictions
- [ ] CloudTrail enabled for API auditing
- [ ] Config enabled for resource compliance monitoring
- [ ] Security groups limited to required ports only
- [ ] IAM roles follow principle of least privilege
- [ ] Secrets Manager used for credential management
- [ ] All data in transit encrypted (TLS 1.2+)
- [ ] All EBS volumes encrypted
- [ ] Multi-factor authentication for console access
- [ ] Regular vulnerability assessments

*Document Version: 1.0*
*Last Updated: [Current Date]* 