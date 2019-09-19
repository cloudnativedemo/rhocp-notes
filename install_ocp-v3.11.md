# Installing Red Hat Openshift (OKD) v3.11

This doc is showing my current experience of installing OKD on a multiple nodes topology on IBM Cloud.

The install process is based on Red Hat OCP's official docs: https://docs.okd.io/latest/install/index.html

In my topology:
  - Master node: 8 CPUs x 32 GB RAM, xvdc: 200GB
  - Infra node: 4 CPUs x 8 GB RAM, xvdc: 200GB
  - Compute node 01: 4 CPUs x 8 GB RAM, xvdc: 200GB, xvde: 75GB (GlusterFS)
  - Compute node 02: 4 CPUs x 8 GB RAM, xvdc: 200GB, xvde: 75GB (GlusterFS)
  - Compnute node 03: 4 CPUs x 8 GB RAM, xvdc: 200GB, xvde: 75GB (GlusterFS)

## 1. Prepare file system  

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
