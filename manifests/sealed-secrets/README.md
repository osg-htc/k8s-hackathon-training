Install Sealed Secrets
======================

Kubernetes secret objects are base64-encoded (i.e., not encrypted) and are therefore not safe to store in Git.
This presents a problem to GitOps deployments as private credentials are critial to functioning services.
Enter the [Sealed Secrets operator](https://github.com/bitnami-labs/sealed-secrets) that can take encrypted secrets and
turn them into decrypted Kubernetes secrets.

1.  To install the Sealed Secrets operator, simply add a reference to `clusters/<YOUR USERNAME>/kustomization.yaml` and
    commit this change to the GitHub repository:

        resources:
          - ../../manifests/sealed-secrets

2.  Verify that the Sealed Secrets operator has started:

        $ kubectl -n kube-system get deploy/sealed-secrets-controller
        NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
        sealed-secrets-controller   1/1     1            1           7m

3.  Copy the public certificate to `clusters/<YOUR USERNAME>/sealed-secrets.pem` and commit it to GitHub:

        kubeseal --fetch-cert

4.  Encrypt the GitHub deploy key secret:

        kubectl -n flux-system get secret/deploy-key -o json \
            | kubeseal --cert sealed-secrets.pem -o yaml \
            > gh-deploy-key.yaml

5.  Add the sealed secret to the GitHub repository and reference it in `clusters/<YOUR USERNAME>/kustomization.yaml`:

        resources:
          - ../../manifests/sealed-secrets
          - gh-deploy-key.yaml

6.  Once the SealedSecret object is created, remove the manually created secret and the sealed secret in case it's
    throwing the following error `Resource "deploy-key" already exists and is not managed by SealedSecret`:

        kubectl -n flux-system delete secret/deploy-key
        kubectl -n flux-system delete sealedsecret/deploy-key

    Don't worry, Flux will recreate it!
