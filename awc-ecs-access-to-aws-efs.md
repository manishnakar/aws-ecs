## Configuring AWS ECS to have access to AWS EFS
If you would like to persist data from your ECS containers, i.e. hosting databases like MySQL or MongoDB with Docker, you need to ensure that you can mount the data directory of the database in the container to volume that's not going to dissappear when your container or worse yet, the EC2 instance that hosts your containers, is restarted or scaled up or down for any reason.

> Don't know how to create your own AWS ECS Cluster? Go [here](https://gist.github.com/duluca/ebcf98923f733a1fdb6682f111b1a832#file-step-by-step-how-to-for-aws-ecs-md)!

## New Cluster
Sadly the EC2 provisioning process doesn't allow you to configure EFS during the initial config. After your create your cluster, follow the guide below.

## New Task Definition for Web App
If you're using an Alpine-based Node server like [duluca/minimal-node-web-server]() follow this guide:


0. Go to Amazon ECS
1. Task Definitions -> Create new Task Definition
2. Name: app-name-task, role: none, network: bridge
3. Add container, name: app-name from before, image: URI from before, but append ":latest"
4. Soft limit, 256 MB for Node.js
5. Port mappings, Container port: 3000
6. Log configuration: awslogs; app-name-logs, region, app-name-prod

## New Task Definition for Database
If you're hosting a lightweight database like [mongo](https://hub.docker.com/_/mongo/) or [excellalabs/mongo](https://hub.docker.com/r/excellalabs/mongo/):


0. Go to Amazon ECS
1. Task Definitions -> Create new Task Definition
2. Name: mongodb-task, role: none, network: bridge
3. Add container, name: mongodb-prod, image: mongo or excellalabs/mongo, append a version number like ":3.4.7"
4. Soft limit, 1024 MB
5. Port mappings, Container port: 27017
6. Log configuration: awslogs; mongodb-prod-logs, region, mongodb-prod
7. Add Env Variables, see [excellalabs/mongo repo](https://github.com/excellalabs/mongo-docker) for details
```
MONGODB_ADMIN_PASS
MONGODB_APPLICATION_DATABASE
MONGODB_APPLICATION_PASS
MONGODB_APPLICATION_USER
```
> It is not a security best practice to store such secrets in an encrypted form. If you'd like to do the right way, here's your homework: https://aws.amazon.com/blogs/compute/managing-secrets-for-amazon-ecs-applications-using-parameter-store-and-iam-roles-for-tasks/
8. Then create a new service based on this task definition.
 8.1. Make sure that under Deployment Options Minimum healthy percent is 0 and Maximum percent 100. You don't _ever_ want two separate Mongo instances mounted to the same data source. 

## Existing ECS Cluster with Existing Task Definition for Container

### Create a new KMS encryption key
If you would like to encrypt your file system at-rest, then you must have a KMS key.
> If not, you may skip but it is **strongly** recommended that you encrypt your data - no matter how unimportant you think your data is at the moment.

3. Headover to IAM -> Encryption Keys
4. Create key
5. Provide Alias and a description
6. Tag with 'Environment': 'production'
7. Carefuly select 'Key Administrators'
8. Uncheck 'Allow key administrators to delete this key.' to prevent accidental deletions 
9. Key Usage Permissions
10. Select the 'Task Role' that was created when configuring your AWS ECS Cluster. If not see the **Create Task Role** section in the guide linked above. You'll need to update existing task definitions, and update your service with the new task definition for the changes to take affect.
11. Finish

### Create a new EFS
1. Launch EFS
2. Create file system
3. Select the VPC that your ECS cluster resides in
4. Select the AZs that your container instances reside in
5. Next
6. Add a name
7. Enable encryption (You WANT this -- see above)
8. Create File System
9. Back on the EFS main page, expand the EFS definition, if not already expanded
10. Copy the **DNS name**

### Update Your Cloud Formation Template
1. CloudFormation
2. Select EC2ContainerService-cluster-name
3. View/edit design template
4. Modify the YML to add `EfsUri` amongst the input parameters
```yml
  EfsUri:
    Type: String
    Description: >
      EFS volume DNS URI you would like to mount your EC2 instances to. Directory -> /mnt/efs
    Default: ''
```
5. Find `EcsInstanceLc` update its `UserData` property to look like:
```yml
UserData: !If
        - SetEndpointToECSAgent
        - Fn::Base64: !Sub |
           #!/bin/bash
           # Install efs-utils
           cloud-init-per once yum_update yum update -y
           cloud-init-per once install_nfs_utils yum install -y amazon-efs-utils
        
           # Create /efs folder
           cloud-init-per once mkdir_efs mkdir /efs
        
           # Mount /efs, ensuring a TLS connection
           cloud-init-per once mount_efs echo -e '${EfsUri}:/ /efs efs tls,_netdev 0 0' >> /etc/fstab
           mount -a
           echo ECS_CLUSTER=${EcsClusterName} >> /etc/ecs/ecs.config
           echo ECS_BACKEND_HOST=${EcsEndpoint} >> /etc/ecs/ecs.config
        - Fn::Base64: !Sub |
           #!/bin/bash
           # Install efs-utils
           cloud-init-per once yum_update yum update -y
           cloud-init-per once install_nfs_utils yum install -y amazon-efs-utils
        
           # Create /efs folder
           cloud-init-per once mkdir_efs mkdir /efs
        
           # Mount /efs, ensuring a TLS connection
           cloud-init-per once mount_efs echo -e '${EfsUri}:/ /efs efs tls,_netdev 0 0' >> /etc/fstab
           mount -a
           echo ECS_CLUSTER=${EcsClusterName} >> /etc/ecs/ecs.config
```
5. Validate the template
6. Save the template to S3 and copy the URL 
7. Select your CloudFormation stack again -> Update stack
8. Paste in the S3 url -> Next 
9. Now you'll see an `EfsUri` parameter, define it using the DNS name copied from the previous part
10. On the review screen make sure it is only updating the Auto Scaling Group (ASG) and the Launch Configuration (LC)
11. Let it update the stack 

### And Now, The Fun Part -- Updating Your ECS Instances
1. ECS -> Cluster
2. Switch to ECS Instances tab
There are two paths forward here, one is the sledgehammer, which will **bring down** your applications:
3. Scale ECS instances to 0 **Note** This is the part where your applications come down
4. After all instances have been brougt down, scale back up to 2 (or more)
Or perform a rolling update, which will **keep alive** your application:
3. Click on the EC2 instance and on the EC2 dashboard, select Actions -> State -> Terminate
4. Wait while the instance is terminated and reprovisioned
5. Rinse and repeat for the next instance

### Update Task Definition to Mount to the EFS Volume
1. ECS -> Task definitions
2. Create new revision
3. If you already have not added it, make sure the Role here matches the one for the KMS key
4. Add volume
5. Name: 'efs', Source Path: '/efs/your-dir' (If this doesn't work try '/mnt/efs/your-dir')
6. Add
7. Click on container name, under Storage and Logs
8. Select mount point 'efs'
9. Provide the internal container path. i.e. for MongoDB default is '/data/db'
10. Update
11. Create

### Update ECS Service with the new Task Definition
1. ECS -> Clusters
2. Click on Service name
3. Update
4. Type in the new task definition name
5. Update service

Your service should re-provision the existing containers and **voila, you're done!**

## Last, But Not Least -- Test
Test what you have done. 

Go ahead and save some data. 

Then scale your EC2 instance size down to 0 (the sledgehammer) and scale it back up again and see if the data is still accessible.
