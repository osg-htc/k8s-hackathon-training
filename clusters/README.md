GitOps Setup
============

Bootstrap Flux
--------------

In this step, we will manually create the Flux Kustomization and GitRepository objects for your cluster.

1.  First, set up the SSH secret associated with this GitHub repository's deploy key:

        flux create secret git deploy-key --url=ssh://git@github.com/osg-htc/k8s-hackathon-training \
                                          --private-key-file=$HOME/.ssh/github.key

2.  Modify the `path` field in `clusters/<YOUR USERNAME>/bootstrap/sync.yaml` to `clusters/<YOUR USERNAME>`,
    replacing `<YOUR USERNAME>` with your UC AF username.
    This tells Flux which directory to look in for a `kustomization.yaml` that describe the objects in your cluster.

3.  Commit your changes to the GitHub repository using the GitHub web UI or your Git workflow of choice.
    N.B. this is a shared repository so you'll want to make sure to rebase any local repository copies frequently!

4.  From your tutorial host, pull in your updates:

        git pull origin

5.  Start Flux by running the following from `clusters/<YOUR USERNAME>/bootstrap`:

        kubectl apply -k .

6.  Ensure that your cluster is pulling updates from this GitHub repository:

        kubectl -n flux-system get gitrepo

7.  Check to see that Flux is succeeding to deploy Kubernetes objects based on the state of the GitHub repository:

        kubectl -n flux-system get ks


Creating Namespaces
-------------------

After installing the sealed secrets operator, your Kubernetes cluster is bootstrapped and all ready for GitOps!
From here on out, deploying services is as simple as pushing commits to your GitHub repository (getting k8s syntax and
references right is another story – `kubectl -n <namespace> describe ks` is your friend!).

First, let's create a namespace for services that you'll administer but want to cordone into their own area for security
or organizational reasons.
For example, you may want to run the KAPEL + Gratia integration in an `osg` namespace:

1.  Create a `clusters/<YOUR USERNAME>/tenants/osg` directory

2.  Create the namespace object in `clusters/<YOUR USERNAME>/tenants/osg/namespace.yaml`:

        ---
        apiVersion: v1
        kind: Namespace
        metadata:
          name: osg

3.  Create a `clusters/<YOUR USERNAME>/tenants/osg/kustomization.yaml` that references the namespace file:

        ---
        namespace: osg

        resources:
          - namespace.yaml

4.  Add a reference to your newly created namespace in the top-level `clusters/<YOUR USERNAME>/kustomization.yaml`:

        resources:
          ...
          - tenants/osg

5.  Commit all these changes to GitHub and check for the namespace or errors in Kustomization:

        kubectl get ns
        kubectl -n flux-system describe ks

This will generate a namespace that anyone with access to the GitHub repository can manage.
However, there may be cases where you'll want to allow other people to manage services within your cluster without
having access to the rest of the cluster (e.g., US ATLAS Squid or XCache operations).
To solve this, we can set up another layer of namespace-specific Flux:

1.  Repeat the above directions, replacing `osg` with `usatlas`

2.  In the `usatlas` directory, create a `service-account.yaml` file to create a Kubernetes ServiceAccount to allow Flux
    to manage the namespace:

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

3.  Prepare the namespace for GitHub integration.
    Note that we are cheating here by using the same GitHub repository and deploy key to simplify things.
    If you were to set this up in production, you would need to set up a new GitHub repository and upload the public
    part of the SSH key out-of-band.

    1.  Since we're using the same GitHub repository, copy the deploy key from the `flux-system` namespace:

            kubectl -n flux-system get secret/deploy-key -o json \
              | jq 'del(.metadata ["creationTimestamp", "ownerReferences", "uid", "resourceVersion"]) \
                    | .metadata.namespace="usatlas"' \
              | kubeseal --cert sealed-secrets.pem -o yaml \
              > tenants/usatlas/deploy-key.yaml

    2.  Create a `clusters/<YOUR USERNAME>/usatlas-ops/kustomization.yaml` with an empty resource list:

            ---
            resources: []

        This directory is where the other service operators would add their commits to deploy services to the `usatlas`
        namespace.

4.  Set up the Kustomization and GitRepository object in `clusters/<YOUR USERNAME>/tenants/usatlas/sync.yaml`:

        ---
        apiVersion: source.toolkit.fluxcd.io/v1
        kind: GitRepository
        metadata:
          annotations:
          name: k8s-hackathon-training
          namespace: usatlas
        spec:
          interval: 1m0s
          ref:
            branch: main
          secretRef:
            name: deploy-key
          timeout: 60s
          url: ssh://git@github.com/osg-htc/k8s-hackathon-training.git
        ---
        apiVersion: kustomize.toolkit.fluxcd.io/v1
        kind: Kustomization
        metadata:
          name: flux-system
          namespace: usatlas
        spec:
          interval: 1m0s
          path: ./clusters/<YOUR USERNAME>/usatlas-ops
          prune: true
          sourceRef:
            kind: GitRepository
            name: k8s-hackathon-training

    Then add the file reference to the resource list in the `clusters/<YOUR USERNAME>/tenants/usatlas/kustomization.yaml`.
