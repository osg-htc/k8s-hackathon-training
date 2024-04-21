Bootstrap Flux
==============

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
