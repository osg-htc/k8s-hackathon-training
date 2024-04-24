Joint US ATLAS / IRIS-HEP Kubernetes Hackathon
==============================================

This repository contains instructions and contents for GitOps Kubernetes deployments using Flux.
It assumes that you have set up a basic Kubernetes cluster.

NOTE: all the commands in this part of the tutorial should be run from your tutorial host (e.g., `cXXX.af`)

0.  Parts of this section will require pushing commits to a shared GitHub repository.
    This is fully possible through the GitHub web editor but can quickly become tedious with the more files that you
    have to edit, so we suggest cloning the repository to your local machine and preparing credentials to push to the
    shared repository.
    See the following documents for a suggested setup:

    -   [Cloning GitHub repositories](https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository)
    -   [Adding an SSH key to GitHub](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)

1.  Clone this repository to your *tutorial* host

        git clone https://github.com/osg-htc/k8s-hackathon-training

2.  [Install Flux components](manifests/flux-install)

3.  [Bootstrap Flux syncing](clusters#bootstrap-flux) with this repository

4.  [Install the Sealed Secrets](manifests/sealed-secrets) operator
    and replace the Flux SSH key secret with a sealed secret

6.  Create `osg` and `usatlas` [namespaces](clusters/README-namespaces.md#)
