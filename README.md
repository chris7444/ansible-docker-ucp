# Introduction

Express Containers with Docker is a complete solution from Hewlett Packard Enterprise that includes all the hardware, software, professional services, and support you need to deploy an operational containers-as-a-service (CaaS) platform, allowing you to get up and running quickly and efficiently. The solution takes the HPE hyperconverged infrastructure and combines it with Docker’s enterprise-grade container platform, popular open source tools, along with deployment and advisory services from HPE Pointnext.

Express Containers with Docker is ideal for customers either migrating legacy applications to containers, transitioning to a container DevOps development model or needing a hybrid environment to support container and non-containerized applications on a common VM platform.  Express Containers with Docker provides a solution for both IT development and IT operations, and comes in two versions.  The version for IT developers (Express Containers with Docker: Dev Edition) addresses the need to provide a cloud-like container development environment with built-in container tooling.  The version for IT operations (Express Containers with Docker: Ops Edition) addresses the need to have a production ready environment that is very easy to deploy and manage.  

This document describes the best practices for deploying and operating the Express Containers with Docker: Ops Edition.   It describes how to automate the provisioning of the environment using a set of Ansible playbooks. It also outlines a set of manual steps to harden, secure and audit the overall status of the system. A corresponding document focused on setting up Express Containers with Docker: Dev Edition is also available. TODO.


**Note**
- The Ansible playbooks described in this document are only intended for Day 0 deployment automation of Docker EE on SimpliVity
- The Ansible playbooks described in this document are not directly supported by HPE and are intended as an example of deploying Docker EE on HPE SimpliVity.  We welcome input from the user community via Github to help us prioritize all future bug fixes and feature enhancements

## About Ansible

Ansible is an open-source automation engine that automates software provisioning, configuration management and application deployment.

As with most configuration management software, Ansible has two types of server: the controlling machine and the nodes. A single controlling machine orchestrates the nodes by deploying modules to the nodes over SSH. The modules are temporarily stored on the nodes and communicate with the controlling machine through a JSON protocol over the standard output. When Ansible is not managing nodes, it does not consume resources because no daemons or programs are executing for Ansible in the background. Ansible uses one or more inventory files to manage the configuration of the multiple nodes in the system.

In contrast with other popular configuration management software such as Chef, Puppet, and CFEngine, Ansible uses an agentless architecture. With an agent-based architecture, nodes must have a locally installed daemon that communicates with a controlling machine. With an agentless architecture, nodes are not required to install and run background daemons to connect with a controlling machine. This type of architecture reduces the overhead on the network by preventing the nodes from polling the controlling machine.
More information about Ansible can be found here: [http://docs.ansible.com](http://docs.ansible.com)


## About Docker Enterprise Edition

Docker Enterprise Edition (EE) is a containers-as-a-service (CaaS) platform for IT that manages and secures diverse applications across disparate infrastructure, both on-premises and in the cloud. Docker EE provides integrated container management and security from development to production. Enterprise-ready capabilities like multi-architecture orchestration and secure software supply chain give IT teams the ability to manage and secure containers without breaking the developer experience.

Docker EE provides

- Integrated management of all application resources from a single web admin UI.
- Frictionless deployment of applications and Compose files to production in a few clicks.
- Multi-tenant system with granular role-based access control (RBAC) and LDAP/AD integration.
- Self-healing application deployment with the ability to apply rolling application updates.
- End-to-end security model with secrets management, image signing and image security scanning.


More information about Docker Enterprise Edition can be found here: [https://www.docker.com/enterprise-edition](https://www.docker.com/enterprise-edition)

## About Simplivity

Simplivity is an enterprise-grade hyper-converged platform uniting best-in-class data services with the world's bestselling server.

Rapid proliferation of applications and the increasing cost of maintaining legacy infrastructure causes significant IT challenges for many organizations. With HPE SimpliVity, you can streamline and enable IT operations at a fraction of the cost of traditional and public cloud solutions by combining your IT infrastructure and advanced data services into a single, integrated solution. HPE SimpliVity is a powerful, simple, and efficient hyperconverged platform that joins best-in-class data services with the world’s best-selling server and offers the industry’s most complete guarantee.

More information about Simplivity can be found here: [https://www.hpe.com/us/en/integrated-systems/simplivity.html](https://www.hpe.com/us/en/integrated-systems/simplivity.html)

**Target Audience:** This document is primarily aimed at technical individuals working in the Operations side of the pipeline, such as system administrators and infrastructure engineers, but anybody with an interest in automating the provisioning of virtual servers and containers may find this document useful.
	
**Assumptions:** The present document assumes a minimum understanding in concepts such as virtualization and containerization and also some knowledge around Linux and VMWare technologies.


## Required Versions

The following versions or higher are required to use the playbooks described in later sections.

- Ansible 2.2 or higher
- Docker EE 17.06 or higher
- Red Hat Enterprise Linux v TODO
- VMware vSphere TODO
- HPE SimpliVity Omni stack v TODO


# Architecture

The Operations environment is comprised of three HPE SimpliVity 380 Gen10 servers. Since the SimpliVity technology relies on VMware virtualization, the servers are managed using vCenter. The load among the three hosts will be shared as per Figure 1.


![Architecture Diagram][architecture]
**Figure 1.** Solution Architecture

The Ansible playbooks can be modified to fit your environment and your high availability (HA) needs. By default, the Ansible Playbooks will set up a 3 node environment.  HPE and Docker recommend a minimal starter configuration of 3 physical nodes for running Docker in production.  The distribution of the Docker and non-Docker modules over the 3 physical nodes via virtual machines (VMs) is as follows:

- 3 Docker Universal Control Plane (UCP) VM nodes for HA and cluster management
- 3 Docker Trusted Registry (DTR) VM nodes for HA of the container registry
- 3 worker VM nodes for container workloads
- Docker UCP load balancer VM to ensure access to UCP in the event of a node failure
- Docker DTR load balancer VM to ensure access to DTR in the event of a node failure
- Docker Swarm Worker node VM load balancer
- Logging server VM for central logging 
- NFS server VM for storage Docker DTR images

These nodes are installed on VMs, spread across three SimpliVity servers. Each server will consist of:

- At least one UCP node, but ideally 3 or 5, spread across the 3 SimpliVity servers
- At least one DTR node, but ideally 3 or 5, spread across the 3 SimpliVity servers
- At least one worker node, but ideally 3 or 5, spread across the 3 SimpliVity servers

In addition to the above, the playbooks set up:

- 3 load balancers, one for each set of nodes (one Worker load balancer, one DTR load balancer and one UCP load balancer)
- 1 central logging server 
- 1 NFS server
- Docker persistent storage driver from VMware
- Prometheus and Grafana monitoring tools
- SimpliVity backup policy for data volumes and Docker images inside DTR

These nodes can live in any of the hosts and they are not redundant.


Sizing considerations

This section describes sizing considerations. The vCPU allocations are described in Table 1 while the memory allocation is described in Table 2.

**Table 1** vCPU

| vCPUs | simply01 | simply02 | simply03 |
|:------|:--------:|:--------:|:--------:|
| ucp1  |	4      |          |	         |
| ucp2  |          |4         |          |
| ucp3	|          |          | 4        |
| dtr1  | 2		   |          |          |
| dtr2  |          | 2	      |          |
| dtr3  |          |          |  2       |
| worker1 |4	   |          |          |	
| worker2| 	       | 	4	  | 
| worker3| 		   |          | 	4
| ucb_lb| 	2      | 		  | 
| dtr_lb| 	       | 	2	  | 
| worker_lb	| 	   |          |  2
| nfs	| 	       |          |  2      | 
| logger	| 	   |  2	      | 
| **Total vCPU per node**|	**12** |**14**|	       **14**|
| **Total vCPU**| 	   | **40**	
| Available CPUs| 24 | 24	  | 24
| Log Proc	| 48   | 	48	  | 48
| **Total Log Proc** (on two nodes) | | **96**	






**Table 2** Memory allocation

| RAM | simply01 | simply02 | simply03 |
|:------|:--------:|:--------:|:--------:|
| ucp1  |	8      |          |	         |
| ucp2  |          |8         |          |
| ucp3	|          |          | 8        |
| dtr1  | 16	   |          |          |
| dtr2  |          | 16       |          |
| dtr3  |          |          |  16      |
| worker1 |64	   |          |          |	
| worker2| 	       | 	64	  | 
| worker3| 		   |          | 	64
| ucb_lb| 	4      | 		  | 
| dtr_lb| 	       | 	4	  | 
| worker_lb	| 	   |          |  4
| nfs	| 	       |          |  4      | 
| logger	| 	   |  4	      | 
| **Total RAM required** (per node)| 	**92**	| **96**| 	**96**
| **Total RAM required**| 	| 	**284**	
| Available RAM	| 384	| 384| 	384
| **Total Available RAM** (on two nodes)| | 		**768**	








# Provisioning the operations environment

This section describes in detail how to provision the environment described previously in the architecture section. The figure below shows the high level steps this guide will take.

![Provisioning steps][provisioning]


## Verify Prerequisites

You will need assemble the information required to assign values to each and every variable used by the playbooks before you start deployment. The variables are fully documented in the following sections “Editing the group variables” and “Editing the vault”. A summary of the information required is presented in Table 3.

**Table 3** Summary of information required

|Component|	Details|
|---------|-----------|
|Virtual Infrastructure|	The FQDN of your vCenter server and the name of the Datacenter which contains the SimpliVity cluster. You will also need administrator credentials in order to create templates, and spin up virtual machines.
|SimpliVity Cluster	|The name of the SimpliVity cluster and the names of the member of this cluster as they appear in vCenter. You will also need to know the name of the SimpliVity datastore where you want to land the various virtual machines. You may have to create this datastore if you just installed your SimpliVity cluster. In addition, you will need the IP addresses of the OmniStack virtual machines. Finally you will need credentials with admin capabilities for using the OmniStack API. These credentials are typically the same as you vCenter admin credentials
|L3 Network requirements|	You will need one IP address for each and every VM configured in the Ansible inventory (see the section “Editing the inventory”). At the time of writing, the example inventory configures 14 virtual machines so you would need to allocate 14 IP addresses to use this example inventory. Note that the Ansible playbooks do not support DHCP so you need static IP addresses.   All the IPs should be in the same subnet. You will also have to specify the size of the subnet (for example /22 or /24) and the L3 gateway for this subnet.
|DNS|	You will need to know the IP addresses of your DNS server. In addition, all the VMs you configure in the inventory should have their names registered in DNS. In addition, you will need the domain name to use for configuring the virtual machines (such as example.com)
|NTP Services|	You need time services configured in your environment. The solution being deployed (including Docker) uses certificates and certificates are time sensitive. You will need the IP addresses of your time servers (NTP).
|Docker Prerequisites|	You will need a URL for the official Docker EE software download and a license file.  Refer to the Docker documentation to learn more about this URL and the licensing requirements here: https://docs.docker.com/engine/installation/linux/docker-ee/rhel/ Learn how to download the Docker EE license key here: https://success.docker.com/KBase/How_do_I_download_my_Docker_license_key 
|Proxy	|The playbooks pull the Docker packages from the Internet. If you environment accesses the Internet through a proxy, you will need the details of the proxy including the fully qualified domain name and the port number.



## Install vSphere Docker Volume Service driver on all ESXi hosts

This is a one-off manual step. In order to be able to use Docker volumes using the vSphere driver, you must first install the latest release of the vSphere Docker Volume Service (vDVS) driver, which is available as a vSphere Installation Bundle (VIB). To perform this operation, login to each of the ESXi hosts in turn and then download and install the latest release of vDVS driver.

```
# esxcli software vib install -v /tmp/vmware-esx-vmdkops-<version>.vib --no-sig-check
```

More information on how to download and install the driver can be found here: https://vmware.github.io/docker-volume-vsphere/documentation/install.html#vib-installation-through-esxclilocalcli 








## Create a VM template

The very first step to our automated solution will be the creation of a VM Template that will be the base of all your nodes. In order to create a VM Template we will first create a Virtual Machine where the OS will be installed and then the Virtual Machine will be converted to a VM Template. Since the goal of the automation is get rid of as many repetitive tasks as possible, the VM Template will be created as lean as possible, so any additional software installs and/or system configuration will be done later by Ansible.

It would be possible to automate the creation the template. However, as this is a one-off task, it is appropriate to do it manually. The steps to create a VM template manually are described below.

1. Log in to vCenter and create a new Virtual Machine. In the dialog box, shown in Figure 2, select ```Typical``` and press ```Next```.  
![Create New Virtual Machine][createnewvm]  
**Figure 2** Create New Virtual Machine  
  
2. Specify the name and location for your template, as shown in Figure 3.  
![Specify name and location for the virtual machine][vmnamelocation]  
**Figure 3** Specify name and location for the virtual machine  
  
3. Choose the host/cluster on which you want to run this virtual machine, as shown in Figure 4.
4. Choose a datastore where the template files will be stored, as shown in Figure 5.
5. Choose the OS, in this case Linux, RHEL7 64bit.
6. Pick the network to attach to your template. In this example we're only using one NIC but depending on how you plan to architect your environment you might want to add more than one.
7. Create a primary disk. The chosen size in this case is 50GB but 20GB should be typically enough.
8. Confirm that the settings are right and press Finish.
9. The next step is to virtually insert the RHEL7 DVD, using the Settings of the newly created VM as shown in Figure 10. Select your ISO file in the Datastore ISO File Device Type and make sure that the “Connect at power on” checkbox is checked.
10. Finally, you can optionally remove the Floppy Disk as this is not required for the VM.
11. Power on the server and open the console to install the OS. On the welcome screen, as shown in Figure 12, pick your language and press ```Continue```.
12. The installation summary screen will appear, as shown in Figure 13.
13. Scroll down and click on Installation Destination, as shown in Figure 14.
14. Select your installation drive, as shown in Figure 15, and click Done.
15. Click Begin Installation, using all the other default settings, and wait for the configuration of user settings dialog, shown in Figure 16.
16. Select a root password, as shown in Figure 17.
17. Click Done and wait for the install to finish. Reboot and log in into the system using the VM console.
18.	The Red Hat packages required during the deployment of the solution come from two repositories: ```rhel-7-server-rpms``` and ```rhel 7-server-extras-rpms```. The first repository can be found on the Red Hat DVD but the second cannot. There are two options, with both requiring a Red Hat Network account.
  - Use Red Hat subscription manager to register your system. This is the easiest way and will automatically give you access to the official Red Hat repositories. It does require having a Red Hat Network account though, so if you don’t have one, you can use a different option. Use the ```subscription-manager register``` command as follows.
```
# subscription-manager register --auto-attach
```
If you are behind a proxy, you must configure this before running the above command to register.
```
# subscription-manager config --server.proxy_hostname=<proxy IP> --server.proxy_port=<proxy port>
```
If you follow this “route”, the playbooks will automatically enable the “extras” repository on the VMs that need it.
  - Use an internal repository. Instead of pulling the packages from Red Hat, you can create copies of the required repositories on a dedicated node. You can then configure the package manager to pull the packages from the dedicated node. Your ```/etc/yum.repos.d/redhat.repo``` could look as follows.
```
[RHEL7-Server]
name=Red Hat Enterprise Linux $releasever - $basearch
baseurl=http://yourserver.example.com/rhel-7-server-rpms/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[RHEL7-Server-extras]
name=Red Hat Enterprise Linux Extra pkg $releasever - $basearch
baseurl=http://yourserver.example.com/rhel-7-server-extras-rpms/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```
The following articles explain how you can create a local mirror of the Red Hat repositories and how to share them.
https://access.redhat.com/solutions/23016  
https://access.redhat.com/solutions/7227

Before converting the VM to a template, you will need to setup up access for the Ansible host to configure the individual VMs. This is explained in the next section. 










## Create the Ansible node

In addition to the VM Template, we need another Virtual Machine where Ansible will be installed. This node will act as the driver to automate the provisioning of the environment and it is essential that it is properly installed. The steps are as follows.

1. Create a Virtual Machine and install your preferred OS (in this example, and for the sake of simplicity, RHEL7 will be used). The rest of the instructions assume that, if you use a different OS, you understand the possible differences in syntax for the provided commands. If you use RHEL 7, select **Infrastructure Server** as the base environment and the **Guests Agents** add-on during the installation.
2. Log in the root account and create an SSH key pair. Do not protect the key with a passphrase (unless you want to use ssh-agent).  
```
# ssh-keygen
```
3. Configure the following yum repositories, rhel-7-server-rpms and rhel-7-server-extras-rpms as explained in the previous section.
4. Configure the EPEL repository. See here for more information: http://fedoraproject.org/wiki/EPEL. Note that ```yum-config-manager``` comes with the Infrastructure Server base environment, if you did not select this environment you will have to install the ```yum-utils``` package.
```
# rpm -ivh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
# yum-config-manager --enable rhel-7-server-extras-rpms	
```
5. Install Ansible. Please note that version 2.2 is the minimum required. The instructions on how to install Ansible are described in its official website: http://docs.ansible.com/ansible/intro_installation.html
```
# yum install ansible
```
6. Make a list of all the hostnames and IPs that will be in your system and update your ```/etc/hosts``` file accordingly. This includes your UCP nodes, DTR nodes, worker nodes, NFS server, logger server and load balancers.
7. Install the following packages which are a mandatory requirement for the playbooks to function as expected. (Update pip if requested).
```
# yum install python-pyvmomi python-netaddr python2-jmespath python-pip gcc python-devel openssl-devel git
# pip install --upgrade pip
# pip install cryptography
# pip install pysphere
```
8. Copy your SSH public key to the VM Template so that, in the future, your Ansible node can SSH without the need of a password to all the Virtual Machines created from the VM Template.
```
# ssh-copy-id root@<VM_Template>
```

Please note that in both the Ansible node and the VM Template you might need to configure the network so one node can reach the other. Instructions for this step have been omitted since it is a basic step and could vary depending on the user’s environment.







## Finalize the template

Now that the VM Template has the public key of the Ansible node, we’re ready to convert this VM to a VM Template. Perform the following steps in the VM Template to finalize its creation:

1. Clean up the template by running the following commands:

```
# rm /etc/ssh/ssh_host_*
# history -c
```

2. Shut down the VM

```# shutdown –h now```

3. Once the Virtual Machine is ready and turned off, convert it to a template as shown in Figure 4.
![Convert to template][converttotemplate]  
**Figure 4** Convert to template  

This completes the creation of the VM Template.







## Prepare your Ansible configuration

On the Ansible node, retrieve the latest version of the playbooks using git.
```
# git clone https://github.com/HewlettPackard/Docker-SimpliVity

```

You now need to prepare the configuration to match your own environment, prior to deploying Docker EE and the rest of the nodes. To do so, you will need to edit and modify three different files:



- ```vm_hosts``` (the inventory file)
- ```group_vars/vars``` (the group variables file)
- ```group_vars/vault``` (the encrypted group variable file)





## Editing the inventory

Change to the directory that you previously cloned using git and edit the ```vm\_hosts``` file.

The nodes inside the inventory are organized in groups. The groups are defined by brackets and the group names are static so they must not be changed. Anything else (hostnames, specifications, IP addresses…) are meant to be amended to match the user needs. The groups are as follows:

- [ucp\_main]: A group containing one single node which will be the main UCP node and swarm leader. Do not add more than one node under this group.
- [ucp]: A group containing all the UCP nodes, including the main UCP node. Typically you should have either 3 or 5 nodes under this group.
- [dtr\_main]: A group containing one single node which will be the first DTR node to be installed. Do not add more than one node under this group.
- [dtr]: A group containing all the DTR nodes, including the main DTR node. Typically you should have either 3 or 5 nodes under this group.
- [worker]: A group containing all the worker nodes. Typically you should have either 3 or 5 nodes under this group.
- [ucp\_lb]: A group containing one single node which will be the load balancer for the UCP nodes. Do not add more than one node under this group.
- [dtr\_lb]: A group containing one single node which will be the load balancer for the DTR nodes. Do not add more than one node under this group.
- [worker\_lb]: A group containing one single node which will be the load balancer for the worker nodes. Do not add more than one node under this group.
- [lbs]: A group containing all the load balances. This group will have 3 nodes, also defined in the three groups above.
- [nfs]: A group containing one single node which will be the NFS node. Do not add more than one node under this group.
- [logger]: A group containing one single node which will be the logger node. Do not add more than one node under this group.
- [local]: A group containing the local Ansible host. It contains an entry that should not be modified.

There are also a few special groups:

- [docker:children]: A group of groups including all the nodes where Docker will be installed.
- [vms:children]: A group of groups including all the Virtual Machines involved apart from the local host.

Finally, you will find some variables defined for each group:

- [ucp:vars]: A set of variables defined for all nodes in the [```ucp```] group.
- [dtr:vars]: A set of variables defined for all nodes in the [```dtr```] group.
- [worker:vars]: A set of variables defined for all nodes in the [```worker```] group.
- [lbs:vars]: A set of variables defined for all nodes in the [```lbs```] group.
- [nfs:vars]: A set of variables defined for all nodes in the [```nfs```] group.
- [logger:vars]: A set of variables defined for all nodes in the [```logger```] group.

If you wish to configure your nodes with different specifications rather than the ones defined by the group, it is possible to declare the same variables at the node level, overriding the group value. For instance, you could have one of your workers with higher specifications by doing:

```
[worker]
worker01 ip_addr='10.0.0.10/16' esxi_host='esxi1.domain.local'
worker02 ip_addr='10.0.0.11/16' esxi_host='esxi1.domain.local'
worker03 ip_addr='10.0.0.12/16' esxi_host='esxi1.domain.local' cpus='16' ram'32768'
[worker:vars]
cpus='4'
ram='16384'
disk2_size='200'
node_policy='bronze'
```

In  the example above, the ```worker03``` node would have 4 times more CPU and double the RAM compared to the rest of worker nodes.

The different variables you can use are as described in Table 4 below. They are all mandatory unless if specified otherwise.

**Table 4.** Variables

| Variable     | Scope      | Description                              |
| ------------ | ---------- | ---------------------------------------- |
| ip\_addr     | Node       | IP address in CIDR format to be given to a node |
| esxi\_host   | Node       | ESXi host where the node will be deployed. If the cluster is configured with DRS, this option will be overriden |
| cpus         | Node/Group | Number of CPUs to assign to a VM or a group of VMs |
| ram          | Node/Group | Amount of RAM in MB to assign to a VM or a group of VMs |
| disk2\_usage | Node/Group | Size of the second disk in GB to attach to a VM or a group of VMs. This variable is only mandatory on Docker nodes (UCP, DTR, worker) and NFS node. It is not required for the logger node or the load balancers. |
| node\_policy | Node/Group | Simplivity backup policy to assign to a VM or a group of VMs. The name has to match one of the backup policies defined in the ```group_vars/vars``` file described in the next section. |





## Editing the group variables

Once the inventory is ready, the next step is to modify the group variables to match your environment. To do so, you need to edit the file ```group_vars/vars``` under the cloned directory containing the playbooks. The variables here can be defined in any order but for the sake of clarity they have been divided into sections.

### VMware configuration

All VMware-related variables are mandatory and are described in Table 5.

**Table 5. VMware variables**

| Variable                 | Description                              |
| ------------------------ | ---------------------------------------- |
| vcenter\_hostname        | IP or hostname of the vCenter appliance  |
| vcenter\_username        | Username to log in to the vCenter appliance. It might include a domain, for example, '[administrator@vsphere.local](mailto:administrator@vsphere.local)'. Note: The corresponding password is stored in a separate file (```group_vars/vault```) with the variable named ```vcenter_password```. |
|vcenter_validate_certs    | ‘no’ |
| datacenter               | Name of the datacenter where the environment will be provisioned |
| vm\_username             | Username to log into the VMs. It needs to match the one from the VM Template, so unless you have created an user, you must use 'root'. Note: The corresponding password is stored in a separate file (```group_vars/vault```) with the variable named ```vm_password```.  |
| vm\_template             | Name of the VM Template to be used. Note that this is the name from a vCenter perspective, not the hostname |
| folder\_name             | vCenter folder to deploy the VMs. If you do not wish to deploy in a particular folder, the value should be ```/```. **Note**: If you want to deploy in a specific folder, you need to create this folder in the inventory of the selected datacenter before starting the deployment. |
| datastores               | List of datastores to be used, in list format, i.e. ['```Datastore1```','```Datastore2```'...]. Please note that from a Simplivity perspective, it is best practice to use just one Datastore. Using more than one will not provide any advantages in terms of reliability and will add additional complexity. This datastore must exist before you run the playbooks. |
| disk2                    | UNIX name of the second disk for the Docker VMs. Typically ```/dev/sdb``` |
| disk2\_part              | UNIX name of the partition of the second disk for the Docker VMs. Typically ```/dev/sdb1``` |
| vsphere\_plugin\_version | Version of the vSphere plugin for Docker. The default is ```0.13``` which is the latest version at the time of writing this document. |



### Simplivity configuration

All SimpliVity variables are mandatory and are described in Table 6.

**Table 6.** SimpliVity variables

<table>
  <tr>
    <th>Variable</th>
    <th>Description</th>
  </tr>
  <tr>
    <td>simplivity_username</td>
	<td>Username to log in to the Simplivity Omnistack appliances. It might include a domain, for example, <a href="mailto:administrator@vsphere.local">administrator@vsphere.local</a>. Note: The corresponding password is stored in a separate file (<code>group_vars/vault</code>) with the variable named <code>simplivity_password</code>.</td>
  </tr>
  <tr>
    <td>omnistack_ovc</td>
	<td>List of Omnistack hosts to be used, in list format, i.e. <code>[‘omni1.local’,’onmi2.local’...]</code>. If your OmniStack virtual machines do not have their names registered in DNS, you can use their IP addresses.  
	</td>
  </tr>
    <td>rest_api_pause</td>
	<td>TODO: Default value is 10.
	</td>
  </tr>	
   <tr>
	<td>backup_policies </td>
	<td>
	List of dictionaries containing the different backup policies to be used along with the scheduling information. Any number of backup policies can be created and they need to match the <code>node_policy</code> variables defined in the inventory. Times are indicated in minutes.  All month calculations use a 30-day month. All year calculations use a 365-day year. The format is as follows:
<pre>
backup_policies:
 - name: daily'   
   days: 'All'   
   start_time: '11:30'   
   frequency: '1440'   
   retention: '10080' 
 - name: 'hourly'   
   days: 'All'   
   start_time: '00:00'   
   frequency: '60'   
   retention: '2880'
</pre>	
	
  </tr>
  <tr>
  <td>dummy_vm_prefix</td>
  <td>In order to be able to backup the Docker volumes, a number of "dummy" VMs need to spin up. This variable will set a recognizable prefix for them. </td>
  </tr>
  <tr>
	<td>docker_volumes_policy</td>
	<td>Backup policy to use for the Docker Volumes.</td>
  </tr>
</table>





### Networking configuration

All network-related variables are mandatory and are described in Table 7.

**Table 7.** Network variables

| Variable     | Description                              |
| ------------ | ---------------------------------------- |
| nic\_name    | Name of the device, for RHEL this is typically ```ens192``` and it is recommended to leave it as is. |
| gateway      | IP address of the gateway to be used     |
| dns          | List of DNS servers to be used, in list format, i.e. ['```10.10.173.1```','```10.10.173.2```'...] |
| domain\_name | Domain name for your Virtual Machines    |
| ntp\_server  | List of NTP servers to be used, in list format, i.e. ['```1.2.3.4```','```0.us.pool.net.org```'...] |




### Docker configuration

All Docker-related variables are mandatory and are described in Table 8.

| Variable      | Description                              |
| ------------- | ---------------------------------------- |
| docker_ee_url | Your ```docker_ee_url``` should be kept secret and you should define it in ```group_vars/vault```. The value for ```docker_ee_url``` is the URL documented at the following address: https://docs.docker.com/engine/installation/linux/docker-ee/rhel/. |
| rhel_version  | Version of your RHEL OS, such as  ```7.3```. The playbooks were tested with RHEL 7.3. and RHEL 7.4.        |
| dtr_version   | Version of the Docker DTR you wish to install. You can use a numeric version or ```latest``` for the most recent one. The playbooks where tested with 2.3.3. |
| ucp_version   | Version of the Docker UCP you wish to install. You can use a numeric version or ```latest``` for the most recent one. The playbooks were tested with UCP 2.2.3. |
| images_folder | Directory in the NFS server that will be mounted in the DTR nodes and that will host your Docker images. |
| license_file  | Full path to your Docker license file (it should be stored in your Ansible host). |
| ucp_username  | Username of the administrator user for UCP and DTR, typically ```admin```. Note: The corresponding password is stored in a separate file (```group_vars/vault```) with the variable named ```ucp_password```.|




### Monitoring configuration

All Monitoring-related variables are described in Table 9. The variables determine the versions of various monitoring software tools that are used and it is recommended that the values given below are used. 

**Table 9.** Monitoring variables

| Variable                | Description                              |
| ----------------------- | ---------------------------------------- |
| cadvisor\_version       | ```v0.25.0``` |
| node\_exporter\_version | ```v1.14.0``` |
| prometheus\_version     | ```v1.7.1``` |
| grafana\_version        | ```4.4.3``` |
| prom_persistent_vol_name | The name of the volume which will be used to store the monitoring data. The volume is created using the vsphere docker volume plugin. |
| prom_persistent_vol_size | The size of the volume which will hold the monitoring data. The exact syntax is dictated by the vSphere Docker Volume plugin. The default value is 10GB. |

### Logspout configuration

All Logspout-related variables are described in Table 10.

**Table 10.** Logspout variables

| Variable                | Description                              |
| ----------------------- | ---------------------------------------- |
| logspout_version       | ```‘latest’``` |


### Environment configuration

All Environment-related variables should be here. All of them are described in the Table 7 below.

| Variable | Description                              |
| -------- | ---------------------------------------- |
| env      | Dictionary containing all environment variables. It contains three entries described below. Please leave empty the proxy related settings if not required: <ul><li>http\_proxy: HTTP proxy URL, i.e. 'http://15.184.4.2:8080'. This variable defines the HTTP proxy url if your environment is behind a proxy.</li><li>https\_proxy: HTTP proxy URL, i.e. 'http://15.184.4.2:8080'. This variable defines the HTTPS proxy url if your environment is behind a proxy.</li><li>no\_proxy: List of hostnames or IPs that don't require proxy, i.e. 'localhost,127.0.0.1,.cloudra.local,10.10.174.'</li></ul> |

## Editing the vault

Once your group variables file is ready, the next step is to create a vault file to match your environment. The vault file is essentially the same thing than the group variables but it will contain all sensitive variables and will be encrypted.

To create a vault we'll create a new file group\_vars/vault and we'll add the following entries:

```
---
vcenter_password: 'xxx'
docker_ee_url: 'yoururl'
vm_password: 'xxx'
simplivity_password: 'xxx'
ucp_password: 'xxx'
rhn_orgid: 'Red Hat Organization ID'
rhn_key: 'Red Hat Activation Key'
```


```rhn_orgid``` and ```rhn_key``` are the credentials needed to subscribe the virtual machines with Red Hat Customer Portal. For more info regarding activation keys see the following URL: https://access.redhat.com/articles/1378093

To encrypt the vault you need to run the following command:

```# ansible-vault encrypt group_vars/vault```

You will be prompted for a password that will decrypt the vault when required. You can update the values in your vault by running:
```
# ansible-vault edit group_vars/vault
```

For Ansible to be able to read the vault, you need to specify a file where the password is stored, for instance in a file called ```.vault_pass```. Once the file is created, take the following precautions to avoid illegitimate access to this file:

1. Change the permissions so only ```root``` can read it using  ```# chmod 600 .vault_pass```
	
2. Add the file to your ```.gitignore``` file if you're pushing the set of playbooks to a git repository.




# Running the playbooks

At this point, the system is ready to be deployed. Go to the root folder and run the following command:

```# ansible-playbook -i vm_hosts site.yml --vault-password-file .vault_pass```

The playbooks should run for 25-35 minutes depending on your server specifications and the size of your environment.


## Post Deployment

The playbooks are meant to deploy a new environment. You should only use them for deployment purposes. 
















# Overview of the playbooks



## Create virtual machines

The playbook [playbooks/create\_vms.yml][create_vms] will create all the necessary Virtual Machines for the environment from the VM Template defined in the vm_template variable.

## Configure network settings
The playbook [config\_networking.yml][config_networking] will configure the network settings in all the Virtual Machines. 

## Distribute public keys
The playbook [distribute\_keys.yml][distribute_keys] distributes public keys between all nodes, to allow each node to password-less login to every other node. As this is not essential and can be regarded as a security risk (a worker node probably should not be able to log in to a UCP node, for instance), this playbook is commented out in site.yml by default and must be explicitly uncommented to enable this functionality.

## Register the VMs with Red Hat
The playbook [config\_subscription.yml][config_subscription] registers and subscribes all virtual machines to the Red Hat Customer Portal. This is only needed if you pull packages from Red Hat.

## Install HAProxy
The playbook [install\_haproxy.yml][install_haproxy] installs and configures the HAProxy package in the load balancer nodes. HAProxy is the chosen tool to implement load balancing between UCP nodes, DTR nodes and worker nodes.

## Install NTP
The playbook [install\_ntp.yml][install_ntp] installs and configures the NTP package in all Virtual Machines in order to have a synchronized clock across the environment. It will use the server or servers specified in the ntp_servers variable in the group variables file.

## Install Docker Enterprise Edition
The playbook [install\_docker.yml][install_docker] installs Docker along with all its dependencies.


## Install rsyslog
The playbook [install_rsyslog.yml][install_rsyslog] installs and configures rsyslog in the logger node and in all Docker nodes. The logger node will be configured to receive all syslogs on port 514 and the Docker nodes will be configured to send all logs (including container logs) to the logger node.


## Configure Docker LVs
The playbook [config_docker_lvs.yml][config_docker_lvs] performs a set of operations on the Docker nodes in order to create a partition on the second disk and carry out the LVM configuration, required for a sound Docker installation.


## Docker post-install configuration
The playbook [docker_post_config.yml][docker_post_config] performs a variety of tasks to complete the installation of the Docker environment.



## Install NFS server 
The playbook [install_nfs_server.yml][install_nfs_server] installs and configures an NFS server on the NFS node.



## Install NFS clients
The playbook [install_nfs_clients.yml][install_nfs_clients] installs the required packages on the DTR nodes to be able to mount an NFS share.


## Install and configure Docker UCP nodes
The playbook [install_ucp_nodes.yml][install_ucp_nodes] installs and configures the Docker UCP nodes defined in the inventory.



## Install and configure DTR nodes
The playbook [install_dtr_nodes.yml][install_dtr_nodes] installs and configures the Docker DTR nodes defined in the inventory. Note that serialization is set to 1 in this playbook as two concurrent installations of DTR may in some cases be assigned the same replica ID.

## Install worker nodes
The playbook [install_worker_nodes.yml][install_worker_nodes] installs and configures the Docker Worker nodes defined in the inventory.


## Configuring monitoring
The playbook [config_monitoring.yml][config_monitoring] configures a monitoring system for the Docker environment by making use of Grafana, Prometheus, cAdvisor and node-exporter Docker containers.

**Note:** If you have your own monitoring solution, you can comment out the corresponding line in the main playbook ```site.yml```
```
#- include: playbooks/config_monitoring.yml
```








# Accessing the UCP UI
Once the playbooks have run and completed successfully, the Docker UCP UI should be available by browsing to the UCP load balancer or any of the nodes via HTTPS. The authentication screen will appear. Enter your credentials and the dashboard will be displayed. You should see all the nodes information in your Docker environment by clicking on Nodes. By looking into the services you should see the monitoring services that were installed during the playbooks execution:

# Accessing the DTR UI
The Docker DTR UI should be available by browsing to the DTR load balancer or any of the nodes via HTTPS. The authentication screen will appear. Enter your UCP credentials and you should see the empty list of repositories. If you navigate to `Settings > Security`, you should see the Image Scanning feature already enabled (note that you need an Advanced license to have access to this feature).





# Security considerations
In addition to having all logs centralized in an unique place and the image scanning feature enabled, there are another few guidelines that should be followed in order to keep your Docker environment as secure as possible.

## Securing the daemon socket
The default is to create a non-networked socket to communicate with the daemon, but you could alternatively communicate using TLS. This will require you to create a CA and server and client keys with OpenSSL. The whole procedure is explained in detail here: [https://docs.docker.com/engine/security/https/](https://docs.docker.com/engine/security/https/)

## Start with known images
Developers can pick base images from the Docker Hub to start with. Although this allows them a great deal of freedom, some caution is needed. Images from Docker Hub are uncurated and may contain vulnerabilities. Use store images where possible. These are curated images. Use small base images to reduce the surface area.

## Enable Docker Content Trust
Notary/Docker Content Trust is a tool for publishing and managing trusted collections of content. Publishers can digitally sign collections and consumers can verify integrity and origin of content. This ability is built on a straightforward key management and signing interface to create signed collections and configure trusted publishers. More information about Notary can be found here: [https://success.docker.com/Architecture/Docker\_Reference\_Architecture%3A\_Securing\_Docker\_EE\_and\_Security\_Best\_Practices#dtr-notary](https://success.docker.com/Architecture/Docker_Reference_Architecture%3A_Securing_Docker_EE_and_Security_Best_Practices#dtr-notary)

## Prevent tags from being overwritten
By default, users with access to push to a repository, can push the same tag multiple times to the same repository. As an example, a user pushes an image to library/wordpress:latest, and later another user can push the image with exactly the same name but different functionality. This might make it difficult to trace back the image to the build that generated it.

To prevent this from happening you can configure a repository to be immutable. Once you push a tag, DTR won't anyone else to push another tag with the same name.

More information about immutable tags can be found here: [https://beta.docs.docker.com/datacenter/dtr/2.3/guides/user/manage-images/prevent-tags-from-being-overwritten/](https://beta.docs.docker.com/datacenter/dtr/2.3/guides/user/manage-images/prevent-tags-from-being-overwritten/)

## Use secrets
Use secrets to pass credentials to a container. Never pass as an environment variable which is clear

## Isolate swarm nodes to a specific team
With Docker EE Advanced, you can enable physical isolation of resources by organizing nodes into collections and granting Scheduler access for different users. To control access to nodes, move them to dedicated collections where you can grant access to specific users, teams, and organizations.

More information about this subject can be found here: [https://beta.docs.docker.com/datacenter/ucp/2.2/guides/admin/manage-users/isolate-nodes-between-teams/](https://beta.docs.docker.com/datacenter/ucp/2.2/guides/admin/manage-users/isolate-nodes-between-teams/)

## Docker Bench for Security
The Docker Bench for Security is a script that checks for dozens of common best-practices around deploying Docker containers in production. The tests are all automated, and are inspired by the CIS Docker Community Edition Benchmark v1.1.0.

The Docker Bench for Security should be run on a regular basis to make sure that our system is as secure as we'd expect it to be.

More information about this tool plus the files to run it can be found in its Github repository: [https://github.com/docker/docker-bench-security](https://github.com/docker/docker-bench-security)

# Contact
Please get in touch via Github if you have any questions.

# Demo
A much briefer video with a quick demo can be found here: https://vimeo.com/229389079


[architecture]: </images/architecture.png> "Figure 1. Solution Architecture"
[provisioning]: </images/provisioning.png> "Provisioning Steps"
[createnewvm]: </images/createnewvirtualmachine.png> "Figure 2. Create New Virtual Machine"
[vmnamelocation]: </images/vmnamelocation.png> "Figure 3. Specify name and location for the virtual machine" 
[converttotemplate]: </images/converttotemplate.png> "Figure 4. Convert to template"


[create_vms]: </playbooks/create_vms.yml>
[config_networking]: </playbooks/config_networking.yml>
[distribute_keys]: </playbooks/distribute_keys.yml>
[config_subscription]: </playbooks/config_subscription.yml>
[install_haproxy]: </playbooks/install_haproxy.yml>
[install_ntp]: </playbooks/install_ntp.yml>
[install_docker]: </playbooks/install_docker.yml>
[install_rsyslog]: </playbooks/install_rsyslog.yml>
[config_docker_lvs]: </playbooks/config_docker_lvs.yml>
[docker_post_config]: </playbooks/docker_post_config.yml>
[install_nfs_server]: </playbooks/install_nfs_server.yml>
[install_nfs_clients]: </playbooks/install_nfs_clients.yml>
[install_ucp_nodes]: </install_ucp_nodes.yml>
[install_dtr_nodes]: </install_dtr_nodes.yml>
[install_worker_nodes]: </install_worker_nodes.yml>
[config_monitoring]: </config_monitoring.yml>

