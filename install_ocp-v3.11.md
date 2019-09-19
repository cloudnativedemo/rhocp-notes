# Installing Red Hat Openshift (OKD) v3.11

This doc is showing my current experience of installing OKD on a multiple nodes topology on IBM Cloud.

The install process is based on Red Hat OCP's official docs: https://docs.okd.io/latest/install/index.html

In my topology:
  - Master node: 8 CPUs x 32 GB RAM, xvdc: 200GB
  - Infra node: 4 CPUs x 8 GB RAM, xvdc: 200GB
  - Compute node 01: 4 CPUs x 8 GB RAM, xvdc: 200GB, xvde: 75GB (GlusterFS)
  - Compute node 02: 4 CPUs x 8 GB RAM, xvdc: 200GB, xvde: 75GB (GlusterFS)
  - Compnute node 03: 4 CPUs x 8 GB RAM, xvdc: 200GB, xvde: 75GB (GlusterFS)

##1. Prepare file system  

| Node    | File system                                                                                                       | Size                                     |
|---------|-------------------------------------------------------------------------------------------------------------------|------------------------------------------|
| master  | /var/lib/openshift   /var/lib/etcd   /var/lib/docker   /var/lib/origin/openshift.local.volumes   /var/lib/kubelet | 10 GB   20 GB   50 GB   Varies   5  GB   |
| infra   | /var/lib/docker   /var/lib/kubelet                                                                                | 50 GB   5  GB                            |
| compute | /var/lib/docker   /var/lib/kubelet                                                                                | 50 GB   5  GB                            |



