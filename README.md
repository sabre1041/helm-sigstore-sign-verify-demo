# helm-sigstore-sign-verify-demo

Demonstration of packaging and signing [Helm](https://helm.sh/) charts using tools such as [Tekton](https://tekton.dev/) and [sigstore](https://sigstore.dev/).

## Deployment

[Kustomize](https://kustomize.io) is used to deploy resources to a Kubernetes environment. Use the following steps to facilitate the deployment.


### Prerequisites

1. [Tekton](https://tekton.dev/) must be deployed to the environment. Follow [these steps](https://tekton.dev/docs/getting-started/) to install Tekton to the Kubernetes environment.
2. A [GitHub Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) must be available to enable publishing the release to GitHub.
3. PGP based encryption is used to sign the Helm chart. A sample public and private keypair is provided. Alternatively, a user provided keypair can be used. No additional action is needed when using the provided keypair.
4. [rekor-cli](https://github.com/sigstore/rekor/releases), [helm](https://github.com/helm/helm/releases) and of course [kubectl](https://github.com/kubernetes/kubectl/releases) will need to be installed locally to execute this demo.

### Release Pipeline

A Tekton pipeline is available to perform the packaging and signing of a Helm chart. Use the following steps to deploy to a Kubernetes. 

1. Fork this repository or use a repository of your own (Additional configurations required)
2. Add your GitHub Personal Access Token to [install/release/base/github-token](install/release/base/github-token)
3. Deploy resources to your cluster:

Depending on whether you are providing PGP keys of your own will determine which command to execute. If you are using the default PGP keys, execute the following command:

```shell
kubectl apply -k install/release/overlays/default
```

If you are providing PGP keys of your own, execute the following:

```shell
kubectl apply -k install/release/overlays/pgp-provided
```

A new namespace called `helm-release` will be created containing all of the resources.

### Add PGP Keys (Optional)

If you providing PGP keys of your own, execute the following command:

```shell
kubectl create secret generic -n helm-release --from-file=private.pgp=<file>
```

## Execution

Once the assets have been deployed to the cluster, a `PipelineRun` resource is available in the `install/release/base/tekton/pipelineruns/pipelinerun.yaml` directory. Apply any modifications to suite the location of your repository and configuration and then execute the following command to add it to your cluster and start the Pipeline.

```shell
kubectl create -f install/release/base/tekton/pipelineruns/pipelinerun.yaml
```

You can monitor the pipeline by using the [Tekton CLI](https://github.com/tektoncd) (`tkn`).

```shell
tkn -n helm-demo pipelinerun logs -L -f
```

Once complete, the following actions will be performed:

1. Publishing a new GitHub Release
2. Tag the repository for the chart release
3. Update the `index.yaml` file in the `gh-pages` branch
4. Using the helm sigstore plugin to upload the chart to Rekor.

## Verification

The following steps can be used to verify the chart

1. Add the chart repository to interact with the published chart (be sure to replace `user_or_organization`)

```shell
helm repo add helm-sigstore-sign-verify-demo https://<user_or_organization>.github.io/helm-sigstore-sign-verify-demo
```

2. Update the local chart cache

```shell
helm repo update
```

2. Pull the chart locally (along with the provenance file)

```shell
helm pull helm-sigstore-sign-verify-demo/podinfo --prov
```

3. Verify the chart with Helm

Note: If the provided PGP key was used to sign the chart, the [public key](install/release/overlays/default/) must be imported into the keyring before executing the following command

```shell
helm verify --keyring=<keyring> <packaged_chart>
```

4. Verify the chart using the sigstore helm plugin

Install the sigstore helm plugin

```shell
helm plugin install https://github.com/sigstore/helm-sigstore
```

Verify the chart

Note: The public key does not need to be added to the keyring. It can be explicitly specified using the `--key` parameter

```shell
helm sigstore verify --keyring=<keyring> <packaged_chart>
```

## Viewing the record in the transparency log

The record published in the Rekor transparency log can be show by first obtaining the UUID from the Tekton `PipelineRun` results and then querying Rekor.

Obtain the UUID

```shell
REKOR_UUID=$(oc -n helm-demo get pipelinerun $(kubectl -n helm-demo get pipelinerun --sort-by=.metadata.creationTimestamp -o jsonpath="{.items[-1:].metadata.name}") -o jsonpath='{ .status.pipelineResults[?(@.name=="sigstore-rekor-uuid")].value }')
```

View the UUID:

```shell
echo $REKOR_UUID
```

Use the [Rekor CLI](https://github.com/sigstore/rekor) to view the record:

```shell
rekor-cli get --uuid=$REKOR_UUID
```
