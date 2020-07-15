# Get Started with GPU Accelerated Spark Using AWS EMR Notebook

This is a getting started guide for XGBoost4J-Spark on AWS EMR. At the end of this guide, the user will be able to run a sample Apache Spark application that runs on NVIDIA GPUs on AWS EMR.

For more information on AWS EMR, please see the [AWS documentation](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-what-is-emr.html).

### Configure and Launch AWS EMR with GPU Nodes

Go to the AWS Management Console and select the `EMR` service from the "Analytics" section. Choose the region you want to launch your cluster in, e.g. US West Oregon, using the dropdown menu in the top right corner. Click `Create cluster` and select `Go to advanced options`, which will bring up a detailed cluster configuration page.

##### Step 1:  Software and Steps

Select **emr-5.27.0** or latest EMR version for the release, uncheck all the software options, and then check **Hadoop 2.8.5** and **Spark 2.4.4**.  (Any EMR version that supports Spark 2.3 or above will work).

In the "Edit software settings" field, add the following snippet to disable Spark Dynamic Allocation by default: `[{"classification":"spark-defaults","properties":{"spark.dynamicAllocation.enabled":"false"}}]`

![Step 1: Software and Steps](pics/emr-step-one-software-and-steps.png)

##### Step 2: Hardware

Select the desired VPC and availability zone in the "Network" and "EC2 Subnet" fields respectively. (Default network and subnet are ok)

In the "Core" node row, change the "Instance type" to **g4dn.xlarge**, **g4dn.2xlarge**, or **p3.2xlarge** and ensure "Instance count" is set to **2**. Keep the default "Master" node instance type of **m5.xlarge** and ignore the unnecessary "Task" node configuration.

![Step 2: Hardware](pics/emr-step-two-hardware.png)

##### Step 3:  General Cluster Settings

Enter a custom "Cluster name" and make a note of the s3 folder that cluster logs will be written to.

*Optionally* add key-value "Tags", configure a "Custom AMI", or add custom "Bootstrap Actions"  for the EMR cluster on this page.

![Step 3: General Cluster Settings](pics/emr-step-three-general-cluster-settings.png)

#####  Step 4: Security

Select an existing "EC2 key pair" that will be used to authenticate SSH access to the cluster's nodes. If you do not have access to an EC2 key pair, follow these instructions to [create an EC2 key pair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#having-ec2-create-your-key-pair).

*Optionally* set custom security groups in the "EC2 security groups" tab.

In the "EC2 security groups" tab, confirm that the security group chosen for the "Master" node allows for SSH access. Follow these instructions to [allow inbound SSH traffic](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/authorizing-access-to-an-instance.html) if the security group does not allow it yet.

![Step 4: Security](pics/emr-step-four-security.png)

##### Finish Cluster Configuration

The EMR cluster management page displays the status of multiple clusters or detailed information about a chosen cluster. In the detailed cluster view, the "Summary" and "Hardware" tabs can be used to monitor the status of master and core nodes as they provision and initialize.

![Cluster Details](pics/emr-cluster-details.png )

When the cluster is ready, a green-dot will appear next to the cluster name and the "Status" column will display **Waiting, cluster ready**.

![Cluster Waiting](pics/emr-cluster-waiting.png)

In the cluster's "Summary" tab, find the "Master public DNS" field and click the `SSH` button. Follow the instructions to SSH to the new cluster's master node.

![Cluster DNS](pics/emr-cluster-dns.png)

![Cluster SSH](pics/emr-cluster-ssh.png)

#####  Above Cluster can also be built using AWS CLI

For g4dn.xlarge
```
aws emr create-cluster --termination-protected --applications Name=Hadoop Name=Spark --tags 'name=nvidia-gpu-spark' --ec2-attributes '{"KeyName":"your-key-name","InstanceProfile":"EMR_EC2_DefaultRole","SubnetId":"your-subnet-ID","EmrManagedSlaveSecurityGroup":"your-EMR-slave-security-group-ID","EmrManagedMasterSecurityGroup":"your-EMR-master-security-group-ID"}' --release-label emr-5.30.0-preview --log-uri 's3n://your-s3-bucket/elasticmapreduce/' --instance-groups '[{"InstanceCount":2,"InstanceGroupType":"CORE","InstanceType":"g4dn.xlarge","Name":"Core - 2"},{"InstanceCount":1,"EbsConfiguration":{"EbsBlockDeviceConfigs":[{"VolumeSpecification":{"SizeInGB":32,"VolumeType":"gp2"},"VolumesPerInstance":2}]},"InstanceGroupType":"MASTER","InstanceType":"m5.xlarge","Name":"Master - 1"}]' --configurations '[{"Classification":"spark-defaults","Properties":{"spark.dynamicAllocation.enabled":"false"}}]' --auto-scaling-role EMR_AutoScaling_DefaultRole --ebs-root-volume-size 10 --service-role EMR_DefaultRole --enable-debugging --name 'nvidia-gpu-spark' --scale-down-behavior TERMINATE_AT_TASK_COMPLETION --region us-east-1
```


For P3.2xlarge
```
aws emr create-cluster --termination-protected --applications Name=Hadoop Name=Spark --tags 'Name=nvidia-gpu-spark' --ec2-attributes '{"KeyName":"your-key-name","InstanceProfile":"EMR_EC2_DefaultRole","SubnetId":"your-subnet-ID","EmrManagedSlaveSecurityGroup":"your-EMR-slave-security-group-ID","EmrManagedMasterSecurityGroup":"your-EMR-master-security-group-ID"}' --release-label emr-5.27.0 --log-uri 's3n://your-s3-bucket/elasticmapreduce/' --instance-groups '[{"InstanceCount":1,"EbsConfiguration":{"EbsBlockDeviceConfigs":[{"VolumeSpecification":{"SizeInGB":32,"VolumeType":"gp2"},"VolumesPerInstance":2}]},"InstanceGroupType":"MASTER","InstanceType":"m5.xlarge","Name":"Master - 1"},{"InstanceCount":2,"EbsConfiguration":{"EbsBlockDeviceConfigs":[{"VolumeSpecification":{"SizeInGB":32,"VolumeType":"gp2"},"VolumesPerInstance":4}]},"InstanceGroupType":"CORE","InstanceType":"p3.2xlarge","Name":"Core - 2"}]' --configurations '[{"Classification":"spark-defaults","Properties":{"spark.dynamicAllocation.enabled":"false"}}]' --auto-scaling-role EMR_AutoScaling_DefaultRole --ebs-root-volume-size 10 --service-role EMR_DefaultRole --enable-debugging --name 'nvidia-gpu-spark' --scale-down-behavior TERMINATE_AT_TASK_COMPLETION --region us-west-2
```
Fill with actual value for KeyName, SubnetId, EmrManagedSlaveSecurityGroup, EmrManagedMasterSecurityGroup, S3 bucket for logs, name and region.


### Build and Execute XGBoost-Spark examples on EMR

SSH to the EMR cluster's master node and run the following steps to setup, build, and run the XGBoost-Spark examples.

#### Install git and maven

```
sudo yum update -y
sudo yum install git -y
wget http://apache.mirrors.lucidnetworks.net/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.zip
unzip apache-maven-3.6.3-bin.zip
export PATH=/home/hadoop/apache-maven-3.6.3/bin:$PATH
mvn --version
```
