# Demo using a streaming processor
Scripts for running a riff streaming demo

## Software prerequisites

Have the following installed:

- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) version used: v1.16.2
- [kapp](https://github.com/k14s/kapp#kapp) version used v0.15.0
- [ytt](https://github.com/k14s/ytt#ytt-yaml-templating-tool) version used v0.22.0

# Kubernetes cluster

Follow the riff instructions for:

- [GKE](https://projectriff.io/docs/v0.4/getting-started/gke)
- [Minikube](https://projectriff.io/docs/v0.4/getting-started/minikube)
- [Docker for Mac](https://projectriff.io/docs/v0.4/getting-started/docker-for-mac)

> NOTE: kapp can't install keda on a Kubernetes cluster running version 1.16 so we need to force the Kubernetes version to be 1.14 or 1.15

## Clone the demo repo

Clone this repo:

```
git clone https://github.com/trisberg/riff-streaming-demo.git
cd riff-streaming-demo
```

## Install riff

Install riff and all dependent packages including cert-manager, kpack, keda, riff-build, istio and core, knative and serving runtimes:

For a cluster that supports LoadBalancer use:

```
./riff-kapp-install.sh
```

For a cluster like "Minikube" or "Docker for Mac" that doesn't support LoadBalancer use:
```
./riff-kapp-install.sh --node-port
```

## Install kafka

```
kapp deploy -y -n apps -a kafka -f https://storage.googleapis.com/projectriff/charts/uncharted/0.5.0-snapshot/kafka.yaml
```

## Create kafka-provider

```
riff streaming kafka-provider create franz --bootstrap-servers kafka.kafka:9092
```

## Create hello function

```
riff function create hello --git-repo https://github.com/projectriff-samples/node-hello.git --artifact hello.js --tail
```

## Create streams

```
riff streaming stream create in --namespace default --provider franz-kafka-provisioner --content-type 'text/plain'
riff streaming stream create out --namespace default --provider franz-kafka-provisioner --content-type 'text/plain'
```

## Create stream processor

```
riff streaming processor create hello --function-ref hello --namespace default --input in --output out --tail
```

## Run dev-utils pod

```
kubectl create serviceaccount dev-utils --namespace default
kubectl create rolebinding dev-utils --namespace default --clusterrole=view --serviceaccount=default:dev-utils
kubectl run dev-utils --image=projectriff/dev-utils:latest --generator=run-pod/v1 --serviceaccount=dev-utils
```

## Subscribe to out stream and write to text file

(open a new terminal window)

```
kubectl exec dev-utils -n default -- subscribe default_out -n default --payload-as-string > result.txt
```

## Publish a few messages on the in stream

```
kubectl exec dev-utils -n default -- publish default_in -n default --content-type "text/plain" --payload "riff #1"
kubectl exec dev-utils -n default -- publish default_in -n default --content-type "text/plain" --payload "riff #2"
kubectl exec dev-utils -n default -- publish default_in -n default --content-type "text/plain" --payload "riff #3"
```

## Check the results

```
cat result.txt
```

## Experimental: use the native image for the streaming-processor

Update the configmap for the streaming-processor before creating the processor:

```
kubectl patch configmap/riff-streaming-processor -n riff-system --type merge \
  -p '{"data":{"processorImage": "index.docker.io/trisberg/streaming-processor-native@sha256:ea96f38f6e86d0fbcc963c464ea7b657948da446ec55576d782896c776f590ba"}}'
```

## Teardown demo

```
riff streaming processor delete hello
riff streaming stream delete in
riff streaming stream delete out
riff function delete hello
```

## Teardown kafka

```
kapp delete -y -n apps -a kafka
```

## Teardown dev-utils

```
kubectl delete pod -n default dev-utils
```

## Teardown riff

```
./riff-kapp-delete.sh
```
