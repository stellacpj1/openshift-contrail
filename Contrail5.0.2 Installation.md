Contrail 5.0.2 Installation 
======================
## Server Requirements
<b>Contrail Command</b>:
- A VM or physical server with:
    - 8 vCPUs
	- 64 GB RAM
	- 300 GB disk out of which 256 GB is allocated to /root directory.
- Internet access to and from the physical server, hereafter referred to as the Contrail Command server.
- (Recommended) x86 server with CentOS 7.5 as the base OS to install Contrail Command.

<b>Contrail Cluster</b>:
- Contrail Controller - 8 VCPU, 64G memory, 300G storage.
- Openstack Controller - 4 vCPU , 32G Memory, 100G Storage.
- CSN - 4 vCPU, 16G Memory, 100G Storage.
- Compute nodes - depends on the workloads.


## Contrail Command Server Installation
#### 0. Prerequisite
docker-py is obsolete in Contrail Release 5.0.2. You must remove docker-py and docker Python packages from all the nodes where you want to install the Contrail Command UI.
- `pip uninstall docker-py docker`

#### 1. Install Docker on the Contrail Command server 
These packages are necessary to automate the deployment of Contrail Command software:
- `yum install -y yum-utils device-mapper-persistent-data lvm2`
- `yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`
- `yum install -y docker-ce`
- `systemctl start docker`

#### 2. Download the contrail-command-deployer Docker container image
This docker container image is used to deploy contrail-command (contrail_command, contrail_mysql containers) from hub.juniper.net. Allow Docker to connect to the private secure registry:  
- `docker login hub.juniper.net --username <container_registry_username> --password <container_registry_password>`

Pull Contrail-Command-Deployer Container from the private secure registry.
- `docker pull hub.juniper.net/contrail/contrail-command-deployer:<container_tag>` # Example, for container_tag: 5.0.1-0.214, use the following command:
docker pull hub.juniper.net/contrail/contrail-command-deployer:5.0.1-0.214

#### 3. Create the input configuration command_servers.yml file:
Refer sample config file [command_server.yaml.public_container_registry](https://git.juniper.net/stella/cn5.0.1_installation/blob/master/command_servers_public.yaml) or [command_server.yaml.internal_container_registry](https://git.juniper.net/stella/cn5.0.1_installation/blob/master/command_servers_internal.yaml)

- `vi command_servers.yml`
    -   Key parameters to update for server:
        - <b>ip</b>
        - <b>ssh_user</b>
        - <b>ssh_pass</b>
        - <b>sudo_pass</b>
    - Key parameters to update for contrail_regitry:
        -  <b>registry_insecure<b>:  `false` if you use a secure public container registry (e.g hub.juniper.net); `true` if you use an internal container registry (e.g repo.englab.juniper.net)
        -  <b>container_registry</b>:  `hub.juniper.net/contrail`(secure public container registry); `repo.englab.juniper.net:5010`(internal container registry)
        -  <b>container_tag</b>: 5.0.1-0.214
        -  <b>container_registry_username</b>: uncomment this if you choose to use an internal container registry, else input your username for public registry. (Write an email to alias "contrail-registry" to apply for your account for contrail public registry if you have not applied yet.)
        -  <b>container_registry_password</b>: uncomment this if you choose to use an internal container registry, else input your password for public registry.

#### 4. Start the Contrail_Command_Deployer container to deploy the Contrail-Command server:   
- `docker run -t --net host -v ABSOLUTE_PATH_TO_COMMAND_SERVERS_FILE:/command_servers.yml -d --privileged --name contrail_command_deployer hub.juniper.net/contrail/contrail-command-deployer:<container_tag>`

<b>ABSOLUTE_PATH_TO_COMMAND_SERVERS_FILE</b>—path to the command_servers.yml file that you created in step 3.

Example, for container_tag: 5.0.1-0.214, use the following command:

`docker run -t --net host -v /root/command_servers.yml:/command_servers.yml -d --privileged --name contrail_command_deployer hub.juniper.net/contrail/contrail-command-deployer:5.0.1-0.214`

The contrail_command and contrail_mysq Contrail Command containers are deployed.
#### 5. Login Contrail Command:
-  Once the playbook completes, log into Contrail Command using https://Contrail-Command-Server-IP-Address:9091. Default username is <b>admin</b> and password is <b>contrail123</b>.
-  The first time to login to Contrail Command:
<div align=center><img width="720" height="400" src="imgs/contrail_command.png"/></div>

### 6. Trouble-shooting
Failed to deploy the Contrail-Command server due to failure to launch container "mysql".
<div align=center><img width="720" height="600" src="imgs/trouble1.png"/></div>
Solution:
<p>
- `cd contrail-command-deployer/playbooks/roles/launch_containers/tasks`
- `vi launch_containers.yml`
- change the <b>delay</b> time for task "wait for mysql to be up" from 5 to 15.

```
- name: wait for mysql to be up
  shell: "docker exec {{ database_container_name }} mysql -uroot -pcontrail123 -e 'show status'"
  register: result
  until: result.stderr.find("Can't connect to MySQL") == -1
  retries: 10
  delay: 15   # change from '5' to '15' to give container more time to become the up status.
  when:
    - contrail_config.database.type == 'mysql'
```


## Contrail Cluster Provison
Contrail Cluster is an OpenStack orchestration coupled with Contrail Networking plugin. This example topic describes how to use Contrail Command to deploy a Contrail Cluster.
#### 0. Prerequisites
- Please make sure that all the servers in your Contrail Cluster have <b>root</b> user configed with password <b>c0ntrail123</b>.
- Please make sure that all the server in your Contrail Cluster with <b>kernel version = kernel-3.10.0-862.3.2.el7.x86_64</b>, else you need to upgrade the kernel version.

```
[root@worker3 ~]# uname -r
3.10.0-862.3.2.el7.x86_64
```
- [<b>For VMware ESXi</b>] Please enable <b>Hardware virtualization</b> for your compute node server:
<div align=center><img width="600" height="400" src="imgs/tip_compute_node.png"/></div>

- [<b>For VMware ESXi</b>] Please modify the nova.conf for your compute node server, and then restart your nova_compute container on your compute node:

     - <b>Trouble:</b> `**** Nova console: stuck in ""Booting from Hard Disk ... GRUB" ****`
       <div align=center><img width="620" height="300" src="imgs/trouble-qemu.png"/></div>
     - <b>Solution:</b>
        - `$ vi /etc/kolla/nova-compute/nova.conf`
        
        ```
        [libvirt]
        hw_machine_type = x86_64=rhel6.5.0    # please add this line;
        ```
        - `$docker restart nova_compute`
 
#### 1. Add physical servers, two ways:
<div align=center><img width="720" height="400" src="imgs/step1.png"/></div>
  - First way: add servers one by one manually;
    <div align=center><img width="720" height="400" src="imgs/step1.1.png"/></div>
  - Secend way: upload a CSV file which contains the details of each server; Refer to [CSV_example file](https://git.juniper.net/stella/cn5.0.1_installation/blob/master/Servers-Bulk-example.csv).
    <div align=center><img width="720" height="400" src="imgs/step1.2.png"/></div>

#### 2. Create a cluster.
- If <b>Container registry</b> = hub.juniper.net . This registry is secure. Unselect the <b>Insecure</b> box. Also, <b>Contrail version</b> = Contrail_GA_version (refer to [Access to Contrail Registry](https://www.juniper.net/documentation/en_US/contrail5.0/information-products/topic-collections/release-notes/readme-contrail-50.pdf)
to choose the right version & tag)
- Default <b>vRouter Gateway</b> = Default gateway for the compute nodes. If any one of the compute nodes has a different default gateway than the one provided here, enter that gateway in [step5](https://git.juniper.net/stella/cn5.0.1_installation/tree/master#5-select-the-compute-nodes).
- <b>Encapsulation Priority</b> must be set as VXLAN, MPLSoUDP, MPLSoGRE.
- Select <b>Show Advanced Options</b>.
    - Set <b>CONTROL_NODES</b> = Data interface of the Contrail controller.
    - (optional) Set <b>CSN_NODES</b> = Data interface of the Contal Service Node.
<div align=center><img width="720" height="400" src="imgs/step2.png"/></div>

#### 3.Select the contrail-control node.
<div align=center><img width="720" height="400" src="imgs/step3.png"/></div>
#### 4. Select the orchestration node.
- Select <b>Show Advanced</b> option to customize your deployment. Control & Data Network Virtual IP address is an internal VIP. Management Network Virtual IP address is an external VIP.
- If ironic is not needed and if you are not going to use Life Cycle Management in Contrail Command then you need not deploy Ironic. In Contrail Command, key-value pair handle parameters that need not be enables or need to be disabled in Kolla Global.
- Set the following in Kolla Globals:
   - ironic_enable = Yes
   - upgrade_kernel = yes (Compute node kernel version = kernel-3.10.0-862.3.2.el7.x86_64; Else kernel upgrade is required.)
<div align=center><img width="720" height="400" src="imgs/step4.png"/></div>
<div align=center><img width="720" height="400" src="imgs/step4.2.png"/></div>

#### 5. Select the Compute Nodes.
For each compute node, enter the gateway, if it is different from what was added in the global parameters in [step2](https://git.juniper.net/stella/cn5.0.1_installation/tree/master#2-create-a-cluster).
<div align=center><img width="720" height="400" src="imgs/step5.png"/></div>
#### 6. Select the Contrail Service Node.
<div align=center><img width="720" height="400" src="imgs/step6.png"/></div>
#### 7. Verify the summary of your cluster configuration and click Provision.
<div align=center><img width="720" height="400" src="imgs/step7.png"/></div>
<div align=center><img width="720" height="400" src="imgs/step7.2.png"/></div>
#### 8. Provisioning 
<div align=center><img width="720" height="400" src="imgs/step8.png"/></div>
<b>Check provison info of Conrtail Cluster:</b> 
<p>
Login to Contrail Command server, then run `docker exec -it contrail_command bash` to login to contrail_command container, after successfully login to contrail_command container:
-  Check <b>instances.yml</b> used for clsuter provisioning: refer to [install_example.yaml](https://git.juniper.net/stella/cn5.0.1_installation/blob/master/instances_example.yaml).
     -  `cat /var/tmp/contrail_cluster/<UUID>/instances.yml`
-  Check <b>provison logs</b> for monitoring the installation process of Contrail Cluster: 
     - `tail -f /var/log/ansible.log`


#### 9. Check Contrail Command Web GUI after successfully provision the Contrail Cluster.
<div align=center><img width="720" height="400" src="imgs/provision_result.png"/></div>
