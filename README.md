Joint US ATLAS / IRIS-HEP Kubernetes Hackathon
==============================================

This repository contains instructions and contents for GitOps Kubernetes deployments using Flux.
It assumes that you have set up a basic Kubernetes cluster.

1.  Clone this repository to your tutorial host

        git clone https://github.com/osg-htc/k8s-hackathon-training

2.  [Install Flux components](manifests/flux-install)

3.  [Bootstrap Flux syncing](clusters#bootstrap-flux) with this repository

4.  [Install the Sealed Secrets](manifests/sealed-secrets) operator
    and replace the Flux SSH key secret with a sealed secret

6.  Create `osg` and `usatlas` [namespaces](clusters/README.md#)
