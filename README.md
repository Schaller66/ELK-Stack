# Deploying an ELK stack on AWS

The below files and instructions are how to deploy an ELK stack in an AWS setting. A basic diagram below shows the structure it will be based on.

*![ELKDiagram](https://user-images.githubusercontent.com/82848972/128159671-f6e0344c-c7e3-4367-88f5-e9fa2684e05d.png)

The files and permissions below are meant to stand up a live ELK deployment on AWS. There will also be a couple code additions to assist in case of debugging issues occur.

These topics will be discussed in order to deploy the ELK stack.

Markup : - VM's to be depolyed
         - Topology associated with the VM's
         - Access Policies for those VM's
         - ELK Configuration
         - Running Ansible Playbooks

With the VM's the need to be deployed, there needs to be some clarification. We will be deploying our ELK machine, Jump box, and 2 web boxes that we will be using for load balancing and our beat programs.

The table below will provide my list of machines used, IP's associated, their functions and what conatiner they have running.

### Virtual Machine Topology

Name | IP address | OS | Container | Function
| :---: | :---: | :---: | :---: | :---: 
ELK | 10.1.0.183 | AWS Linux | Elk:761 | Elk Server
Jump Box | 10.0.0.54 | AWS Linux | Ansible | SSH Gateway
Web 1 | 10.0.1.155 | AWS Linux | DVWA | Web Server LB 
Web 2 | 10.0.1.241 | AWS Linux | DVWA | web Server LB

The Elk machine in this table is going to be the hub for our front end that will be a host for kibana on a browser, but since AWS is a bit different to Azure, we will have to assign our Elk machine an Elastic IP. This does double duty both giving the machine a public IP we can type into our url header and it properly categorizes or Elk machine for the Elastic search part of our E.L.K. stack.

The Jump Box is our way to access all of our machines and configure them as well. Our jump box is going to have Ansible configured so that we can deploy our appropriate containers all from the same machine so we don't have to do it manually.

Our Web 1 & 2 VM's are simply meant for availability as both of them are going to be under a load balancer. They are also going to be the host for our beats programs, Filebeat and Metricbeat. Filebeat for monitoring log files and locations, and Metricbeat for sending metrics and statistics to Elasticsearch.

### Access Policies

Our machines are up but they have no connections they can make, inbound or outbound. In order to fix this, we need to add some rules to our machines so that they can connect to each other.

Name | Public Access | IP Allowed | Port
| :---: | :---: | :---: | :---:
ELK | No | Private IP | Port 5601
Jump Box | No | Private IP | Port 22
Web 1 | Yes | All ipv4 | Port 80
Web 2 | Yes | All ipv4 | Port 80

The Jump box is only allowed to communicate with port 22 since its purpose is to pull an Ansible container for most of the hard work and to comunicate to the other boxes, for this reason we can leave this box cut off from as much as possible.

Our Elk box needs to stay locked down so for this we are only allowing a connection on Port 5601 so that the website can still be displayed. 

Our web servers are going to be very simple, since they only need to be listening on port 80, they only need the one rule attached. The option is available to add port 443 for a more secure option, but over all it is unnecessary.

#### Troubleshooting Tip

If you encounter any issues in later portions of the Stack deployment, try adding port 22 connections to the box(es) in question and attempting manually. This adds redundancy as a way to more easily diagnose issues.

### ELK configuration and deployment

Instead of deploying everything manually, we instead opted to automate the configuration of the machines using Ansible. This allows a very smooth transition if any additional machines need to be deployed in the future.

Before we run any playbooks we want to make sure we update our Ansible config file and our hosts file to accomodate the private IP addresses of the webserver boxes and the elk machine this will be further explained in a later section.

Our first playbook is going to configure our ELK box.

The `ELKplaybookAWS.yml` runs the following tasks and performs cerrtain actions.

Markup : - Changes a `systemctl` file to allow the container to access more memory so that the container doesn't crash.
         - Installs Docker on the machine
         - Installs Python3 pip
         - Installs Six
         - Starts Docker
         - Installs pip Docker module
         - Pulls the docker container while allowing the required ports that need to be listening, it also changes the base ulimit in order to properly change the memory requirement
         - Start Docker service upon boot

After a successful installation, not only should docker be up and running but the container should also be up and running.

Running `docker ps` should yeild a running container, refer to screenshot below for reference
*![Elkcontainer](https://user-images.githubusercontent.com/82848972/128278101-02cb2591-d867-4e4c-a7b5-256d907c5cf6.PNG)

### Machines & Beat

We have configured our Elk machine to monitor the following machines.

Markup : - Web 1 10.0.1.155
         - Web 2 10.0.1.241

We have installed the following Beats on the machines.

Markup : - Filebeat
         - Metricbeat

Filebeat is used to collect and parse logs created by the system logs of common linux based distros.

Metricbeat fetches the metrics from the docker service.

### Running the Playbooks

Before running these playbooks make sure your control node is provisioned. (Container)

Attach to the container node and procced to the following steps:

Markup : - Copy the `ELKplaybookAWS.yml` file (or whatever you would like to clal it) to the ansible container
         - Open the `hosts` file in the `/etc/ansible` directory add the servers you want to update, in this case we want to add an elk category since we are deploying an Elk playbook
         - Run the playbook `ELKplaybookAWS.yml`. If all worked correctly, go to your brower and type in your ELK VM's public IP on port 5601. You should see a working kibana page like the one below

 ![kibanahome](https://user-images.githubusercontent.com/82848972/128301344-1032acb7-6597-42c9-b9d8-998cc7d33089.PNG)

After the ELK machine is running proerly, we need to install Filebeat and Meticbeat on our web servers.

Markup : - Back in your ansible container, copy the `FilebeatAWS.yml` file to the container.
         - Edit the `hosts` file to contain the webservers and their IP's
         - Run the playbook
         - Go back to kibana and fill out the module status. To find it, Go to kibana> Click Add Log Data > System Logs > `module status` > click `Check Data` . 
         - If you were successful you should see something like this.

![filebeat](https://user-images.githubusercontent.com/82848972/128309453-7e5158f3-c8e4-4395-be1a-dfc04058aea1.PNG)

Rinse and repeat for Metricbeat with some minor changes and your ELK stack is done.
