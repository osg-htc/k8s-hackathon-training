Joint US ATLAS / IRIS-HEP Kubernetes Hackathon
==============================================

This repository contains instructions and contents for GitOps Kubernetes deployments using Flux.
It assumes that you have set up a basic Kubernetes cluster by completing the first phase of the tutorial.

NOTE: all the commands in this part of the tutorial should be run from the login host (e.g., `login05`)

Setting Up the Git Repository
-----------------------------

Parts of this tutorial will require pushing commits to a shared GitHub repository.
This is fully possible through the GitHub web editor but quickly becomes tedious with the more files that have to
edit.
If you don't already have an SSH key associated with your GitHub account, set one up on the login host using the
following instructions:

-   [Generate an SSH key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
-   [Add the SSH key to GitHub](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)

1.  Clone this repository to the login host:

        git clone ssh://git@github.com/osg-htc/k8s-hackathon-training

2.  This is a shared repository so create a Git branch for your work (remember your branch name!):

        cd k8s-hackathon-training
        git checkout -b YOUR_BRANCH_NAME main

    Replacing `YOUR_BRANCH_NAME` with a branch name of your choice, e.g. your `login05` username.
    Remember this branch name, you'll need it in future steps!

3.  

Setting Up Flux
---------------

In this tutorial we will follow the manual installation procedure for [Flux](https://fluxcd.io/),
a GitOps Kubernetes integration that relies on [Kustomize](https://kustomize.io/).
Kustomize is Kubernetes' built-in configuration management designed to tackle the repetitive nature of Kubernetes objects. 
If you're familiar with Puppet or other configuration management systems, Kustomize has similar features to "deep merge"
manifests along with other means to stand up multiple instances of a service (e.g., different CEs that share a common base).

Normally Flux is installed onto a Kubernetes cluster with its `flux bootstrap` command.
If you're interested in additional details, see the [standard Flux installation](manifests/flux-install#standard-flux-installation)
section for the procedure and drawbacks.


### Installing Flux components ###

This step installs the necessary Flux components definitions, namespace, etc:
Set up of the cluster-level Flux instance (Kustomization, GitRepository, and SSH secret objects) will come in a later step.

From the root of the Git repository on the login host, run Kustomize manually to install the Flux components:

```
kubectl apply -k manifests/flux-install
```


### Starting a cluster-level Flux ###

In this step, we will manually create the Flux Kustomization and GitRepository objects to manage your entire cluster.

1.  First, set up the SSH secret associated with this GitHub repository's deploy key:

        flux create secret git deploy-key --url=ssh://git@github.com/osg-htc/k8s-hackathon-training \
                                          --private-key-file=/usr/local/tutorial/gitops/k8s-hackathon-training.key

2.  Change the `branch` field in `clusters/uchicago/bootstrap/sync.yaml` from `main` to the branch that you chose [above](#setting-up-the-git-repository).

3.  Commit your changes to the GitHub repository.

4.  Start Flux by running the following from the root directory of this repository:

        kubectl apply -k clusters/uchicago/bootstrap

6.  Ensure that your cluster is pulling updates from this GitHub repository:

        kubectl -n flux-system get gitrepo

You should see a line showing that the GitRepository object is ready, such as:

        NAME                     URL                                                       AGE    READY   STATUS        
        k8s-hackathon-training   ssh://git@github.com/osg-htc/k8s-hackathon-training.git   3m4s   True    stored artifact for revision 'main@sha1:ebdc88552e4c4889439f48dff208a49e9da9f42a'

7.  Check to see that Flux is successfully deploying Kubernetes objects based on the state of the GitHub repository:

        kubectl -n flux-system get ks

You should see a line showing that the Flux Kustomization object is ready, such as:

        NAME          AGE     READY   STATUS
        flux-system   5m33s   True    Applied revision: main@sha1:ebdc88552e4c4889439f48dff208a49e9da9f42a


Installing Sealed Secrets
-------------------------

Kubernetes secret objects are base64-encoded (i.e., not encrypted) and are therefore not safe to store in Git.
This presents a problem to GitOps deployments as private credentials are critial to functioning services.
Enter the [Sealed Secrets operator](https://github.com/bitnami-labs/sealed-secrets) that can take encrypted secrets and
turn them into decrypted Kubernetes secrets.

1.  To install the Sealed Secrets operator, create `clusters/uchicago/kustomization.yaml` with the following contents and
    commit this change to the GitHub repository:

        resources:
          - ../../manifests/sealed-secrets

2.  Verify that the Sealed Secrets operator has started:

        $ kubectl -n kube-system get deploy/sealed-secrets-controller
        NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
        sealed-secrets-controller   1/1     1            1           7m

3.  Fetch the your Sealed Secret public certificate and commit it to GitHub:

        kubeseal --fetch-cert > clusters/uchicago/sealed-secrets.pem

4.  Encrypt the GitHub deploy key secret:

        kubectl -n flux-system get secret/deploy-key -o json \
            | kubeseal --cert clusters/uchicago/sealed-secrets.pem -o yaml \
            > clusters/uchicago/gh-deploy-key.yaml

5.  Add the sealed secret to the GitHub repository and reference it in `clusters/uchicago/kustomization.yaml`:

        resources:
          - ../../manifests/sealed-secrets
          - gh-deploy-key.yaml

6.  Once the SealedSecret object is created, remove the manually created secret and the sealed secret in case it's
    throwing the following error `Resource "deploy-key" already exists and is not managed by SealedSecret`:

        kubectl -n flux-system delete secret/deploy-key
        kubectl -n flux-system delete sealedsecret/deploy-key

    Don't worry, Flux will recreate it!


Creating Namespaces
-------------------

6.  Create `osg` and `usatlas` [namespaces](clusters/README-namespaces.md#)


What Next?
----------

At this stage, you're done setting up your Kubernetes cluster to support DevOps integration

Take the "Hello, World!" application from part 1 of the tutorial and add it to the `k8s-hackathon-ops` GitHub repository
in step (6) to deploy it to the `usatlas` namespace!
