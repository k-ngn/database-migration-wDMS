
## Infrastructure Diagram

- **AWS Infras**
![image](https://drive.google.com/uc?export=view&id=1rXMicto2C_MDjTMv-5mgyOoe9VoJEk3N)


---
## Demo Structures

1. Provision the environment and review tasks
2. Configure On-premises Side infrastructure 
3. Establish Private Connectivity between the environments (VPC Peer)
4. Create & configure the AWS Side infrastructure (App and DB)
5. Migrate Database & Cutover
6. Clean up

---
## Provision the environment and review tasks

Click the link to deploy the **Base-infras deployment:** [Stack deployment](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateURL=https://aws-labs-resources-kng318.s3.amazonaws.com/cfn-templates/DMS.yaml&stackName=DMS)

After the stack is in `CREATE_COMPLETE` state, this stack will create a simulated **on-premises** environment and an AWS environment using 2 different VPCs then we can connect them together using VPC Peering (simulate VPN/Direct Connect connection).

---
## Configure On-premises Side infrastructure

Move on to EC2 console, [click here](https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#Home:) --> Click on **Instances** in the left sidebar, you'll see 2 EC2 Instances named `SimpleCRUDApp` and `DB` use to simulated the Application and Database run on-premises. Now we need to configure a final step for it to work.
- Choose the `DB` instance and copy down it Public IPv4
- Choose the `SimpleCRUDApp` and click **Connect** --> Connect with Session Manager -> Inside the terminal, type `sudo -i`
	- Move to the webroot folder `cd /var/www/html`
	- Edit the file named `dbcon.php` with `vi dbcon.php`
	- Press `I` and replace `DB_Public_IPv4_Address` with your DB's IPv4 you're just copied. Then press `ESC` type `:wq` to save the file
	- Run `systemctl start httpd` to start the webserver

Now you can test the application by copy the IPv4 of the `SimpleCRUDApp` and paste it in another tab. -> In the page, you can add any data you want to indicate the data is migrated complete in the Migrating DB step.

---
## Establish Private Connectivity between the environments (VPC Peer)

#### Create a VPC Peer between On-prem and AWS

Move to the VPC console, [click here](https://console.aws.amazon.com/vpc/home?region=us-east-1#) -> Click on the **Peering Connections** in the left sidebar -> Click `Create Peering Connection`
- `Peering Connection name tag` = `ON-PREM-TO-AWS` 
- `VPC (Requester)` = `ONPREMVPC` 
- `VPC (Accepter)` = `AWSVPC`
- Scroll down and create the peering connection
- Click `Actions` and `Accecpt Request` to accept peering request

#### Create Routes on the On-prem

Move to the **route tables** console in the left sidebar of VPC console -> Locate and select the `OnpremPublicRouteTable` -> Click on the `Routes` tab, now we're going to add a route to point traffics to AWS side using the VPC Peer
- Click `Edit Routes`
- Click `Add Route`
- `Destination` = `10.16.0.0/16`
- Click `Target` dropdown -> Choose `Peering Connection`--> Select the `ON-PREM-TO-AWS` connection 
- Save the route

> Now traffic can go from On-prem side to AWS side, but for bi-directional traffic flow, we need to do the same in the AWS side 

#### Create Routes on the AWS side

You can following the same steps in creating route for the On-Premises but change the `Destination` to point to the On-Prem side. 

In the **route tables** console -> Locate and select the `AWSPublicRouteTable` -> Click on the `Routes` tab 
- Click `Edit Routes`
- Click `Add Route`
- `Destination` = `192.168.10.0/24`
- Click `Target` dropdown -> Choose `Peering Connection`--> Select the `ON-PREM-TO-AWS` connection 
- Save the route

Following the same steps as you've just performed with `AWSPublicRouteTable` to add the same route to the `AWSPrivateRouteTable`

> At this point, you've completly created the peering connection between both VPC to simulated VPN/Direct Connect connection between On-Premises and AWS

---
## Create & configure the AWS Side infrastructure (App and DB)

#### Create the RDS instance

Move to RDS Console, [click here](https://console.aws.amazon.com/rds/home?region=us-east-1) -> Click **Subnet groups** on the left sidebar -> Then click `Create DB Subnet Group`
- `Name` & `Description` = `AWS-DBSubnetGroup`
- `VPC` = `AWSVPC`
- `Availability zones` = `us-east-1a` & `us-east-1b`
- `Subnets` = `10.16.32.0/20` (privateA) & `10.16.96.0/20` (privateB)
- Scroll down and create

Click **Databases** on the left sidebar -> Then `Create database`
- `Creation method` = `Standard Create`
- `Engine` = `MySQL`
- `Version` = `MySQL 8.0.35`
- `Template` = `Free tier`
	> We're going to use the same database credentials with the on-prem DB to keep things simple for the demo, but in production this could be different.
- `DB instance identifier` = `DMSDemoDB`
- `Master username` = `remoteadmin`
- `Master password` & `Confirm password` = `MySQLPassW0rd`
- `VPC` = `AWSVPC`
- `DB subnet groups` = `aws-dbsubnetgroup`
- `Public access` = `No`
- `VPC security groups` = `Choose Existing` -> `***-awsSecurityGroupDB-***` (*** is stack name and random values)
- Expand `Additional configuration`, for `Initial database name` = `demo`
- Scroll down and create database

> Wait until it create complete to continue.

#### Create the EC2 instance
Move to EC2 console, [click here](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Home) -> Click **Instances** on the left sidebar -> Then `Launch instance`.
- `Name` = `AWS-SimpleCRUDApp`
- `OS` = `Amazon Linux` -> `Amazon Linux 2023 (Free tier)`
- `Architecture` = `64-bit(x86)`
- `Instance type` = `t2/t3.micro` (free tier)
- `Keypair` = `Process without ...`
- Click `edit` the `Network Settings`,
- `VPC` = `AWSVPC`
- `Subnets` = `aws-PublicA`
- `Security Groups` = `Select an existing group` -> `***-awsSecurityGroupWeb-***` (*** is stack name and random values)
- Scroll down and expand `Advanced Details`, for `IAM Instance Profile` = `***-awsInstanceProfile-***` (*** is stack name and random values)
- Paste this into the user-data
```bash
#!/bin/bash -xe
dnf update -y
dnf install httpd mod_ssl php php-mysqlnd -y

systemctl enable httpd

systemctl start php-fpm
systemctl enable php-fpm

wget https://aws-labs-resources-kng318.s3.amazonaws.com/materials/simplecrudwphpmysql.zip
unzip simplecrudwphpmysql.zip -d /var/www/html

```
	 
- Scroll down and launch the instance
- Click `View All Instances`

> Wait until the instance is in running state with 2/2 health checks before continuing.

Choose the `AWSSimpleCRUDApp` and click **Connect** --> Connect with Session Manager -> Inside the terminal, type `sudo -i`
- Move to the webroot folder `cd /var/www/html`
- Edit the file named `dbcon.php` with `vi dbcon.php`
- Press `I` and replace `DB_Public_IPv4_Address` with your DB's IPv4 you're just copied. Then press `ESC` type `:wq` to save the file
- Run `systemctl start httpd` to start the webserver

Move to EC2 running instances console, [click here](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances) -> Select the `aws-web` instance -> Copy its `Public IPv4 DNS` -> Open it on a new tab with just `http` connection -> If working you will see the same data you enter with the onprem instance, this Web Instance (aws) is now loading using the on-premises database.

---
## Migrate Database & Cutover

#### Create the DMS Subnet Group

Move to the **DMS Subnet Group**, [click here](https://us-east-1.console.aws.amazon.com/dms/v2/home?region=us-east-1#subnetGroup) -> Click **Create subnet group**
- `Name` & `Description` = `DemoDMSSubnetGroup`
- `VPC` = `AWSVPC`
- `Add Subnets` = `aws-privateA` & `aws-privateB`
- Then create subnet group

#### Create the DMS Replication Instance
Move to the **DMS Replication instances**, [click here](https://us-east-1.console.aws.amazon.com/dms/v2/home?region=us-east-1#replicationInstances) -> Click **Create Replication Instance**
- `Name` & `Description` = `ONPREM2AWS`
- `Instance Class` = `dms.t3.micro`
- `High Availability` = `Dev or test workload (Single-AZ)`
- `Virtual private cloud (VPC) for IPv4` = `AWSVPC`
- `Replication subnet group` = `DemoDMSSubnetGroup`
- Uncheck the `Publicly Accessible`
- Expand the `Advanced settings` -> For `VPC security groups` = `***-awsSecurityGroupDB-***` (*** is stack name and random values)
- Scroll down and create

#### Create the DMS Source Endpoint

Move to the **DMS Endpoints**, [click here](https://us-east-1.console.aws.amazon.com/dms/v2/home?region=us-east-1#endpointList) -> Click **Create endpoint**
- `Endpoint type` = `Source endpoint`
	> Make sure `Select RDS DB instance` is unchecked. (Because our source on-prem DB is on EC2)
- `Endpoint identifier` = `OnpremDB`
- `Source Engine` = `MySQL`
- Choose `Provide access information manually`
- `Server name` = `Private_IPv4_of_OnpremDB` (Get from the EC2 console)
- `Port` = `3306`
- `User name` = `remoteadmin` 
- `Password` = `MySQLPassW0rd`
- Scroll down and create the endpoint

#### Create the DMS Destination Endpoint (RDS)

Create another endpoint in the DMS Endpoints console
- `Endpoint type` = `Target endpoint`
- Check the `Select RDS DB instance`
- `RDS Instance` = `DMSDemoDB` -> It will prepopulate the boxes
- Scroll down and Choose `Provide access information manually`
- `Password` = `MySQLPassW0rd`
- Scroll down and create the endpoint

#### Test the Endpoint

Before we move on to the main part of this demo **make sure the replication instance is ready**. Verify by going to **Replication Instances** in the left sidebar and make sure the status is `Available`  

Go back to **Endspoints**
- Select the `DMSDemoDB` endpoint, click `Actions` and then `Test Connections` -> Click `Run Test` and make sure after a few minutes the status moves to `successful`  
- Go back to **Endpoints** -> Select the `OnpremDB` endpoint, click `Actions` and then `Test Connections` -> Click `Run Test` and make sure after a few minutes the status moves to `successful`

If both of these are successful you can continue to the next step.

#### Migrate

Now move to the **Database migration tasks**, [click here](https://us-east-1.console.aws.amazon.com/dms/v2/home?region=us-east-1#tasks) -> Click **Create task**
- `Task identifier` = `MigrateONPREM2AWS`
- `Replication instance` = the one you've created in previous part 
- `Source database endpoint` = `OnpremDB`
- `Target database endpoint` = `DMSDemoDB`
- `Migration type` = `migrate existing data` (**you could pick and replicate changes here if this were a high volume production DB**)
- Scroll down `Table mappings` = `Wizards`
- Click `Add new selection rule`
- `Schema` = `Enter a Schema`
- `Source name` = `demo`
- Scroll down and create the task

This will start the replication task and does a full load from on-premises`DB` to the RDS Instance -> It will create the task -> Then start the task -> Then it will be in the `Running` State until it moves into `Load complete`.

At this point the data has been migrated into the RDS instance

#### Cutover the application instance

Move to the RDS Console, [click here](https://console.aws.amazon.com/rds/home?region=us-east-1#) -> Click **Databases** on the left sidebar -> Click `DMSDemoDB` -> Under `Endpoint & Port` copy the `Endpoint` into your clipboard

Move to the EC2 console [https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances):

Choose the `SimpleCRUDApp` and click **Connect** --> Connect with Session Manager -> Inside the terminal, type `sudo -i`
- Move to the webroot folder `cd /var/www/html`
- Edit the file named `dbcon.php` with `vi dbcon.php`
- Press `I` and for the `$servername` value replace on-premises IP address with your RDS Endpoint you're just copied. Then press `ESC` type `:wq` to save the file
- Run `systemctl restart httpd` to start the webserver

Move to the EC2 console [https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances):  
Select `CatDB`, right click `stop instance`  
Move to the EC2 console [https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances):  
Select `CatWeb`, right click `stop instance`

Move to the EC2 console [https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances):  
Select `awsCatWeb` and get its `Public IPv4 DNS` Open this in a new tab It should still show the application ... this is now pointed at the RDS instance after a full migration

---
## Clean up

#### EC2
Move to the EC2 Console [https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances):  
Select `AWS-SimpleCRUDApp`, right click, instance state, terminate instance, terminate

#### RDS
Move to the RDS Console [https://console.aws.amazon.com/rds/home?region=us-east-1#databases](https://console.aws.amazon.com/rds/home?region=us-east-1#databases):  
Click `Databases`  
Select `dmsdemodb`  
Click `Actions` then `Delete`  
Uncheck `Create final snapshot?`  
Check the `I acknowledge...`, type `delete me` into the box and cick `Delete`

**wait for this to delete complete**  
Click on the **Subnet groups** on the left sidebar
Select the `a4ldbsngroup` click `Delete` then `Delete` again

#### DMS
go to DMS Database migration tasks [https://console.aws.amazon.com/dms/v2/home?region=us-east-1#tasks](https://console.aws.amazon.com/dms/v2/home?region=us-east-1#tasks)  
Select the `migrateonprem2aws` task, click `Actions`, `Delete` and then `Delete` again.

Click on **Replication instances** on the left sidebar
Delect the `onprem2aws` replication instance, click `Actions`, `Delete` and then `Delete` again.

Click on **Endpoints** on the left sidebar
Select both the endpoints, click `Actions`, `Delete` and then `Delete`.

**wait for the replication instance to finish deleting** 

Click on **Subnet groups** on the left sidebar
Select `demodmssubnetgroup` -> Click `Actions` -> `Delete` and then `Delete` again.

#### VPC PEER

Move to the VPC Console [https://console.aws.amazon.com/vpc/home?region=us-east-1](https://console.aws.amazon.com/vpc/home?region=us-east-1)  
Click **Route tables** on the left sidebar
Select `AWSPrivateRouteTable`  
Click `Routes`  
Click `Edit Routes`  
Click `X` next to the VPC Peer Route  
Click `Save Routes` and click `Close`  
Select `AWSPublicRouteTable`  
Click `Routes`  
Click `Edit Routes`  
Click `X` next to the VPC Peer Route  
Click `Save Routes` and click `Close`  
Select `OnpremPublicRouteTable`  
Click `Routes`  
Click `Edit Routes`  
Click `X` next to the VPC Peer Route  
Click `Save Routes` and click `Close`

Move to VPC Peering Connections, [click here](https://console.aws.amazon.com/vpc/home?region=us-east-1#PeeringConnections:sort=vpcPeeringConnectionId)  
Select the `VPC Peer` you created earlier `ON-PREM-TO-AWS`, click `Actions`, `Delete Peering Connection`, Click `Delete`

#### Cloudformation 

Move to the cloudformation console [https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks?filteringText=&filteringStatus=active&viewNested=true&hideStacks=false](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks?filteringText=&filteringStatus=active&viewNested=true&hideStacks=false)  
Select the `DMS` stack  
Click `Delete` then `Delete Stack`