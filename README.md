Joint US ATLAS / IRIS-HEP Kubernetes Hackathon
==============================================

This repository contains instructions and contents for GitOps Kubernetes deployments using Flux.
It assumes that you have set up a basic Kubernetes cluster by completing the first phase of the tutorial.

NOTE: all the commands in this part of the tutorial should be run from the login host (e.g., `login05`)

0.  Parts of this section will require pushing commits to a shared GitHub repository.
    This is fully possible through the GitHub web editor but quickly becomes tedious with the more files that have to
    edit, so you should create a new SSH key and associate it with your GitHub account:
    See the following documents:

    -   [Generate an SSH key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
    -   [Adding an SSH key to GitHub](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)

1.  Clone this repository to the login host:

        git clone ssh://git@github.com/osg-htc/k8s-hackathon-training

2.  [Install Flux components](manifests/flux-install)

3.  [Bootstrap Flux syncing](clusters#bootstrap-flux) with this repository

4.  [Install the Sealed Secrets](manifests/sealed-secrets) operator
    and replace the Flux SSH key secret with a sealed secret

6.  Create `osg` and `usatlas` [namespaces](clusters/README-namespaces.md#)
