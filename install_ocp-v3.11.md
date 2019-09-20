# Installing Red Hat Openshift (OKD) v3.11

This doc is showing my current experience of installing OKD on a multiple nodes topology on IBM Cloud.

The install process is based on Red Hat OCP's official docs: https://docs.okd.io/latest/install/index.html

In my topology:
  - Master node: 8 CPUs x 32 GB RAM, xvdc: 200GB (for IBM common services layer to be installed later, min requirement is 4 cores)
  - Infra node: 4 CPUs x 8 GB RAM, xvdc: 200GB
  - Compute node 01: 4 CPUs x 8 GB RAM, xvdc: 200GB, xvde: 75GB (GlusterFS)
  - Compute node 02: 4 CPUs x 8 GB RAM, xvdc: 200GB, xvde: 75GB (GlusterFS)
  - Compnute node 03: 4 CPUs x 8 GB RAM, xvdc: 200GB, xvde: 75GB (GlusterFS)

## 1. Firewall/Security groups configuration
- Master node: 22/TCP, 53/UDP, 53/TCP, 8053/TCP, 8053/UDP, 10250/TCP, 2379-2380/TCP, 8443/TCP, 4789/UDP, 8444/TCP, 8445/TCP
- Infra node: 80/TCP, 443/TCP, 1936/TCP, 8080/TCP, + Ports from compute nodes
- Compute node: 22/TCP, 53/TCP, 53/UDP, 8053/TCP, 8053/UDP, 10010/TCP, 4789/UDP, 8445/TCP, 24007-24009/TCP, 10250/TCP

## 2. Configure SSH access and key

Since I'm installing OCP from my master node, I need to ensure that I can ssh to other nodes using a ssh key  
If your nodes don't allow `root` access, you must enable it
```shell
vi /etc/ssh/sshd_config

# Set PermitRootLogin to yes
```
- Generate SSH keys if you don't have any

```shell
mkdir -p /root/.ssh
sudo ssh-keygen -b 4096 -t rsa -f /root/.ssh/id_rsa -N ""
```
- Copy SSH public key to other node  
```shell
export SSH_KEY=$(cat /root/.ssh/id_rsa.pub)
ssh root@master "echo ${SSH_KEY} | tee -a /root/.ssh/authorized_keys"
ssh root@infra "echo ${SSH_KEY} | tee -a /root/.ssh/authorized_keys"
ssh root@node01 "echo ${SSH_KEY} | tee -a /root/.ssh/authorized_keys"
ssh root@node02 "echo ${SSH_KEY} | tee -a /root/.ssh/authorized_keys"
ssh root@node03 "echo ${SSH_KEY} | tee -a /root/.ssh/authorized_keys"

tee /root/.ssh/config << EOF
IdentityFile /root/.ssh/id_rsa
EOF
```

## 3. Prepare file system  

| Node    | File system                                                                                                       | Size                                     |
|---------|-------------------------------------------------------------------------------------------------------------------|------------------------------------------|
| master  | /var/lib/openshift<br/>   /var/lib/etcd<br/>   /var/lib/docker<br/>   /var/lib/origin/openshift.local.volumes<br/>   /var/lib/kubelet<br/> | 10 GB<br/>   20 GB<br/>   50 GB<br/>   Varies<br/>   5  GB<br/>   |
| infra   | /var/lib/docker<br/>   /var/lib/kubelet<br/>                                                                                | 50 GB<br/>   5  GB<br/>                            |
| compute | /var/lib/docker<br/>   /var/lib/kubelet<br/>                                                                                | 50 GB<br/>   5  GB<br/>                            |


It's ideal if you can have these file system created separately, but in a cloud environment if you can only request a single storage device, you can use the following method to mount all these fs to the device

In my environment, the device that I had for OCP install is `/dev/xvdc`, you might have a different device name

Run the following shell commands in **all nodes** 

```shell
mkfs.xfs /dev/xvdc
lsblk -o NAME,FSTYPE,UUID # Note down the UUID value of the disk
mkdir /mnt/ocp # Or replace it with your own mounting path

tee -a /etc/fstab << EOF

#OCP disk
UUID=<REPLACE_WITH_YOUR_DEV_UUID>       /mnt/ocp        xfs     defaults        0 0
EOF

mount -a 
```

**On master node:**
```shell
{
mkdir -p /mnt/ocp/lib/docker /var/lib/docker;
mkdir -p /mnt/ocp/lib/etcd /var/lib/etcd;
mkdir -p /mnt/ocp/lib/openshift /var/lib/openshift;
mkdir -p /mnt/ocp/lib/origin /var/lib/origin;
mkdir -p /mnt/ocp/lib/kubelet /var/lib/kubelet;
}

# Mount volumes for Master node

tee -a /etc/fstab << EOF

#RHOCP volumes
/mnt/ocp/lib/openshift       /var/lib/openshift        none     rbind        0 0
/mnt/ocp/lib/etcd            /var/lib/etcd             none     rbind        0 0
/mnt/ocp/lib/origin          /var/lib/origin           none     rbind        0 0
/mnt/ocp/lib/docker          /var/lib/docker           none     rbind        0 0
/mnt/ocp/lib/kubelet         /var/lib/kubelet          none     rbind        0 0
EOF
mount -a

```

**On infra and compute (worker) nodes**

```shell
{
mkdir -p /mnt/ocp/lib/docker /var/lib/docker;
mkdir -p /mnt/ocp/lib/kubelet /var/lib/kubelet;
}

# Mount volumes for Master node

tee -a /etc/fstab << EOF

#RHOCP volumes
/mnt/ocp/lib/docker        /var/lib/docker         none     rbind        0 0
/mnt/ocp/lib/kubelet       /var/lib/kubelet        none     rbind        0 0
EOF
mount -a

```

## 4. Install prerequisite packages

- Check if NetworkManager is installed, enabled and started
- Install required yum packages

```shell
yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
subscription-manager repos --enable=rhel-7-server-extras-rpms

yum install NetworkManager wget git net-tools bind-utils yum-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct -y

systemctl start NetworkManager
systemctl enable NetworkManager

yum update -y

reboot

```

OCP requires the following properties to be set on the ethernet interface's network script. If you have 2 eth interfaces, update both of them

```shell
vi /etc/sysconfig/network-scripts/ifcfg-eth0
# Update or add the following properties
# NM_CONTROLLED=yes
# PEERDNS=yes
```

## 5. Update Ansible hosts file

Replace the hostname and DNS with your actual hostname.   
***Note:*** 
  - The nodes' hostname must be FQDN. If they're not, you can update your nodes' hostname using `hostnamectl` 
  - In my configuration, I use `HTPasswdPasswordIdentityProvider` as the identity provider. You can use a htpasswd generator to generate an username/password for the Openshift admin console. I found this one, it works perfectly - http://www.htaccesstools.com/htpasswd-generator/
  - To enable glusterfs, you need an additional raw storgage device on each of the compute (worker) node, it's '/dev/xvde' in my environment
  
```
vi /etc/ansible/hosts
```
```yaml
#/etc/ansible/hosts

[OSEv3:children]
masters
nodes
etcd
glusterfs

# Set variables common for all OSEv3 hosts
[OSEv3:vars]
ansible_ssh_user=root
openshift_deployment_type=origin

# Set the ports to 8443
openshift_master_api_port=8443
openshift_master_console_port=8443

# Going in as root can be another user with sudo
#ansible_ssh_user=<RHEL_USER_IF_NOT_ROOT>
#ansible_become=<'yes' IF_USER_IS_NOT_ROOT>

# Set the debug level to 4
debug_level=4

# Skipping checks that are hit and miss
openshift_disable_check=memory_availability,docker_image_availability,disk_availability

# use RHEL firewalld
os_firewall_use_firewalld=true

# uncomment the following to enable htpasswd authentication; defaults to DenyAllPasswordIdentityProvider
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]
openshift_master_htpasswd_users={'<YOUR_ADMIN_USERNAME_GOES_HERE>': '<YOUR_HT_PASSWORD_GOES_HERE>'}

#Provide a sub-domain for apps deployed on OCP
openshift_master_default_subdomain=apps.<YOUR_OCP_DOMAIN.COM>

#Provide the FQDN of the master node
openshift_master_cluster_hostname=master.<YOUR_OCP_DOMAIN.COM>

openshift_hosted_router_selector='node-role.kubernetes.io/infra=true'

#glusterfs configuration
openshift_storage_glusterfs_namespace=app-storage
openshift_storage_glusterfs_storageclass=true
openshift_storage_glusterfs_storageclass_default=false
openshift_storage_glusterfs_block_deploy=true
openshift_storage_glusterfs_block_host_vol_size=75 # REPLACE WITH THE ACTUAL SIZE OF /dev/xvde (in GB)
openshift_storage_glusterfs_block_storageclass=true
openshift_storage_glusterfs_block_storageclass_default=false

# host group for masters
[masters]
master.<YOUR_OCP_DOMAIN.COM>

# host group for etcd
[etcd]
master.<YOUR_OCP_DOMAIN.COM>

[glusterfs]
node01.<YOUR_OCP_DOMAIN.COM> glusterfs_devices='[ "/dev/xvde" ]'
node02.<YOUR_OCP_DOMAIN.COM> glusterfs_devices='[ "/dev/xvde" ]'
node03.<YOUR_OCP_DOMAIN.COM> glusterfs_devices='[ "/dev/xvde" ]'

[nodes]
master.<YOUR_OCP_DOMAIN.COM> openshift_node_group_name='node-config-master'
infra.<YOUR_OCP_DOMAIN.COM>  openshift_node_group_name='node-config-infra'
node01.<YOUR_OCP_DOMAIN.COM> openshift_node_group_name='node-config-compute'
node02.<YOUR_OCP_DOMAIN.COM> openshift_node_group_name='node-config-compute'
node03.<YOUR_OCP_DOMAIN.COM> openshift_node_group_name='node-config-compute'

```

## 6. You're now ready to install OKD/OCP
- If you're installing OCP, follow OCP docs to enable your OCP subscription
- If you're installing OKD (like me), `git clone` OKD ansible playbooks
```shell
mkdir -p /usr/share/ansible
cd /usr/share/ansible
git clone https://github.com/openshift/openshift-ansible
cd openshift-ansible
git checkout release-3.11
```

- Run OCP prerequisite playbook
```shell
ansible-playbook -i /etc/ansible/hosts /usr/share/ansible/openshift-ansible/playbooks/prerequisites.yml
```

- Run OCP deploy playbook
```shell
ansible-playbook -i /etc/ansible/hosts /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml
```

- If you need to uninstall
```shell
ansible-playbook -i /etc/ansible/hosts /usr/share/ansible/openshift-ansible/playbooks/adhoc/uninstall.yml

# make sure the following directories are also deleted in all nodes
# rm -rf ansible .ansible* .kube/ openshift_bootstrap/ .pki/
```
