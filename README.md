# Snapshot Tool for Amazon Aurora Postgres

The Snapshot Tool for Amazon Aurora automates the task of creating manual snapshots, copying them into a different region, and deleting them after a specified number of days. It also allows to specify the backup schedule (at what times and how often) and a retention period in days. 

## Solution Architecture

### Source Region 
![Image of Source Region architecture](source_image.png)

### Destination Region 
![Image of Destination Region architecture](destination_image.png)

## Getting Started

Inorder to deploy the solution in EDP account, you will need to use the Cloudformation templates provided for source and destination region. Solution Architecture is optimized to reuse most of the components. For the first time setup, IAM roles and lambda functions needs to be deployed. Subsequent implementations require cluster specific deployments i.e. cloudwatch event & alarm, step fucntions and SNS. 

**IMPORTANT** Run the Cloudformation templates inn the same region where your Aurora clusters run (both in the source and destination regions). 

### Source Region
#### Components

The following components will be created in the source account: 

##### Initial Setup(TO BE EXECUTED ONLY FIRST TIME IN SOURCE ACCOUNT):

IAM Roles
* 3 IAM Roles to execute Lambda functions, Step Function invocation and Execution
* A Cloudformation stack containing these resources

Lambda Functions
* 4 Lambda functions (TakeSnapshotsAurora, ShareSnapshotsAurora, DeleteOldSnapshotsAurora, CheckSnapshotStatus) 

***NOTE*** Create an S3 bucket to hold the Lambda function zip files. The bucket must be in the same region where the Lambda functions will run. And the Lambda functions must run in the same region as the RDS instances.

* A Cloudformation stack containing these these resources

##### Cluster specific Setup:

* 1 State Machine (Amazon Step Functions) to trigger execution of orchesterated Lambda functions   (stateMachineTakeSnapshotAurora)
* 1 Cloudwatch Event Rules to trigger the state functions
* 1 Cloudwatch Alarms and 1 associated SNS Topic to alert on State Machines failures
* A Cloudformation stack containing these resources

#### Installing in the source account 

1. Run Following command on Stacker to setup IAM Roles (FIRST TIME ONLY):
2. Run Following command on Stacker to setup LAMBDA FUNCTIONS (FIRST TIME ONLY):
3. Run Following command on Stacker to setup Cluster Specific DR resources:

You will need to specify the different parameters in env file to meet the cluster specific backup requirements.

Here is a break down of each parameter for the source template:

* **BackupInterval** - how many hours between backup
* **BackupSchedule** - at what times and how often to run backups. Set in accordance with **BackupInterval**. For example, set **BackupInterval** to 8 hours and **BackupSchedule** 0 0,8,16 * * ? * if you want backups to run at 0, 8 and 16 UTC. If your backups run more often than **BackupInterval**, snapshots will only be created when the latest snapshot is older than **BackupInterval**
* **ClusterNamePattern** - set to the name of the cluster you want this tool to back up. This tool supports backup of one cluster at a time. 
* **DestinationRegion** - the account where you want snapshots to be copied to
* **LogLevel** - The log level you want as output to the Lambda functions. ERROR is usually enough. You can increase to INFO or DEBUG. 
* **RetentionDays** - the amount of days you want your snapshots to be kept. Snapshots created more than **RetentionDays** ago will be automatically deleted (only if they contain a tag with Key: CreatedBy, Value: Snapshot Tool for Aurora)
* **CodeBucket** - this parameter specifies the bucket where the code for the Lambda functions is located.  
* **DeleteOldSnapshots** - Set to TRUE to enable functionanility that will delete snapshots after **RetentionDays**. Set to FALSE if you want to disable this functionality completely.
* **ShareSnapshots** - Set to TRUE to enable functionality that will share snapshots with **DestinationRegion**. Set to FALSE to completely disable sharing. 
* **KmsKeySource** KMS Key to be used for copying encrypted snapshots on the source region. If you are copying to a different region, you will also need to provide a second key in the destination region. 
* **KmsKeyDestination** KMS Key to be used for copying encrypted snapshots to the destination region.

### Destination Region
#### Components

The following components will be created in the destination account: 

##### Initial Setup(TO BE EXECUTED ONLY FIRST TIME IN DESTINATION ACCOUNT):

Lambda Functions
* 1 Lambda functions (DeleteOldSnapshotsAurora)
* A Cloudformation stack containing these these resources

##### Cluster specific Setup:

* 1 State Machine (Amazon Step Functions) to trigger execution of orchesterated Lambda functions   (stateMachineTakeSnapshotAurora)
* 1 Cloudwatch Event Rules to trigger the state functions
* 1 Cloudwatch Alarms and 1 associated SNS Topic to alert on State Machines failures
* A Cloudformation stack containing these resources

#### Installing in the destination account 

1. Run Following command on Stacker to setup LAMBDA FUNCTIONS (FIRST TIME ONLY):
2. Run Following command on Stacker to setup Cluster Specific DR resources:

As before, you will need to run it in a region where Step Functions is available. 
The following parameters are available:

* **SnapshotPattern** - similar to ClusterNamePattern. See above
* **DeleteOldSnapshots** - Set to TRUE to enable functionanility that will delete snapshots after 
* **RetentionDays** - as in the source region, the amount of days you want your snapshots to be kept. **Do not set this parameter to a value lower than the source region.** Snapshots created more than **RetentionDays** ago will be automatically deleted (only if they contain a tag with Key: CopiedBy, Value: Snapshot Tool for Aurora)

