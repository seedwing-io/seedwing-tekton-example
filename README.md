# seedwing-tekton-example

This repository contains Tekton resources that demonstrate how to use Seedwing Policy in a pipeline. It provides resources to build and attach attestations, verify project-specific policies, and verify build attestation according to policies.

Before using this repository, you will need a Kubernetes cluster, the Cosign tool, and the tkn tool.

## Setting up Tekton

To set up Tekton, you need to install the pipeline and chains components by running these commands:

``` 4d
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.42.0/release.yaml
kubectl apply --filename https://storage.googleapis.com/tekton-releases/chains/previous/v0.14.0/release.yaml
```

## Generate signing keys

You also need to generate keys used to sign artifacts:

``` 4d
cosign generate-key-pair k8s://tekton-chains/signing-secrets
```

## Configure chains

To configure chains, run these commands:

```
kubectl create configmap tekton-policies --from-file=policies/
kubectl patch configmap chains-config -n tekton-chains -p='{"data":{"artifacts.taskrun.format": "in-toto"}}'
kubectl patch configmap chains-config -n tekton-chains -p='{"data":{"artifacts.taskrun.storage": "oci"}}'
kubectl patch configmap chains-config -n tekton-chains -p='{"data":{"transparency.enabled": "true"}}'
kubectl delete po -n tekton-chains -l app=tekton-chains-controller
```

## Setup registry

You can use an existing OCI registry or run a local registry using podman. If you choose to run a local registry, make sure you can access the registry host from within your Kubernetes cluster. Then run these commands:

``` 4d
mkdir -p auth
htpasswd -bBc auth/htpasswd testuser 1234
podman run -p 5000:5000 -e REGISTRY_AUTH=htpasswd -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd -v $PWD/auth:/auth:z -e REGISTRY_STORAGE_DELETE_ENABLED=true -ti registry:2

cat<<EOF > /etc/containers/registries.conf.d/001-testregistry.conf
[[registry]]
location = "my-registry-host:5000"
insecure = true
EOF
```

### Configuring the build pipeline

To configure the build pipeline, modify the environment variables to match your environment, and then run these commands:

``` 4d
export REGISTRY=my-registry-host:5000
export REGISTRY_USER=testuser
podman login $REGISTRY -u $REGISTRY_USER --authfile config.json
kubectl create secret generic registry-credentials --type=kubernetes.io/dockerconfigjson --from-file=.dockerconfigjson=config.json --from-file=config.json
kubectl create -f tasks
kubectl create -f pipelines
```

### Running a build

To trigger a build, create a pipeline run by running this command:

``` 4d
kubectl create -f quarkus-pipeline-run.yaml
```

This will fetch the source from git as configured in the YAML file and start the build. If the build completes successfully, it will create a container image and push it to the registry as configured in the pipeline run. 

The pipeline will then apply project-specific policies using the `swio` tool, before applying our policies.

To follow the progress of the pipeline, we can use the `tkn` tool:

``` 4d
tkn pipelinerun logs -f
```

This will show the logs as the different tasks get executed. Pay attention to the last task (`seedwing`), as it will fail if policy checks are not passed.
