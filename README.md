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

3.  Push your new branch to GitHub:

        git push origin YOUR_BRANCH_NAME

    Replacing `YOUR_BRANCH_NAME` with the branch name that you chose in the previous step.

Setting Up Flux
---------------

In this tutorial we will follow the manual installation procedure for [Flux](https://fluxcd.io/),
a GitOps Kubernetes integration that relies on [Kustomize](https://kustomize.io/).
Kustomize is Kubernetes' built-in configuration management designed to tackle the repetitive nature of Kubernetes objects. 
If you're familiar with Puppet or other configuration management systems, Kustomize has the ability to "deep merge"
manifests along with other means to stand up multiple instances of a service (e.g., different CEs that share a common
base).

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

After installing the Sealed Secrets operator, your Kubernetes cluster is bootstrapped and all ready for GitOps!
From here on out, deploying services is as simple as pushing commits to your GitHub repository (getting k8s syntax and
references right is another story – `kubectl -n <namespace> describe ks` is your friend!).


### Create an admin-managed namespace ###

First, let's create a namespace for services that you'll administer but want to cordone into their own area for security
or organizational reasons.
For example, you may want to run the KAPEL + Gratia integration in an `osg` namespace.

1.  Create a `clusters/uchicago/tenants/osg` directory

2.  Create the namespace object in `clusters/uchicago/tenants/osg/namespace.yaml`:

        ---
        apiVersion: v1
        kind: Namespace
        metadata:
          name: osg

3.  Create a `clusters/uchicago/tenants/osg/kustomization.yaml` that references the namespace file:

        ---
        namespace: osg

        resources:
          - namespace.yaml

4.  Add a reference to your newly created namespace in the cluster-level `clusters/uchicago/kustomization.yaml`:

        resources:
          ...
          - tenants/osg

5.  Commit all these changes to GitHub and check for the namespace or errors in Kustomization:

        kubectl get ns
        kubectl -n flux-system describe ks


### Creating an operator-managed namespace ###

The above will generate a namespace that anyone with access to the GitHub repository can manage.
However, there may be cases where you'll want to allow other people to manage services within your cluster without
having access to the rest of the cluster (e.g., US ATLAS Squid or XCache operations).
To solve this, we can set up another layer of namespace-specific Flux and use a different GitHub repository that our
theoretical operators will have write access to

#### Prepare the operator Git repository ####

1.  Clone this repository to the login host:

        cd ~
        git clone ssh://git@github.com/osg-htc/k8s-hackathon-ops

2.  This is a shared repository so create a Git branch for your work (remember your branch name!):

        cd k8s-hackathon-ops
        git checkout -b YOUR_BRANCH_NAME main

    Replacing `YOUR_BRANCH_NAME` with a branch name of your choice, e.g. your `login05` username.
    Remember this branch name, you'll need it in future steps!

3.  Push your new branch to GitHub:

        git push origin YOUR_BRANCH_NAME

    Replacing `YOUR_BRANCH_NAME` with the branch name that you chose in the previous step.


#### Setting up Flux ####

1.  Go back to the cluster configuration repository, `k8s-hackathon-training`.
    Depending on your directory layout, that may look something like:
    
        cd ~/k8s-hackathon-training

2.  Repeat the [above namespace creation directions](#create-an-admin-managed-namespace), replacing `osg` with `usatlas`

3.  In the `cluters/uchicago/tenants/usatlas` directory, create a `service-account.yaml` file to create a Kubernetes
    ServiceAccount to allow Flux to manage the namespace:

        ---
        apiVersion: v1
        kind: ServiceAccount
        metadata:
          labels:
            toolkit.fluxcd.io/tenant: usatlas
          name: usatlas-flux
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: Role
        metadata:
          name: flux
        rules:
          - apiGroups:
              - '*'
            resources:
              - '*'
            verbs:
              - '*'
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: RoleBinding
        metadata:
          name: flux
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: Role
          name: flux
        subjects:
          - name: usatlas-flux
            kind: ServiceAccount

4.  Create a Sealed Secret for the deploy key associated with the `k8s-hackathon-ops` GitHub repository:

        flux create secret git deploy-key --url=ssh://git@github.com/osg-htc/k8s-hackathon-ops \
                                          --private-key-file=/usr/local/tutorial/gitops/k8s-hackathon-ps.key
                                          --export \
        | yq '.metadata.namespace="usatlas"' -o json \
        | kubeseal --cert clusters/uchicago/sealed-secrets.pem -o yaml \
        > clusters/uchicago/tenants/usatlas/gh-deploy-key.yaml

4.  Add a reference to `deploy-key.yaml` to `clusters/uchicago/tenants/usatlas/kustomization.yaml` and commit
    `gh-deploy-key.yaml` along with your changes to `kustomization.yaml`.
    When you're done, it should look like the following:

            ---
            namespace: usatlas

            resources:
              - gh-deploy-key.yaml
              - namespace.yaml

4.  Set up the Kustomization and GitRepository object in `clusters/uchicago/tenants/usatlas/sync.yaml`, replacing the
    value of `branch` with the branch name that you chose for the `k8s-hackathon-ops` repo above:

        ---
        apiVersion: source.toolkit.fluxcd.io/v1
        kind: GitRepository
        metadata:
          annotations:
          name: k8s-hackathon-ops
          namespace: usatlas
        spec:
          interval: 1m0s
          ref:
            branch: main
          secretRef:
            name: deploy-key
          timeout: 60s
          url: ssh://git@github.com/osg-htc/k8s-hackathon-ops.git
        ---
        apiVersion: kustomize.toolkit.fluxcd.io/v1
        kind: Kustomization
        metadata:
          name: flux-system
          namespace: usatlas
        spec:
          interval: 1m0s
          path: ./clusters/uchicago/usatlas-ops
          prune: true
          sourceRef:
            kind: GitRepository
            name: k8s-hackathon-ops

    Then add the file reference to the resources list in the `clusters/uchicago/tenants/usatlas/kustomization.yaml`.


What Next?
----------

At this stage, you're done setting up your Kubernetes cluster to support GitOps at the cluster-level and in a namespace
dedicated to a specific group!

From here, you can attempt to put previous services that you set up manually into Flux.
For example, you can take the "Hello, World!" application from part 1 of the tutorial and add it to the
`k8s-hackathon-ops` GitHub repository to deploy it to the `usatlas` namespace!
