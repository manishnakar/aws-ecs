## Setting up Your Own AWS ECS Cluster
This is a multi-step configuration -- easy mistakes are likely. Be patient! The pay-off will be worth it. Rudimentary knowledge and awareness of the AWS landscape is not necessarily required, but will make it easier to set things up.

> Enable fantastic Blue-Green deployments with [_npm scripts for AWS ECS_](https://gist.github.com/duluca/2b67eb6c2c85f3d75be8c183ab15266e#file-npm-scripts-for-aws-ecs-md).

Some of the instructions make references to `package.json` for `npm script for AWS ECS` users. You may safely ignore these steps. 
  
### Creating Amazon ECS Infrastructure

#### Create a new IAM role
> If you plan on having multiple clusters (which is likely to happen at some point) then you should define its own IAM role to prevent any future unintended or malicious access AWS resources.
1. IAM -> Roles
2. Create new role
3. Select Amanzon EC2
4. Select AmazonEC2ContainerServiceforEC2Role policy -> Next
5. prod-ecs-instanceRole

#### Create Cluster
1. Go to Amazon ECS
2. Clusters -> Create Cluster
3. Name: prod-ecs-cluster
4. On-Demand Instance
5. 2 m4.large instances across two AZs for highly available config
6. Create new prod-vpc
7. Create new prod-security-group
8. Allow port 80 and 443 for HTTP and HTTPS inbound
9. Allow port range 32768-61000 so that ECS can dynamically scale instances and run healh checks
10. Container instance IAM role: select 'prod-ecs-instanceRole' that you just created, if not 'ecsIntanceRole'
11. Create

#### Verify Security Group Config
This is a big deal.
1. Go EC2 -> Network & Security -> Security Groups
2. Verify there ports are open:

| Type | Protocol | Port Range | Source |
| --- | --- | --- | --- |
| HTTP (80) | TCP (6) |80 | 0.0.0.0/0 |
| HTTP (80) | TCP (6) |80 | ::/0 |
| Custom TCP Rule | TCP (6) | 32768-61000 | 0.0.0.0/0 |
| HTTPS (443) | TCP (6) | 443| 0.0.0.0/0 |
| HTTPS (443) | TCP (6) | 443| ::/0 |

#### Create Container Repository
1. Go to Amazon ECS
2. Repositories -> Create Repository
3. Enter your app-name
4. Copy repository URI, add to package.json 
“imageRepo”: “000000000000.dkr.ecr.us-east-1.amazonaws.com/app-name"
5. Create

#### Create Task Definition
0. Go to Amazon ECS
1. Task Definitions -> Create new Task Definition
2. Name: app-name-task, role: none, network: bridge
3. Add container, name: app-name from before, image: URI from before, but append ":latest"
4. Soft limit, 256 MB for Node.js
5. Port mappings, Container port: 3000
6. Log configuration: awslogs; app-name-logs, region, app-name-prod

#### Create ELB
1. Go to Amazon EC2
2. Load Balancers -> Create Load Balancer
3. Application Load Balancer
4. Name: app-name-prod-elb
5. Add listener: HTTPS, 443
6. AZs, select prod-vpc, select all
7. Tags -> Domain, app-name.yourdomain.com
8. Next
9. Choose or create SSL cert (star is recommended: add *.yourdomain.com and yourdomain.com separately on the cert)
10. Select default ELB security policy
11. Next
12. Create prod-cluster specific security group only allowing port 80 and 443 inbound
13. Next
14. New target group, name: app-name
15. Health-checks: Keep default "/" if serving a website on HTTP, but if deploying an API and/or redirecting all HTTP calls to HTTPS, ensure your app defines a custom route that is not redirected to HTTPS. On HTTP server GET "/healthCheck" return simple 200 message saying "I'm healthy" -- verify that this does not redirect to HTTPS, otherwise lot's of pain and suffering will occur. Health checks on AWS will fail.
16. DO NOT REGISTER ANY TARGETS: ECS will do this for you, if you do so yourself, you will provision a semi-broken infrastructure
17. Next:Review, then Create
 
#### Create Service
1. Go to Amazon ECS
2. Clusters -> Select "prod-ecs-cluster"
3. Task Definition: app-name-task from before
4. Service name: app-name
5. No of tasks: 2, min healthy: 100, max healthy: 200 for highly available blue/green deployment setup
6. Configure ELB
  * Application Load Balancer
  * ecsServiceRole
  * Select app-name-prod-elb from before
  * Select app-name:0:3000 container from before
  * Add to ELB
  * Target Group Name: app-name from before
  * Save
7. Create Service
8. View Service
9. Verify information
10. Build image with npm run image:build
11. Publish and release image with npm run aws:publish
12. On the Service Events tabs keep an eye on health check errors 

#### Update package.json
```json
"awsRegion": "us-east-1",
"awsEcsCluster": "prod-ecs-cluster",
"awsService": "app-name"
```
#### Setup Logs
1. cloudwatch -> logs
2. Create Log group
3. app-name-logs
 
#### Route 53 DNS Update
> If you don't use Route 53, don't panic. Just create an A record to the ELB's DNS address and you're done.
1. hosted zone
2. select domain
3. create record set
4. alias 'yes'
5. Select ELB App load balancer from the list
6. Create

## Phew!!
### Now what? 
Now you need to deploy an application on your newly-minted cloud infrastructure. Enable fantastic Blue-Green deployments with _[npm scripts for AWS ECS]_(https://gist.github.com/duluca/2b67eb6c2c85f3d75be8c183ab15266e#file-npm-scripts-for-aws-ecs-md).

### Then what?
Go to the ELB DNS address and see if your app works.
If you used Route 53 to connect your domain with your ELB or through your own DNS provider, then go to the URL and see if things work.

### I Would Like to Persist Data
If you'd like to persist data in your containers via Docker volume mounting, then configure EFS. See [this guide](https://gist.github.com/duluca/ebcf98923f733a1fdb6682f111b1a832#file-awc-ecs-access-to-aws-efs-md).

## Troubleshooting
1. ELB DNS works, but URL doesn't? Your DNS configuration is wrong.
2. ELB DNS doesn't work. Then check the health of your ECS Service, see step 3 below.
3. Go to ECS -> Your Cluster -> click on Your Service and switch to the events tab:
If you don't see `service your-app has reached a steady state.` then your container is having trouble starting or AWS is failing to perform a health check.
4. To see what's wrong with your container, go to the Cloudwatch Logs you setup earlier and you'll be able to see the console logs of your application.
5. Service is healthy, logs look fine. Things still don't work? Then re-check security group port rules and target group port rules and any AWS IAM security role you may have setup or may be overriding some default behavior that hasn't been covered. 
6. Call someone who knows better :)
