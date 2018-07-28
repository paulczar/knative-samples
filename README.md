# Knative Samples

These were built using Pivotal Container Service (PKS) but should work roughly the same way on any Kubernetes cluster.

## Install Knative

Install istio:

```bash
  kubectl apply -f install/istio.yaml
  kubectl label namespace default istio-injection=enabled
```

Install knative:

```bash
  kubectl apply -f install/knative.yaml
```

## Use Knative

The examples here deploy the Spring Pet Clinic example application.

* [Run a prebuild image](run/README.md)
* [Build and Run from Source](src-to-run/README.md)
* [Only use the Build service](knative-build-only/README.md)
