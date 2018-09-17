# OpenShift Origin Cluster on AWS

This tutorial covers the installation of an OpenShift Origin (now OKD) cluster on AWS with a working Cloudformation template.
Credits go to Sysdig for providing an excellent blog post about the whole procedure: 
https://sysdig.com/blog/deploy-openshift-aws/

## Prerequisites
- Basic knowledge of AWS (especially EC2, S3 and Cloudformation)
- AWS account 
- EC2 Keypair
- AWS cli installed
- S3 bucket

## Get started
### Get image id

```bash
$ aws ec2 describe-images --owners aws-marketplace --filters Name=product-code,Values=aw0evgkw8e5c1q413zgy5pjce --query 'Images[*].[CreationDate,Name,ImageId]' --filters "Name=name,Values=CentOS Linux 7*" --region eu-central-1 --output table | sort -r
|                                                               DescribeImages                                                                |
|  2018-06-13T15:57:10.000Z|  CentOS Linux 7 x86_64 HVM EBS ENA 1805_01-b7ee8a69-ee97-4a49-9e68-afaee216db2e-ami-77ec9308.4  |  ami-dd3c0f36  |
|  2018-05-17T09:03:15.000Z|  CentOS Linux 7 x86_64 HVM EBS ENA 1804_2-b7ee8a69-ee97-4a49-9e68-afaee216db2e-ami-55a2322a.4   |  ami-9a183671  |
|  2018-04-04T00:12:23.000Z|  CentOS Linux 7 x86_64 HVM EBS ENA 1803_01-b7ee8a69-ee97-4a49-9e68-afaee216db2e-ami-8274d6ff.4  |  ami-2882ddc3  |
|  2017-12-05T14:48:56.000Z|  CentOS Linux 7 x86_64 HVM EBS 1708_11.01-b7ee8a69-ee97-4a49-9e68-afaee216db2e-ami-95096eef.4   |  ami-1e038d71  |

```
### Subscription
To use the centos7 image you need to go on the AWS marketplace and subscribe for using the image. It's free so don't hesitate ;)

### Upload the template to the S3 bucket

## Cloudformation
### Create stack
On the AWS web console go to **Cloudformation** and click on _Create stack_.
Now upload the template in yaml format (json is also possible).

### Set values for parameters
Set the parameters for:
- Availability Zone
- Key Name (in pem format)-> <YOUR_EC2_KEYPAIR>
- Instance type

### Start creation of the stack 
The stack creation can take 10-20 minutes depending on the stack. You can see in the events if some error occurs.
When the stack is complete you can see a big fat **CREATION_COMPLETE**.

## Stack operation
### Start instances
If the instances are not started automatically you can start them on the web console in **EC2**.
You can see all the information of the chosen instance when you click on it. There are _Instance ID_, _Public DNS_, _Public IP_ and else.

### Show logoutput of the instances
```bash
$ aws ec2 get-console-output --instance-id <instance_id>
```
Example:
```bash
$ aws ec2 get-console-output --instance-id i-0dcdb53f1d3277e11
{
    "InstanceId": "i-0dcdb53f1d3277e11", 
    "Output": "s)\r\n[    0.076007] Initializing cgroup subsys memory\r\
...
```

If there is an error message that you don't have permission to read the key you have to change permissions:
```bash
$ chmod 400 test_centos.pem
```

### Now ssh into the OpenShift master
```bash
$ ssh -i <EC2_KEYPAIR> centos@<PUBLIC_DNS_NAME>
```

You should now have a ssh console open:
```bash
[centos@ip-10-0-0-5 ~]$ hostname
ip-10-0-0-5.eu-central-1.compute.internal
```

## Preparation for Ansible 

### Set root password
On all instances you can set now a root password
```bash
$ sudo passwd
```

### Next set the password for the user centos
```bash
$ sudo passwd centos
```

### Enable password authentication in sshd (on all nodes!)
Now enable password authentication in sshd conf otherwise ssh-copy-id will not work! It needs the password only for the transaction afterwards you can disable it again.
```bash
$ sudo vi /etc/ssh/sshd

// Comment the lines like this:
#PasswordAuthentication yes
#PermitEmptyPasswords no
PasswordAuthentication no

// Then restart sshd
$ sudo systemctl restart sshd
```

### Generate ssh key (No passphrase!)
$ ssh-keygen

### Distribute ssh key 
Use the following loop to distribute the generated ssh key. Use the private DNS names of your nodes.

```bash
for host in ip-10-0-0-5.eu-central-1.compute.internal ip-10-0-0-5.eu-central-1.compute.internal ip-10-0-0-13.eu-central-1.compute.internal ip-10-0-0-4.eu-central-1.compute.internal; \
    do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; \
    done
```


### Install the correct Ansible from Epel repositories
```bash
yum -y install \
    https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
``` 

### Enable the repos 
```bash
$ sudo sed -i -e "s/^enabled=1/enabled=0/" /etc/yum.repos.d/epel.repo
``` 


### Check with ansible ping if everything worked
[centos@ip-10-0-0-5 ~]$ ansible -m ping all
ip-10-0-0-5.eu-central-1.compute.internal | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
ip-10-0-0-13.eu-central-1.compute.internal | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
ip-10-0-0-4.eu-central-1.compute.internal | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}

## Prepare the OpenShift installation

### Clone Openshift-Ansible repo
```bash
[centos@ip-10-0-0-5 ~]$ git clone https://github.com/openshift/openshift-ansible
```

### Checkout the latest release branch
```bash
[centos@ip-10-0-0-5 openshift-ansible]$ git checkout release-3.10
Zweig release-3.10 konfiguriert zum Folgen von externem Zweig release-3.10 von origin.
Gewechselt zu einem neuen Zweig 'release-3.10'
```

### Create inventory
Check the latest Origin Documentation for the recent exemplary inventory:
https://docs.okd.io/latest/install/example_inventories.html#single-master

Now set the public dns names of your nodes in the inventory (don't forget to put the masters also in the nodes!)
```bash
ip-10-0-0-5.eu-central-1.compute.internal openshift_node_group_name="node-config-master-infra" openshift_node_labels="{'region':'infra','zone':'east'}" openshift_schedulable=true
ip-10-0-0-13.eu-central-1.compute.internal openshift_node_group_name="node-config-compute" openshift_node_labels="{'region': 'primary', 'zone': 'east'}"
ip-10-0-0-4.eu-central-1.compute.internal openshift_node_group_name="node-config-compute" openshift_node_labels="{'region': 'primary', 'zone': 'east'}"
```

### Prepare Playbook ausführen
ansible-playbook all prepare.yml

### Install prerequisites packages
$ ansible all -m shell -a "yum install wget git net-tools bind-utils yum-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct -y" --become

### Updates 
$ ansible all -m shell -a "yum update -y" --become


### Check prerequisites
```bash
ansible-playbook [-i /path/to/inventory] \
    ~/openshift-ansible/playbooks/prerequisites.yml
```

# Install cluster
If everything has been working fine till now let's install the OpenShift cluster:
```bash
ansible-playbook [-i /path/to/inventory] \
    ~/openshift-ansible/playbooks/deploy_cluster.yml
```
This step can take a long time! So grab a coffee.

### Check installation
```bash
[centos@ip-10-0-0-5 ~]$ oc get nodes
NAME                                         STATUS    ROLES          AGE       VERSION
ip-10-0-0-13.eu-central-1.compute.internal   Ready     compute        9m        v1.10.0+b81c8f8
ip-10-0-0-4.eu-central-1.compute.internal    Ready     compute        9m        v1.10.0+b81c8f8
ip-10-0-0-5.eu-central-1.compute.internal    Ready     infra,master   13m       v1.10.0+b81c8f8
```

## After the installation
### Create a user
```bash
$ sudo htpasswd /etc/origin/passwd <USERNAME>
```

### Provide the role cluster-admin to user
```bash
$ oc adm policy add-cluster-role-to-user cluster-admin <USERNAME>
```

### Check acess on web console
The web console can be accessed over:
```bash
https://<PUBLIC_DNS>:8443
```

## Next steps
### Update the Stack
The stack can be updated by uploading a new template file. AWS is checking if everything is according the configuration.

### change instances
You can change the instances by shutting down the instance (on OS level) and stop the instance (on AWS).
Then you can change the instancetype and start it again.

