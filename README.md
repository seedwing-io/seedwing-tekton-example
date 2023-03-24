# seedwing-tekton-example

Example of using Seedwing Policy in a Tekton pipeline.

## prerequisites

* A Kubernetes cluster
* Cosign

## Setting up Tekton

If you have not already set up Tekton, install pipelines and chains components:

``` 4d
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.42.0/release.yaml
kubectl apply --filename https://storage.googleapis.com/tekton-releases/chains/previous/v0.14.0/release.yaml
```

## Generate signing keys

This will generate keys used to sign artifacts:

``` 4d
cosign generate-key-pair k8s://tekton-chains/signing-secrets
```

## Configure chains

```
kubectl patch configmap chains-config -n tekton-chains -p='{"data":{"artifacts.taskrun.format": "in-toto"}}'
kubectl patch configmap chains-config -n tekton-chains -p='{"data":{"artifacts.taskrun.storage": "oci"}}'
kubectl patch configmap chains-config -n tekton-chains -p='{"data":{"transparency.enabled": "true"}}'
kubectl delete po -n tekton-chains -l app=tekton-chains-controller
```

## Configure Kaniko build task

``` 4d
kubectl apply -f https://raw.githubusercontent.com/tektoncd/chains/main/examples/kaniko/kaniko.yaml
```

## Setup registry

You can use an existing OCI registry, or use a local one using `podman`.

### Running a local registry

Make sure you can access the registry host from within your Kubernetes cluster.

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

Modify the environment variables to match your environment

``` 4d
export REGISTRY=my-registry-host:5000
export REGISTRY_USER=testuser
podman login $REGISTRY -u $REGISTRY_USER --authfile config.json
kubectl create secret generic registry-credentials --type=kubernetes.io/dockerconfigjson --from-file=.dockerconfigjson=config.json --from-file=config.json
kubectl create -f tasks
kubectl create -f pipelines
```
