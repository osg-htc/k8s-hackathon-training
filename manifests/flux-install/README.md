Additional Flux Notes
=====================

Standard Flux Installation
--------------------------

In a standard installation, [Flux](https://fluxcd.io/) can be installed into a Kubernetes cluster with its first-class
boostrap support:

```
flux bootstrap --owner osg-htc \
               --repository k8s-hackathon-training \
               --token-auth=false \
               --private-key-file=$HOME/.ssh/github.key
```

However, this method has a few downsides:

1.  This method requires an administrative-level access token for the target Git repository (e.g., GitHub PAT)

2.  The resultant setup creates a running Flux within your cluster that makes Flux responsible for updating itself.

Updates
-------

To update Flux in the future, use the following procedure:

1.  Download the `flux` binary from <https://github.com/fluxcd/flux2/releases>

2.  Generate the necessary Flux Kubernetes objects as YAML that can then be added to Git. For example:

        flux install --namespace=flux-system \
                     --components-extra="image-reflector-controller,image-automation-controller" \ # image auto updates
                     --export \
                     > gotk-components-v2.2.3.yaml

3.  Add or modify patches in `kustomization.yaml` as necessary
    (e.g. prevent Flux from force pushing to the Git repository)

4.  Update Flux manually:

        kubectl apply -k .
