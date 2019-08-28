# 数讯Contrial 5.0.3部署手册

## 1. 配置基础架构

整体架构请参考：

- `数讯部署方案`
- `SDS生产环境部署信息`

### 1.1 控制节点裸机关键配置

#### 1.1.1 网络配置

以c2b13-stack01节点为例，实际安装过程中使用nmtui命令进行配置。

**网桥配置**

```bash
bridge name     bridge id               STP enabled     interfaces
br-ctl          8000.040973c3b8c8       yes             eth1.702
                                                        vnet*
                                                        
br-data         8000.040973c3b8c2       yes             eth2.704
                                                        vnet*
                                                        
br-mgmt         8000.040973c3b8c0       yes             eth0
                                                        vnet*
                                                
br-nas          8000.040973c3b8ca       yes             eth3
                                                        vnet*         
```

**网卡配置**

br-mgmt

```ini
STP=yes
BRIDGING_OPTS=priority=32768
TYPE=Bridge
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
IPADDR=192.168.184.13
PREFIX=24
GATEWAY=192.168.184.254
DNS1=221.114.220.18
DOMAIN=cloud.shuxun.net
DEFROUTE=yes
IPV4_FAILURE_FATAL=yes
IPV6INIT=no
NAME=br-mgmt
UUID=85c178a2-5897-4cf8-b469-c13ca774dd17
DEVICE=br-mgmt
ONBOOT=yes
```

eth0

```ini
TYPE=Ethernet
NAME=eth0
UUID=bdae4540-6dde-4851-8e56-6c7fc44e8855
DEVICE=eth0
ONBOOT=yes
BRIDGE=br-mgmt
BRIDGE_UUID=85c178a2-5897-4cf8-b469-c13ca774dd17
```

br-ctl

```ini
DEVICE=br-ctl
TYPE=Bridge
BOOTRPOTO=none
ONBOOT=yes
STP=yes
BRIDGING_OPTS=priority=32768
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
IPADDR=10.0.0.13
PREFIX=24
DEFROUTE=yes
IPV4_FAILURE_FATAL=yes
IPV6INIT=no
NAME=br-ctl
UUID=046c58dd-a31b-1253-7eb4-4967e46c0683
```

eth2.702

```ini
VLAN=yes
TYPE=Vlan
PHYSDEV=eth1
VLAN_ID=702
REORDER_HDR=yes
GVRP=no
MVRP=no
NAME=Vlan702
UUID=27c5825d-848b-44c8-81ce-4201d4579276
DEVICE=eth1.702
ONBOOT=yes
BRIDGE_UUID=046c58dd-a31b-1253-7eb4-4967e46c0683
BRIDGE=br-ctl
```

br-data

```ini
STP=yes
BRIDGING_OPTS=priority=32768
TYPE=Bridge
PROXY_METHOD=none
BROWSER_ONLY=no
DEFROUTE=yes
IPV4_FAILURE_FATAL=yes
IPV6INIT=no
NAME=br-data
UUID=d22666cc-c835-4e9d-841b-ebbc6be7f126
DEVICE=br-data
ONBOOT=yes
MTU=9000
```

* *请注意MTU值为9000*

```ini
VLAN=yes
TYPE=Vlan
PHYSDEV=eth2
VLAN_ID=704
REORDER_HDR=yes
GVRP=no
MVRP=no
MTU=9000
NAME=vlan704
UUID=a48394de-8a28-4c6d-8297-8a5430c9da2f
DEVICE=eth2.704
ONBOOT=yes
BRIDGE=br-data
BRIDGE_UUID=d22666cc-c835-4e9d-841b-ebbc6be7f126
```

- *请注意MTU值为9000*

br-nas

```ini
STP=yes
BRIDGING_OPTS=priority=32768
TYPE=Bridge
PROXY_METHOD=none
BROWSER_ONLY=no
IPV6INIT=no
NAME=br-nas
UUID=b5e93176-f636-42e0-b0a8-1b81631160a0
DEVICE=br-nas
ONBOOT=yes
BOOTPROTO=none
IPADDR=10.0.4.13
PREFIX=24
DEFROUTE=yes
IPV4_FAILURE_FATAL=yes
```

eth3

```ini
TYPE=Ethernet
NAME=eth3
UUID=0c799576-7196-46a6-bcb5-8d86f0d678eb
DEVICE=eth3
ONBOOT=yes
BRIDGE_UUID=b5e93176-f636-42e0-b0a8-1b81631160a0
BRIDGE=br-nas
```

#### 1.1.2 YUM源配置

/etc/yum.repos.d/local.repo

```
CentOS7.6-base]
name=CentOS7.6-base
baseurl=http://192.168.199.135/centos7.6/base/
gpgcheck=0
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[CentOS7.6-updates]
name=CentOS7.6-updates
baseurl=http://192.168.199.135/centos7.6/updates
gpgcheck=0
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[CentOS7.6-epel]
name=CentOS7.6-epel
baseurl=http://192.168.199.135/centos7.6/epel
gpgcheck=0
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[CentOS7.6-extras]
name=CentOS7.6-extras
baseurl=http://192.168.199.135/centos7.6/extras
gpgcheck=0
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[CentOS7.6-centosplus]
name=CentOS7.6-centosplus
baseurl=http://192.168.199.135/centos7.6/centosplus
gpgcheck=0
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[CentOS7.6-docker-ce-stable]
name=CentOS7.6-docker-ce-stable
baseurl=http://192.168.199.135/centos7.6/docker-ce-stable
gpgcheck=0
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
```

#### 1.1.3 安装Libvirt

```bash
yum install libvirt qemu-kvm virt-install -y
systemctl start libvirtd
systemctl enable libvirtd
virsh net-destroy default
virsh net-undefine default
```

### 1.2 控制平面虚拟机创建

c2b13-stack01:/root/ayang/create_vm.sh

```bash
#!/bin/bash

VM_NAME=c2b13-cbuilder.cloud.shuxun.net
MEM_SIZE=16384
VCPUS=8
DISK=/dev/mapper/mpathd
virt-install \
     --name ${VM_NAME} \
     --virt-type kvm \
     --memory ${MEM_SIZE} \
     --vcpus ${VCPUS} \
     --os-type linux \
     --location http://192.168.184.250/centos7.6/os/ \
     --disk path=${DISK},cache=none,format=raw \
     --network bridge=br-mgmt,model=virtio --network bridge=br-ctl,model=virtio --network bridge=br-data,model=virtio --network bridge=br-nas,model=virtio \
     --autostart \
     --graphics vnc,listen=0.0.0.0 \
     --console pty,target_type=serial \
     -x "console=ttyS0,115200n8 serial" \
     -x "ks=http://192.168.184.250/vm-ks.cfg"

VM_NAME=c2b13-cstack-ctl01.cloud.shuxun.net
MEM_SIZE=24576
VCPUS=12
DISK=/dev/mapper/mpathc
virt-install \
     --name ${VM_NAME} \
     --virt-type kvm \
     --memory ${MEM_SIZE} \
     --vcpus ${VCPUS} \
     --os-type linux \
     --location http://192.168.184.250/centos7.6/os/ \
     --disk path=${DISK},cache=none,format=raw \
     --network bridge=br-mgmt,model=virtio --network bridge=br-ctl,model=virtio --network bridge=br-data,model=virtio --network bridge=br-nas,model=virtio \
     --autostart \
     --graphics vnc,listen=0.0.0.0 \
     --console pty,target_type=serial \
     -x "console=ttyS0,115200n8 serial" \
     -x "ks=http://192.168.184.250/vm-ks.cfg"

VM_NAME=c2b13-cstack-nal01.cloud.shuxun.net
MEM_SIZE=32768
VCPUS=12
DISK=/dev/mapper/mpathe
virt-install \
     --name ${VM_NAME} \
     --virt-type kvm \
     --memory ${MEM_SIZE} \
     --vcpus ${VCPUS} \
     --os-type linux \
     --location http://192.168.184.250/centos7.6/os/ \
     --disk path=${DISK},cache=none,format=raw \
     --network bridge=br-mgmt,model=virtio --network bridge=br-ctl,model=virtio --network bridge=br-data,model=virtio --network bridge=br-nas,model=virtio \
     --autostart \
     --graphics vnc,listen=0.0.0.0 \
     --console pty,target_type=serial \
     -x "console=ttyS0,115200n8 serial" \
     -x "ks=http://192.168.184.250/vm-ks.cfg"
```

c2b13-stack01:/root/ayang/create_vm-csn-cmd.sh

```bash
#!/bin/bash

VM_NAME=c2b13-ccsn01.cloud.shuxun.net
MEM_SIZE=16384
VCPUS=4
DISK=/dev/mapper/mpatha
virt-install \
     --name ${VM_NAME} \
     --virt-type kvm \
     --memory ${MEM_SIZE} \
     --vcpus ${VCPUS} \
     --os-type linux \
     --location http://192.168.184.250/centos7.6/os/ \
     --disk path=${DISK},cache=none,format=raw \
     --network bridge=br-mgmt,model=virtio --network bridge=br-ctl,model=virtio --network bridge=br-data,model=virtio --network bridge=br-nas,model=virtio \
     --autostart \
     --graphics vnc,listen=0.0.0.0 \
     --console pty,target_type=serial \
     -x "console=ttyS0,115200n8 serial" \
     -x "ks=http://192.168.184.250/vm-ks.cfg"

VM_NAME=c2b13-ccmd01.cloud.shuxun.net
MEM_SIZE=65536
VCPUS=8
DISK=/dev/mapper/mpathb
virt-install \
     --name ${VM_NAME} \
     --virt-type kvm \
     --memory ${MEM_SIZE} \
     --vcpus ${VCPUS} \
     --os-type linux \
     --location http://192.168.184.250/centos7.6/os/ \
     --disk path=${DISK},cache=none,format=raw \
     --network bridge=br-mgmt,model=virtio --network bridge=br-ctl,model=virtio --network bridge=br-data,model=virtio --network bridge=br-nas,model=virtio \
     --autostart \
     --graphics vnc,listen=0.0.0.0 \
     --console pty,target_type=serial \
     -x "console=ttyS0,115200n8 serial" \
     -x "ks=http://192.168.184.250/vm-ks.cfg"
```

c2b14-cstack02:/root/ayang/create_vm.sh

```bash
#!/bin/bash

VM_NAME=c2b14-cstack-ctl02.cloud.shuxun.net
MEM_SIZE=24576
VCPUS=12
DISK=/dev/mapper/mpatha
virt-install \
     --name ${VM_NAME} \
     --virt-type kvm \
     --memory ${MEM_SIZE} \
     --vcpus ${VCPUS} \
     --os-type linux \
     --location http://192.168.184.250/centos7.6/os/ \
     --disk path=${DISK},cache=none,format=raw \
     --network bridge=br-mgmt,model=virtio --network bridge=br-ctl,model=virtio --network bridge=br-data,model=virtio --network bridge=br-nas,model=virtio \
     --autostart \
     --graphics vnc,listen=0.0.0.0 \
     --console pty,target_type=serial \
     -x "console=ttyS0,115200n8 serial" \
     -x "ks=http://192.168.184.250/vm-ks.cfg"

VM_NAME=c2b14-cstack-nal02.cloud.shuxun.net
MEM_SIZE=32768
VCPUS=12
DISK=/dev/mapper/mpathb
virt-install \
     --name ${VM_NAME} \
     --virt-type kvm \
     --memory ${MEM_SIZE} \
     --vcpus ${VCPUS} \
     --os-type linux \
     --location http://192.168.184.250/centos7.6/os/ \
     --disk path=${DISK},cache=none,format=raw \
     --network bridge=br-mgmt,model=virtio --network bridge=br-ctl,model=virtio --network bridge=br-data,model=virtio --network bridge=br-nas,model=virtio \
     --autostart \
     --graphics vnc,listen=0.0.0.0 \
     --console pty,target_type=serial \
     -x "console=ttyS0,115200n8 serial" \
     -x "ks=http://192.168.184.250/vm-ks.cfg"
```

c3b13-cstack03:/root/ayang/create_vm.sh

```
#!/bin/bash

VM_NAME=c3b13-cstack-ctl03.cloud.shuxun.net
MEM_SIZE=24576
VCPUS=12
DISK=/dev/mapper/mpathb
virt-install \
     --name ${VM_NAME} \
     --virt-type kvm \
     --memory ${MEM_SIZE} \
     --vcpus ${VCPUS} \
     --os-type linux \
     --location http://192.168.184.250/centos7.6/os/ \
     --disk path=${DISK},cache=none,format=raw \
     --network bridge=br-mgmt,model=virtio --network bridge=br-ctl,model=virtio --network bridge=br-data,model=virtio --network bridge=br-nas,model=virtio \
     --autostart \
     --graphics vnc,listen=0.0.0.0 \
     --console pty,target_type=serial \
     -x "console=ttyS0,115200n8 serial" \
     -x "ks=http://192.168.184.250/vm-ks.cfg"

VM_NAME=c3b13-cstack-nal03.cloud.shuxun.net
MEM_SIZE=32768
VCPUS=12
DISK=/dev/mapper/mpatha
virt-install \
     --name ${VM_NAME} \
     --virt-type kvm \
     --memory ${MEM_SIZE} \
     --vcpus ${VCPUS} \
     --os-type linux \
     --location http://192.168.184.250/centos7.6/os/ \
     --disk path=${DISK},cache=none,format=raw \
     --network bridge=br-mgmt,model=virtio --network bridge=br-ctl,model=virtio --network bridge=br-data,model=virtio --network bridge=br-nas,model=virtio \
     --autostart \
     --graphics vnc,listen=0.0.0.0 \
     --console pty,target_type=serial \
     -x "console=ttyS0,115200n8 serial" \
     -x "ks=http://192.168.184.250/vm-ks.cfg"
```

### 1.3 控制平面虚拟机基础配置

由于虚拟机数量较大，此处不详细描述配置，只说明关键配置。

#### 1.3.1 网络配置

- 参考规划`SDS生产环境部署信息`进行接口配置
- CSN节点的eth2网卡为应命名为vhost0

c2b13-ccsn01:/etc/sysconfig/network-scripts/ifcfg-eth2

```
# Generated by parse-kickstart
DEVICE=eth2
IPV6INIT=no
BOOTPROTO=none
UUID=32c65a57-71b9-408d-8f9a-3a28e212cb67
ONBOOT=yes
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
DEFROUTE=yes
IPV4_FAILURE_FATAL=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
NAME="System data"
IPADDR=10.0.2.202
PREFIX=24
NM_CONTROLLED=no
```

c2b13-ccsn01:/etc/sysconfig/network-scripts/ifcfg-vhost0

```bash
DEVICE=vhost0
BOOTPROTO=none
ONBOOT=yes
IPADDR=10.0.2.202
PREFIX=24
NETWORK=10.0.2.0
TYPE=kernel
NM_CONTROLLED=no
BIND_INT=eth2
MTU=9000
```

#### 1.3.2 主机路由配置

在以下节点增加如下路由配置。

- c2b13-ccsn01

/etc/sysconfig/network-scripts/route-vhost0

```bash
172.16.6.34/32 via 10.0.2.254 dev vhost0
172.16.6.36/32 via 10.0.2.254 dev vhost0
172.16.6.37/32 via 10.0.2.254 dev vhost0
```

- c2b13-cstack-ctl01
- c2b14-cstack-ctl02
- c3b13-cstack-ctl03

/etc/sysconfig/network-scripts/route-eth1

```
172.16.6.34/32 via 10.0.0.254 dev eth1
172.16.6.36/32 via 10.0.0.254 dev eth1
172.16.6.37/32 via 10.0.0.254 dev eth1
```

### 1.4 计算节点配置

由于计算节点数量较多，此处不详细描述配置，只说明关键配置。

#### 1.4.1 网络配置

参考规划`SDS生产环境部署信息`进行接口配置

相关配置可以参考1.1章节中的网卡配置

vhost0网卡配置参考如下：

/etc/sysconfig/network-scripts/ifcfg-eth2

```ini
# Generated by parse-kickstart
DEVICE=eth2
IPV6INIT=yes
BOOTPROTO=dhcp
UUID=3050e494-be9b-4e28-830d-5d4301f605eb
ONBOOT=no
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
#NAME="System eth2"
NAME="System eth2"
NM_CONTROLLED=no
MTU="9000"
```

/etc/sysconfig/network-scripts/ifcfg-eth2.704

```ini
VLAN=yes
TYPE=Vlan
PHYSDEV=eth2
VLAN_ID=704
REORDER_HDR=yes
GVRP=no
MVRP=no
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
IPADDR=10.0.2.1
PREFIX=24
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME="eth2.704"
DEVICE=eth2.704
ONBOOT=yes
MTU="9000"
```

/etc/sysconfig/network-scripts/ifcfg-vhost0

```ini
DEVICE=vhost0
BOOTPROTO=none
ONBOOT=yes
IPADDR=10.0.2.1
PREFIX=24
NETWORK=10.0.2.0
TYPE=kernel
NM_CONTROLLED=no
BIND_INT=eth2.704
MTU="9000"
```

#### 1.4.2 主机路由配置

所有计算节点增加如下主机路由配置

/etc/sysconfig/network-scripts/route-vhost0

```
172.16.6.34/32 via 10.0.2.254 dev vhost0
172.16.6.36/32 via 10.0.2.254 dev vhost0
172.16.6.37/32 via 10.0.2.254 dev vhost0
```

### 1.5 操作系统基础配置

所有节点的基础配置

#### 1.5.1 NTP配置

/etc/ntp.conf

```ini
tinker panic 0

disable monitor
restrict default kod nomodify notrap nopeer noquery
restrict -6 default kod nomodify notrap nopeer noquery
restrict 127.0.0.1
restrict -6 ::1
server 192.168.184.254 iburst


# Driftfile.
driftfile /var/lib/ntp/drift
```

Contrail Ansible Deployer部署的节点，NTP为自动配置。

#### 1.5.2 本地源配置

/etc/yum.repos.d/local.repo

```ini
[CentOS7.6-base]
name=CentOS7.6-base
baseurl=http://192.168.199.135/centos7.6/base/
gpgcheck=0
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[CentOS7.6-updates]
name=CentOS7.6-updates
baseurl=http://192.168.199.135/centos7.6/updates
gpgcheck=0
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[CentOS7.6-epel]
name=CentOS7.6-epel
baseurl=http://192.168.199.135/centos7.6/epel
gpgcheck=0
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[CentOS7.6-extras]
name=CentOS7.6-extras
baseurl=http://192.168.199.135/centos7.6/extras
gpgcheck=0
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[CentOS7.6-centosplus]
name=CentOS7.6-centosplus
baseurl=http://192.168.199.135/centos7.6/centosplus
gpgcheck=0
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[CentOS7.6-docker-ce-stable]
name=CentOS7.6-docker-ce-stable
baseurl=http://192.168.199.135/centos7.6/docker-ce-stable
gpgcheck=0
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
```

#### 1.5.3 Docker配置

/etc/docker/daemon.json

```
{"insecure-registries":["192.168.184.201:5000"]}
```

- `192.168.184.201`为c2b13-cbuilder的IP

## 2. 安装准备

### 2.1 安装Docker Registry

在c2b13-cbuilder上执行

```bash
yum install python-setuptools python-pip -y
yum install -y yum-utils device-mapper-persistent-data lvm2
yum install -y docker-ce
systemctl start docker
systemctl enable docker
mkdir /opt/registry
docker run -d -p 5000:5000 -v /opt/registry:/var/lib/registry --name registry registry:latest
```

### 2.2 拉取离线镜像

```bash
# 数讯的账号: JNPR-Customer63
# 密码：FwkLHQ6cCKXBR2AJETdV
docker login hub.juniper.net --username JNPR-Customer63 --password FwkLHQ6cCKXBR2AJETdV

list="contrail-controller-control-named contrail-vrouter-kernel-init-dpdk contrail-controller-webui-web contrail-analytics-alarm-gen contrail-kubernetes-cni-init contrail-analytics-topology contrail-vcenter-manager contrail-nodemgr contrail-kubernetes-kube-manager contrail-debug contrail-external-dhcp contrail-openstack-ironic-notification-manager contrail-node-init contrail-vrouter-agent contrail-analytics-api contrail-controller-config-schema contrail-vrouter-kernel-init contrail-external-redis contrail-vrouter-kernel-build-init contrail-external-cassandra contrail-analytics-snmp-collector contrail-controller-control-dns contrail-analytics-query-engine contrail-vrouter-agent-dpdk contrail-analytics-collector contrail-controller-control-control contrail-openstack-compute-init contrail-controller-config-devicemgr contrail-controller-webui-job contrail-openstack-neutron-init contrail-openstack-heat-init contrail-vcenter-plugin contrail-controller-config-api contrail-external-zookeeper contrail-status contrail-external-rabbitmq contrail-external-tftp contrail-external-kafka contrail-controller-config-svcmonitor"

registry=192.168.184.201:5000
version=5.0.3-0.493-queens
oldver=hub.juniper.net/contrail

for image in $list
do
   docker pull $oldver/$image:$version
   docker tag $oldver/$image:$version $registry/$image:$version
   docker push $registry/$image:$version
   echo_g "Push $image:$version success!"
done 
```

## 3. 安装Contrail

### 3.1 安装contrail-ansible-deployer

在2b13-cbuilder上执行。

```bash
yum install -y ansible-2.4.2.0 git
cd /root
git clone -b R5.0 http://github.com/Juniper/contrail-ansible-deployer
```

### 3.2 配置部署集群信息

在2b13-cbuilder上执行。

/root/contrail-ansible-deployer/config/instances.yaml

```yaml
provider_config:
  bms:
    ssh_pwd: shuxun.@123
    ssh_user: root
    ssh_public_key: /root/.ssh/id_rsa.pub
    ssh_private_key: /root/.ssh/id_rsa
    ntpserver: 192.168.184.254
instances:
  bms1:
    provider: bms
    ip: 10.0.0.101
    roles:
      config_database:
      config:
      control:
      webui:
  bms2:
    provider: bms
    ip: 10.0.0.102
    roles:
      config_database:
      config:
      control:
      webui:
  bms3:
    provider: bms
    ip: 10.0.0.103
    roles:
      config_database:
      config:
      control:
      webui:
  bms4:
    provider: bms
    ip: 10.0.0.104
    roles:
      analytics_database:
      analytics:
  bms5:
    provider: bms
    ip: 10.0.0.105
    roles:
      analytics_database:
      analytics:
  bms6:
    provider: bms
    ip: 10.0.0.106
    roles:
      analytics_database:
      analytics:
  bms7:
    provider: bms
    ip: 10.0.0.1
    roles:
      vrouter:
  bms8:
    provider: bms
    ip: 10.0.0.2
    roles:
      vrouter:
  bms9:
    provider: bms
    ip: 10.0.0.3
    roles:
      vrouter:
  bms10:
    provider: bms
    ip: 10.0.0.4
    roles:
      vrouter:
  bms11:
    provider: bms
    ip: 10.0.0.17
    roles:
      vrouter:
  bms12:
    provider: bms
    ip: 10.0.0.18
    roles:
      vrouter:
  bms13:
    provider: bms
    ip: 10.0.0.19
    roles:
      vrouter:
  bms14:
    provider: bms
    ip: 10.0.0.20
    roles:
      vrouter:
  bms15:
    provider: bms
    ip: 10.0.0.202
    roles:
      vrouter:
        TSN_EVPN_MODE: True
        TSN_NODES: 10.0.2.202
  c2b07-stack-cmp07:
    provider: bms
    ip: 10.0.0.7
    roles:
      vrouter:
  c3b07-stack-cmp07:
    provider: bms
    ip: 10.0.0.23
    roles:
      vrouter:
global_configuration:
  CONTAINER_REGISTRY: 192.168.184.201:5000
  REGISTRY_PRIVATE_INSECURE: True
contrail_configuration:
  CONTRAIL_VERSION: 5.0.3-0.493-queens
  CLOUD_ORCHESTRATOR: openstack
  CONTROLLER_NODES: 10.0.0.101,10.0.0.102,10.0.0.103
  CONTROL_NODES: 10.0.0.101,10.0.0.102,10.0.0.103
  CONFIGDB_NODES: 10.0.0.101,10.0.0.102,10.0.0.103
  ANALYTICS_NODES: 10.0.0.104,10.0.0.105,10.0.0.106
  ANALYTICSDB_NODES: 10.0.0.104,10.0.0.105,10.0.0.106
  VROUTER_GATEWAY: 10.0.2.254
  PHYSICAL_INTERFACE: eth2.704
  AAA_MODE: cloud-admin
  AUTH_MODE: keystone
  KEYSTONE_AUTH_ADMIN_TENANT: admin
  KEYSTONE_AUTH_ADMIN_USER: admin
  KEYSTONE_AUTH_ADMIN_PASSWORD: oss!SDS@openstack20190530
  KEYSTONE_AUTH_HOST: 10.0.0.254
  KEYSTONE_AUTH_URL_VERSION: '/v3'
  KEYSTONE_AUTH_URL_TOKENS: '/v3/auth/tokens'
  KEYSTONE_AUTH_USER_DOMAIN_NAME: default
  KEYSTONE_AUTH_PROJECT_DOMAIN_NAME: default
  #IPFABRIC_SERVICE_HOST: 192.168.184.100
  #IPFABRIC_SERVICE_HOST: 10.0.0.100
  CONFIG_DATABASE_NODEMGR__DEFAULTS__minimum_diskGB: 4
  DATABASE_NODEMGR__DEFAULTS__minimum_diskGB: 4
  METADATA_PROXY_SECRET: v3YH6KeEkyhVJMa2glm25XkEkvnEr4qbkZpb5n9N
  BGP_ASN: 64515
kolla_config:
  kolla_globals:
    #enable_ironic: "no"
    #enable_swift: "no"
    kolla_internal_vip_address: 10.0.0.254
    kolla_external_vip_address: 10.0.0.254
    contrail_api_interface_address: 10.0.0.254
    #kolla_external_vip_address: 192.168.184.100
    #contrail_api_interface_address: 192.168.184.100
  kolla_passwords:
    #keystone_admin_password: contrail123
    keystone_admin_password: oss!SDS@openstack20190530
```

### 3.3 部署

在2b13-cbuilder上执行。

```bash
cd /root/contrail-ansible-deployer
# 基础配置
ansible-playbook -e orchestrator=openstack -i inventory/ playbooks/configure_instances.yml
# 安装contrail
ansible-playbook -e orchestrator=openstack -i inventory/ playbooks/install_contrail.yml
```

* *其中没有执⾏行行 ansible-playbook -i inventory/ playbooks/install_openstack.yml是因为openstack是客户⾃自⼰己使⽤用kolla部署。*

### 3.4 Neutron对接

三个节点c2b13-stack01 ，c2b14-stack012， c3b13-stack03都要修改neutron-server容器。

#### 3.4.1 修改Neutron配置

修改⽂文件 /etc/neutron/neutron.conf。

```ini
[DEFAULT]
core_plugin = neutron_plugin_contrail.plugins.opencontrail.contrail_plugin.NeutronPluginContrailCoreV2
service_plugins = neutron_plugin_contrail.plugins.opencontrail.loadbalancer.v2.plugin.LoadBalancerPluginV2
api_extensions_path = /usr/lib/python2.7/site-packages/neutron_plugin_contrail/extensions:/usr/lib/python2.7/site-packages/neutron_lbaas/extensions

[quotas]
quota_network = -1
quota_subnet = -1
quota_port = -1
quota_driver =
neutron_plugin_contrail.plugins.opencontrail.quota.driver.QuotaDriver
```

#### 3.4.2 增加Core Plugin配置文件

增加`/etc/neutron/plugins/opencontrail/ContrailPlugin.ini`

权限`-rw-------`，用户`neutron`，组`neutron`

```
[APISERVER]
api_server_ip = 10.0.0.254
api_server_port = 8082
multi_tenancy = True

contrail_extensions = ipam:neutron_plugin_contrail.plugins.opencontrail.contrail_plugin_ipam.NeutronPluginContrailIpam,policy:neutron_plugin_contrail.plugins.opencontrail.contrail_plugin_policy.NeutronPluginContrailPolicy,route-table:neutron_plugin_contrail.plugins.opencontrail.contrail_plugin_vpc.NeutronPluginContrailVpc,contrail:None,service-interface:None,vf-binding:None

[COLLECTOR]
analytics_api_ip = 10.0.0.254
analytics_api_port = 8081
```

#### 3.4.3 增加代码

拷贝`root@c2b13-stack01` 上的文件夹`/root/contrail- openstack-neutron-init-queens/`下的所有⽂件到容器`neutron_server`内的⽬目录/usr/lib/python2.7/site-packages/。

涉及三个节点`c2b13-stack01` ，`c2b14-stack012`， `c3b13-stack03`。

#### 3.4.4 启动命令配置文件替换

将`neutron-server`容器中`/usr/bin/neutron-server`的配置文件更换。使用`--config-file /etc/neutron/plugins/opencontrail/ContrailPlugin.ini`替换掉`--config-file /etc/neutron/plugins/ml2/ml2_conf.ini`

### 3.5  MX960关键配置

```
# 1. GRE Tunnel Setting
set groups __contrail_overlay_bgp__ routing-options dynamic-tunnels _contrail_asn-64515 source-address 172.16.6.34
set groups __contrail_overlay_bgp__ routing-options dynamic-tunnels _contrail_asn-64515 udp
set groups __contrail_overlay_bgp__ routing-options dynamic-tunnels _contrail_asn-64515 destination-networks 10.0.2.0/24

set chassis fpc 5 pic 0 tunnel-services
set chassis fpc 5 pic 0 inline-services bandwidth 1g
set chassis fpc 5 pic 1 tunnel-services
set chassis fpc 5 pic 2 tunnel-services
set chassis fpc 7 pic 0 tunnel-services
set chassis fpc 7 pic 1 tunnel-services
set chassis fpc 7 pic 2 tunnel-services

# 2. OpenStack iBGP Configuration
set groups __contrail_basic__ snmp community public authorization read-only
set groups __contrail_overlay_bgp__ routing-options router-id 172.16.6.34
set groups __contrail_overlay_bgp__ routing-options route-distinguisher-id 172.16.6.34
set groups __contrail_overlay_bgp__ routing-options autonomous-system 64515
set groups __contrail_overlay_bgp__ routing-options resolution rib bgp.rtarget.0 resolution-ribs inet.0
set groups __contrail_overlay_bgp__ protocols bgp group _contrail_asn-64515 type internal
set groups __contrail_overlay_bgp__ protocols bgp group _contrail_asn-64515 local-address 172.16.6.34
set groups __contrail_overlay_bgp__ protocols bgp group _contrail_asn-64515 hold-time 90
set groups __contrail_overlay_bgp__ protocols bgp group _contrail_asn-64515 family inet-vpn unicast
set groups __contrail_overlay_bgp__ protocols bgp group _contrail_asn-64515 family inet6-vpn unicast
set groups __contrail_overlay_bgp__ protocols bgp group _contrail_asn-64515 family evpn signaling
set groups __contrail_overlay_bgp__ protocols bgp group _contrail_asn-64515 family route-target
set groups __contrail_overlay_bgp__ protocols bgp group _contrail_asn-64515 export mpls_over_udp
set groups __contrail_overlay_bgp__ protocols bgp group _contrail_asn-64515 export _contrail_ibgp_export_policy
set groups __contrail_overlay_bgp__ protocols bgp group _contrail_asn-64515 vpn-apply-export
set groups __contrail_overlay_bgp__ protocols bgp group _contrail_asn-64515 multipath
set groups __contrail_overlay_bgp__ protocols bgp group _contrail_asn-64515 neighbor 10.0.0.101 peer-as 64515
set groups __contrail_overlay_bgp__ protocols bgp group _contrail_asn-64515 neighbor 10.0.0.102 peer-as 64515
set groups __contrail_overlay_bgp__ protocols bgp group _contrail_asn-64515 neighbor 10.0.0.103 peer-as 64515
set groups __contrail_overlay_bgp__ protocols bgp group _contrail_asn-64515 neighbor 172.16.6.36 peer-as 64515
set groups __contrail_overlay_bgp__ protocols bgp group _contrail_asn-64515 neighbor 172.16.6.37 peer-as 64515
set groups __contrail_overlay_bgp__ policy-options policy-statement mpls_over_udp term 1 then community add encap-udp
set groups __contrail_overlay_bgp__ policy-options policy-statement mpls_over_udp term 1 then accept
set groups __contrail_overlay_bgp__ policy-options policy-statement _contrail_ibgp_export_policy term inet-vpn from family inet-vpn
set groups __contrail_overlay_bgp__ policy-options policy-statement _contrail_ibgp_export_policy term inet-vpn then next-hop self
set groups __contrail_overlay_bgp__ policy-options policy-statement _contrail_ibgp_export_policy term inet6-vpn from family inet6-vpn
set groups __contrail_overlay_bgp__ policy-options policy-statement _contrail_ibgp_export_policy term inet6-vpn then next-hop self
set groups __contrail_overlay_bgp__ policy-options community encap-udp members 0x030c:64512:13
set groups __contrail_overlay_fip_snat__
set groups __contrail_overlay_networking__
set apply-groups __contrail_basic__
set apply-groups __contrail_overlay_bgp__
set apply-groups __contrail_overlay_fip_snat__
set apply-groups __contrail_overlay_networking__


# 3. RIB Group and Routing Instance Configuration

## 3.1 CTCC 64515:100 OpenStack 120.136.156.128/26
set routing-instances _CTCC-openstack-100 instance-type vrf
set routing-instances _CTCC-openstack-100 vrf-target target:64515:100
set routing-instances _CTCC-openstack-100 vrf-table-label
commit check
commit

set routing-options rib-groups inet0-to-vrf import-rib inet.0
set routing-options rib-groups inet0-to-vrf import-rib _CTCC-openstack-100.inet.0
commit check
commit

set routing-instances _CTCC-openstack-100 routing-options interface-routes rib-group inet inet0-to-vrf
set protocols bgp group ebgp family inet unicast rib-group inet0-to-vrf
commit check
commit

set routing-options static route 120.136.156.128/26 next-table _CTCC-openstack-100.inet.0
commit check
commit

## 3.2 BGP 64515:101 OpenStack 120.136.134.128/26
set routing-instances _BGP-openstack-101 instance-type vrf
set routing-instances _BGP-openstack-101 vrf-target target:64515:101
set routing-instances _BGP-openstack-101 vrf-table-label
commit check
commit

set routing-options rib-groups inet0-to-vrf import-rib _BGP-openstack-101.inet.0
set routing-instances _BGP-openstack-101 routing-options interface-routes rib-group inet inet0-to-vrf 
set routing-options static route 120.136.134.128/26 next-table _BGP-openstack-101.inet.0
commit check
commit

## 3.3 CUCC 64515:102 OpenStack 120.136.178.128/26
set routing-instances _CUCC-openstack-102 instance-type vrf
set routing-instances _CUCC-openstack-102 vrf-target target:64515:102
set routing-instances _CUCC-openstack-102 vrf-table-label
commit check
commit

set routing-options rib-groups inet0-to-vrf import-rib _CUCC-openstack-102.inet.0
set routing-instances _CUCC-openstack-102 routing-options interface-routes rib-group inet inet0-to-vrf 
set routing-options static route 120.136.178.128/26 next-table _CUCC-openstack-102.inet.0
commit check
commit

## 3.4 CUCC 64515:103 OpenStack 120.136.178.0/25
set routing-instances _CUCC-openstack-103 instance-type vrf
set routing-instances _CUCC-openstack-103 vrf-target target:64515:103
set routing-instances _CUCC-openstack-103 vrf-table-label
commit check
commit

set routing-options rib-groups inet0-to-vrf import-rib _CUCC-openstack-103.inet.0
set routing-instances _CUCC-openstack-103 routing-options interface-routes rib-group inet inet0-to-vrf 
set routing-options static route 120.136.178.0/25 next-table _CUCC-openstack-103.inet.0
commit check
commit

## 3.5 CTCC 64515:104 OpenStack 120.136.156.0/25
set routing-instances _CTCC-openstack-104 instance-type vrf
set routing-instances _CTCC-openstack-104 vrf-target target:64515:104
set routing-instances _CTCC-openstack-104 vrf-table-label
commit check
commit

set routing-options rib-groups inet0-to-vrf import-rib _CTCC-openstack-104.inet.0
set routing-instances _CTCC-openstack-104 routing-options interface-routes rib-group inet inet0-to-vrf
set routing-options static route 120.136.156.0/25 next-table _CTCC-openstack-104.inet.0
commit check
commit

# 3.6 Graceful Restart
set protocols bgp graceful-restart
set protocols bgp group CStack_Controller family inet-vpn unicast graceful-restart long-lived restarter stale-time 20
set protocols bgp group CStack_Controller family route-target graceful-restart long-lived restarter stale-time 20
set protocols bgp group CStack_Controller graceful-restart restart-time 300
delete routing-options nonstop-routing
```



## 4. 新增计算节点

以c2b07-stakc-cmp07为例。

### 4.1 网络配置

/etc/sysconfig/network-scripts/ifcfg-eth0

```ini
DEVICE=eth0
ONBOOT=yes
HWADDR=98:F2:B3:A3:C9:90
TYPE=Ethernet
BOOTPROTO=none
IPADDR=192.168.184.7
GATEWAY=192.168.184.254
NETMASK=255.255.255.0
NM_CONTROLLED=no
```

/etc/sysconfig/network-scripts/ifcfg-eth1

```
DEVICE=eth1
ONBOOT=yes
HWADDR=98:F2:B3:A3:C9:98
TYPE=Ethernet
BOOTPROTO=none
NM_CONTROLLED=no
```

/etc/sysconfig/network-scripts/ifcfg-eth1.702

```
DEVICE=eth1.702
BOOTPROTO=none
ONBOOT=yes
IPADDR=10.0.0.7
PREFIX=24
NETWORK=10.0.0.0
VLAN=yes
NM_CONTROLLED=no
```

/etc/sysconfig/network-scripts/ifcfg-eth2

```ini
DEVICE=eth2
IPV6INIT=no
BOOTPROTO=none
ONBOOT=no
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
DEFROUTE=no
IPV4_FAILURE_FATAL=no
MTU="9000"
```

/etc/sysconfig/network-scripts/ifcfg-eth2.704

```ini
VLAN=yes
TYPE=Vlan
PHYSDEV=eth2
VLAN_ID=704
REORDER_HDR=yes
GVRP=no
MVRP=no
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=no
IPADDR=10.0.2.7
PREFIX=24
DEFROUTE=no
IPV4_FAILURE_FATAL=no
IPV6INIT=no
NAME="eth2.704"
DEVICE="eth2.704"
ONBOOT=yes
MTU="9000"
```

/etc/sysconfig/network-scripts/ifcfg-eth3

```ini
DEVICE=eth3
ONBOOT=no
HWADDR=98:F2:B3:A3:C9:9A
TYPE=Ethernet
BOOTPROTO=none
NM_CONTROLLED=no
```

配置完成后，请运行`systemctl restart network.service `。以确保网络配置正确。

### 4.2 移除OpenVSwitch

```bash
docker exec -it openvswitch_vswitchd bash
ovs-dpctl dump-dps
ovs-dpctl del-dp system@ovs-system
exit

docker stop neutron_openvswitch_agent openvswitch_vswitchd openvswitch_db
rmmod vport_vxlan
rmmod openvswitch
docker ps --all | grep openv | awk '{print $1}' | xargs -i docker rm {}
```

### 4.3 移除docker-py

```bash
yum install python-setuptools python-pip -y
pip install --upgrade pip
pip uninstall docker-py docker
```

如果在4.4`install_contrail`步骤中遇到docker-py相关错误，执行以上步骤。

### 4.4 安装Contrail vRouter

在c2b13-cbuilder节点的/root/contrail-ansible-deployer/config/instances.yaml中增加如下内容，内容放置在instances节下。

```ini
instances:
  c2b07-stack-cmp07:
    provider: bms
    ip: 10.0.0.7
    roles:
      vrouter:
```

在c2b13-cbuilder节点执行部署。

```bash
ansible-playbook -e orchestrator=openstack -e '{"instances":{"c2b07-stack-cmp07":{"ip":"10.0.0.7","provider":"bms","roles":{"vrouter": \n}}}}' -i inventory/ playbooks/configure_instances.yml

ansible-playbook -e orchestrator=openstack -e '{"instances":{"c2b07-stack-cmp07":{"ip":"10.0.0.7","provider":"bms","roles":{"vrouter": \n}}}}' -i inventory/ playbooks/install_contrail.yml
```

- 不同的节点请注意修改`-e`参数的内容；

### 4.5 对接数讯nova-compute

在c2b13-cbuilder节点执行，拷贝相关代码到c2b07-stack-cmp07。

```bash
scp /root/compute-plugin.tgz c2b07-stack-cmp07.cloud.shuxun.net:/root/
```

在c2b07-stack-cmp07上执行如下步骤，进行安装

```bash
cd /root/
tar -zxvf compute-plugin.tgz

docker cp /root/contrail-openstack-compute-init-queens/site-packages/nova_contrail_vif nova_compute:/usr/lib/python2.7/site-packages/
docker cp /root/contrail-openstack-compute-init-queens/site-packages/nova_contrail_vif-0.1-py2.7.egg-info/ nova_compute:/usr/lib/python2.7/site-packages/
docker cp /root/contrail-openstack-compute-init-queens/site-packages/vif_plug_vrouter/ nova_compute:/usr/lib/python2.7/site-packages/
docker cp /root/contrail-openstack-compute-init-queens/bin/vrouter-port-control nova_compute:/usr/bin/

docker cp /root/contrail-openstack-compute-init-queens/site-packages/nova_contrail_vif nova_libvirt:/usr/lib/python2.7/site-packages/
docker cp /root/contrail-openstack-compute-init-queens/site-packages/nova_contrail_vif-0.1-py2.7.egg-info/ nova_libvirt:/usr/lib/python2.7/site-packages/
docker cp /root/contrail-openstack-compute-init-queens/site-packages/vif_plug_vrouter/ nova_libvirt:/usr/lib/python2.7/site-packages/
docker cp /root/contrail-openstack-compute-init-queens/bin/vrouter-port-control nova_libvirt:/usr/bin/

## Libvirt bug
docker cp /root/contrail-openstack-compute-init-queens/designer.py nova_compute:/lib/python2.7/site-packages/nova/virt/libvirt/designer.py

docker restart nova_compute
docker restart nova_libvirt
```

### 4.6 部署后配置

/etc/sysconfig/network-scripts/route-vhost0

```ini
172.16.6.34/32 via 10.0.2.254 dev vhost0
172.16.6.36/32 via 10.0.2.254 dev vhost0
172.16.6.37/32 via 10.0.2.254 dev vhost0
```

执行

```bash
ip r add 172.16.6.34/32 via 10.0.2.254 dev vhost0
ip r add 172.16.6.36/32 via 10.0.2.254 dev vhost0
ip r add 172.16.6.37/32 via 10.0.2.254 dev vhost0

docker restart vrouter_vrouter-agent_1
docker restart vrouter_nodemgr_1
```

##  5. Contrail Command安装

参考文档：https://www.juniper.net/documentation/en_US/contrail5.0/topics/task/configuration/import-cluster-data-contrail-command.html

### 5.1 准备镜像

在c2b13-cbuilder上执行。

```bash
docker login hub.juniper.net --username JNPR-Customer63 --password FwkLHQ6cCKXBR2AJETdV

docker pull hub.juniper.net/contrail/contrail-command-deployer:5.0.3-0.493
docker pull hub.juniper.net/contrail/contrail-command:5.0.3-0.493
docker tag hub.juniper.net/contrail/contrail-command-deployer:5.0.3-0.493 192.168.184.201:5000/contrail-command-deployer:5.0.3-0.493
docker tag hub.juniper.net/contrail/contrail-command:5.0.3-0.493 192.168.184.201:5000/contrail-command:5.0.3-0.493
docker push 192.168.184.201:5000/contrail-command-deployer:5.0.3-0.493
docker push 192.168.184.201:5000/contrail-command:5.0.3-0.493
```

### 5.2 创建虚拟机和配置基础网络

在c2b13-ccmd01上执行。

参考1.2和1.3章节（已包含`c2b13-ccmd01`虚拟机配置）

参考1.5进行基础配置

### 5.3 安装前准备

在c2b13-ccmd01上执行。

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum install -y docker-ce
systemctl start docker
systemctl enable docker

mkdir /root/ayang

# 将c2b13-cbuilder上/root/contrail-ansible-deployer/config/instances.yaml拷贝到/root/ayang下，并改名为instances.yml
```

编辑/root/ayang/command_servers.yml

```yaml
---
command_servers:
    server1:
        ip: 10.0.0.203
        connection: ssh
        ssh_user: root
        ssh_pass: shuxun.@123
        sudo_pass: shuxun.@123
        ntpserver: 192.168.184.254

        # Specify either container_path
        #container_path: /root/contrail-command-051618.tar
        # or registry details and container_name
        # registry_insecure: true
        # container_registry: ci-repo.englab.juniper.net:5010
        registry_insecure: true
        container_registry: 192.168.184.201:5000
        container_name: contrail-command
        container_tag: 5.0.3-0.493
        #container_registry_username:
        #container_registry_password:
        config_dir: /etc/contrail

        # contrail command container configurations given here go to /etc/contrail/contrail.yml
        contrail_config:
            # Database configuration. MySQL/PostgreSQL supported
            database:
                # MySQL example
                type: mysql
                dialect: mysql
                host: localhost
                user: root
                password: shuxun.@123
                name: contrail_test
                # Postgres example
                #connection: "user=root dbname=contrail_test sslmode=disable"
                #type: postgres
                #dialect: postgres

                # Max Open Connections for DB Server
                max_open_conn: 100
                connection_retries: 10
                retry_period: 3s

            # Log Level
            log_level: debug

            # Server configuration
            server:
              enabled: true
              read_timeout: 10
              write_timeout: 5
              log_api: true
              address: ":9091"

              # TLS Configuration
              tls:
                  enabled: true
                  key_file: /usr/share/contrail/ssl/cs-key.pem
                  cert_file: /usr/share/contrail/ssl/cs-cert.pem

              # Enable GRPC or not
              enable_grpc: false

              # Static file config
              # key: URL path
              # value: file path. (absolute path recommended in production)
              static_files:
                  /: /usr/share/contrail/public

              # API Proxy configuration
              # key: URL path
              # value: String list of backend host
              #proxy:
              #    /contrail:
              #    - http://localhost:8082

              notify_etcd: false

            # Keystone configuration
            keystone:
                local: true
                assignment:
                    type: static
                    data:
                      domains:
                        default: &default
                          id: default
                          name: default
                      projects:
                        admin: &admin
                          id: admin
                          name: admin
                          domain: *default
                        demo: &demo
                          id: demo
                          name: demo
                          domain: *default
                      users:
                        admin:
                          id: admin
                          name: Admin
                          domain: *default
                          password: shuxun.@123
                          email: admin@juniper.nets
                          roles:
                          - id: admin
                            name: Admin
                            project: *admin
                        bob:
                          id: bob
                          name: Bob
                          domain: *default
                          password: bob_password
                          email: bob@juniper.net
                          roles:
                          - id: Member
                            name: Member
                            project: *demo
                store:
                    type: memory
                    expire: 36000
                insecure: true
                authurl: https://localhost:9091/keystone/v3

            # disable authentication with no_auth true and comment out keystone configuraion.
            #no_auth: true
            insecure: true

            etcd:
              endpoints:
                - localhost:2379
              username: ""
              password: ""
              path: contrail

            watcher:
              enabled: false
              storage: json

            client:
              id: admin
              password: shuxun.@123
              project_name: admin
              domain_id: default
              schema_root: /
              endpoint: https://localhost:9091

            compilation:
              enabled: false
              # Global configuration
              plugin_directory: 'etc/plugins/'
              number_of_workers: 4
              max_job_queue_len: 5
              msg_queue_lock_time: 30
              msg_index_string: 'MsgIndex'
              read_lock_string: "MsgReadLock"
              master_election: true
              
              # Plugin configuration
              plugin:
                  handlers:
                      create_handler: 'HandleCreate'
                      update_handler: 'HandleUpdate'
                      delete_handler: 'HandleDelete'

            agent:
              enabled: true
              backend: file
              watcher: polling
              log_level: debug

         # The following are optional parameters used to patch/cherrypick
         # revisions into the contrail-ansible-deployer sandbox. These configs
         # go into the /etc/contrail/contrail-cluster.tmpl file
#        cluster_config:
#            ansible_fetch_url: "https://review.opencontrail.org/Juniper/contrail-ansible-deployer refs/changes/80/40780/20"
#            ansible_cherry_pick_revision: FETCH_HEAD
#            ansible_revision: GIT_COMMIT_HASH
```

执行

```bash
docker run -t --net host -e orchestrator=openstack -e action=import_cluster -v /root/ayang/command_servers.yml:/command_servers.yml -v /root/ayang/instances.yml:/instances.yml -d --privileged --name contrail_command_deployer 192.168.184.201:5000/contrail-command-deployer:5.0.3-0.493
```

