## Installing Red Hat Openshift (OKD) v3.11

This doc is showing my current experience of installing OKD on a multiple nodes topology on IBM Cloud.

The install process is based on Red Hat OCP's official docs: https://docs.okd.io/latest/install/index.html

In my topology:
  - Master node: 8 CPUs x 32 GB RAM, xvdc: 200GB
  - Infra node: 4 CPUs x 8 GB RAM, xvdc: 200GB
  - Compute node 01: 4 CPUs x 8 GB RAM, xvdc: 200GB, xvde: 75GB (GlusterFS)
  - Compute node 02: 4 CPUs x 8 GB RAM, xvdc: 200GB, xvde: 75GB (GlusterFS)
  - Compnute node 03: 4 CPUs x 8 GB RAM, xvdc: 200GB, xvde: 75GB (GlusterFS)

