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



