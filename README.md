# Ansible Configuration Using Jenkins

You can use this cool feature for visitors count on your profile:
```
![Visitor Count](https : //profile-counter.glitch.me/{YOUR USER}/count.svg)
```

This documentation will help you understand the configuration of 2 EC2 instances on aws and configuring them as apache server (using `dynamic inventory`) with custom webpage that will show the Private and Public IP of the slave node whose ip you are hitting.

And Further also we will explore about creating ec2 with diffenet tags for `test` and `prod` environment.

We'll start from the very basics. And we'll setup dynamic inventory in order to complete our goals.

## Table of Contents
- [Installing Requirements](#installing-requirements)
- [Permissions to access AWS](#step-2-provide-your-aws-credentials)
- [Playbooks Needed](#step-3-create-playbooks-and-ansiblecfg-file)
- [Jenkins Deployment](#step-4-jenkins-deployment)




## Installing Requirements

Step 1: Install all your requirements for dynamic inventory using aws_ec2 ansible plugins:

    You must have the following versions of software packages:

    - python ≥ 3.10
    - boto3 ≥ 1.18.0
    - botocore ≥ 1.21.0
    - Install ansible on your machine with the following commands:


```
 sudo apt update
 ```
```
sudo apt install python3-pip
sudo pip install boto boto3

```

 ```
 sudo apt install software-properties-common
 ```
 ```
 sudo apt-add-repository --yes --update ppa:ansible/ansible
 ```
 ```
 sudo apt install ansible
```

**Note: Your Ansible version must be higher than 2.9.* in order for the AWS EC2 plugin to function.

If you use the Ansible ppa for installation, install pip using the following command:

```
sudo apt-get install python-boto3
```
Install amazon.aws collection using the ansible-galaxy command:

```
ansible-galaxy collection install amazon.aws
```

## Step 2: Provide your AWS Credentials 

On Ansible Contol Node:

Use either `aws configure`  or export the `aws_access_key`, `aws_secret_key` to environment variables

By default, these cred will be saved in default profile but we will change the profile to `devops`.

```
aws configure --profile devops
```
we will use this profile for creating instances on `AWS`.

## Step 3: Create Playbooks and ansible.cfg file

Here, I have two playbooks.

- First is for creating 2 EC2 instances and, 
- Second is to configure them as a webserver by installing apache on both.

Add on, our playbook will have the ec2 configuration and variables that will be passed through the Jenkins Environment Variables.

And, In the second playbook we also want to change the document root for our apache.

First playbook name is `playbook.yml`. And, second playbook is `webserver.yml` {view them for reference.}

```
playbook.yml
---
- hosts: localhost  # This playbook is executed locally on the Ansible control node.
  gather_facts: false  # Disable gathering facts about the local system.

  vars:
    region: "{{ region }}"  # Variables to be replaced with actual values during execution.
    instance_type: "{{ instance_type }}"
    ami: "{{ image_id }}"
    keypair: "{{ key_name }}"
    subnetid: "{{ subnet_id }}"
    aws_profile: "{{ aws_profile }}"

  tasks:
    - name: Create EC2 Key Pair  # This task creates an EC2 key pair.
      ec2_key:
        profile: "{{ aws_profile }}"  # AWS profile to use for authentication.
        region: "{{ region }}"  # AWS region where the key pair will be created.
        name: my_keypair  # Name of the key pair.
        key_material: "{{ lookup('file', '/home/ubuntu/managedkey.pub') }}"  # Public key material is read from the file.

    - name: Launch EC2 Instance for Test  # This task launches an EC2 instance for testing.
      ec2_instance:
        key_name: "{{ keypair }}"  # Key pair to associate with the instance.
        profile: "devops"  # AWS profile to use.
        security_group: default  # The security group for the instance.
        instance_type: "{{ instance_type }}"  # Instance type, e.g., t2.micro.
        image_id: "{{ ami }}"  # The Amazon Machine Image (AMI) to use.
        wait: true  # Wait for the instance to be in a running state.
        region: "{{ region }}"  # AWS region for the instance.
        vpc_subnet_id: "{{ subnetid }}"  # The ID of the subnet where the instance will be launched.
        count: 1  # Launch a single instance.
        tags:  # Tags to associate with the instance for easy identification.
          Env: test
          Name: web1
          Team: dev
        network:
          assign_public_ip: true  # Assign a public IP address to the instance.
```
```
webserver.yml
---
- hosts: aws_ec2  # Target hosts where tasks will be executed.
  become: true  # Run tasks with escalated privileges.

  tasks:
    - name: Update package lists  # Update the package lists on the remote server.
      apt:
        update_cache: yes

    - name: Install Apache2  # Install the Apache2 package.
      apt:
        name: apache2
        state: latest

    - name: "Creating Document root"  # Create a document root directory.
      file:
          state: directory
          dest: "/home/ubuntu/app/"

    - name: Webpage  # Create a custom web page with server information.
      copy:
        content: "<h1> WebPage of Ansible-Slave node Whose Private IP is {{ ansible_facts['default_ipv4']['address'] }} and public IP is {{ inventory_hostname }} </h1>"
        dest: "/home/ubuntu/app/index.html"

```



And now we'll create the dynamic inventory file named as `inventory.aws_ec2.yml` `aws_ec2.yml is required` in the name of dynamic inventory file without this it will not work.


NOTE: Your ansible server should that required permission {you have to attach a role to your ec2 instance and give sufficient permissions to it. I have given EC2FullAccess in my case.}


```
---
plugin: aws_ec2
# aws pulgin for ec2
regions:
  - us-east-1
# by default it will search in all region, but you can specify a specific region as well.
filters:
  instance-state-name : running
keyed_groups:
  - key: tags
    prefix: tag
# a way to group your instances by name or OS or can be anything
hostnames:
  - ip-address
# output details of inventory will be ip-address
```

NOTE: I have made some changes in my inventory according to the requirement. Remember that by `default` the inventory will show the `running instance only`.

Now one more step is required,

Go to `cd /etc/ansible/`

and `sudo vi ansible.cfg`

and provide the following details:

```
[defaults]
inventory = /home/ubuntu/inventory.aws_ec2.yml
# provide the path to your inventory file

[inventory]
enable_plugins = aws_ec2
```

Now, run the command:

```
ansible-inventory --graph
```

```
ubuntu@ip-x-x-x-x:/etc/ansible$ ansible-inventory --graph
@all:
  |--@aws_ec2:
  |  |--ip-x-x-x-x.ec2.internal
  |  |--ip-x-x-x-x.ec2.internal
  |--@ungrouped:
ubuntu@ip-1x-x-x-x:/etc/ansible$
```

This output confirms that your `dynamic inventory` is created based on your requirements.

Congratulations, your inventory is now ready.

Now we have to setup the Jenkins serrver which will run the playbooks on our Ansible control node.

## Step 4: Jenkins Deployment 

Setup jenkins server using this offical documentation:

https://www.jenkins.io/doc/book/installing/linux/

I have used SSH Agent Plugin you can also use the worker node for running playbooks.

Go to manage jenkins -> Pulgins -> install SSH plugins

Then, provide the private key in the global credentials with username: ubuntu and private_key also.

Manage Jenkins -> Configure -> SSH Agent 

hostname : `ipv4_address of ansible node`

and according to your need you can do more modifications.

We will give the required details of variables in the environment variables. I have used the environment variables in the pipeline using `-e` which set additional variables as key=value.



- Create the Jenkins Job using pipeline that will the following work -

1. Get the playbooks from github to your ansible control node.

2. Runs the first playbook that will create the 2 EC2 instances  with the provided environment variables through Jenkins ENV.

3. Wait for them to come up.

4. Configure the apache on both managed node or slave nodes with custom web page.

The Jenkinsfile is 

```
pipeline {
  agent any  // Execute the pipeline on any available Jenkins agent.

  stages {
    stage('Execute Ansible playbook') {  // Define the 'Execute Ansible playbook' stage.
      steps {
        sshagent(['controlNode']) {  // Start an SSH agent to securely connect to remote servers.

          // Clone a Git repository on the remote server using SSH.
          sh 'ssh -tt -o StrictHostKeyChecking=no ubuntu@${ip} git clone https://github.com/GauravBarakoti/ansibleWithJenkins.git'

          // Execute an Ansible playbook on the remote server with environment variables.
          sh 'ssh -tt -o StrictHostKeyChecking=no ubuntu@${ip} "cd ansibleWithJenkins; ansible-playbook playbook.yml -e region=$REGION -e instance_type=$INSTANCE -e image_id=$AMI -e key_name=$KEY -e subnet_id=$SUBNET -e aws_profile=$AWS_PROFILE;"'

          // Sleep for 40 seconds (this is for the Ansible playbook to complete ec2 povision tasks).
          sh 'sleep 40'

          // Execute another Ansible playbook on the remote server.
          sh 'ssh -tt -o StrictHostKeyChecking=no ubuntu@${ip} "cd ansibleWithJenkins;ansible-playbook webserver.yml;"'
        }
      }
    }
  }
}
```


In this way, we can achieve Ansible Configuration using Jenkins as a CI/CD.

Feel free to adapt this documentation to your specific requirements and Flask application configuration.


# **Thank You**

I hope you find it useful. If you have any doubt in any of the step then feel free to contact me.
If you find any issue in it then let me know.

<!-- [![Build Status](https://img.icons8.com/color/452/linkedin.png)](https://www.linkedin.com/in/gaurav-barakoti-27002223b) -->


<table>
  <tr>
    <th><a href="https://www.linkedin.com/in/gaurav-barakoti-27002223b" target="_blank"><img src="https://img.icons8.com/color/452/linkedin.png" alt="linkedin" width="30"/><a/></th>
    <th><a href="mailto:bestgaurav1234@gmail.com" target="_blank"><img src="https://img.icons8.com/color/344/gmail-new.png" alt="Mail" width="30"/><a/>
</th>
  </tr>
</table>









