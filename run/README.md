# Use Knative Serving to run a prebuilt image

This example shows the easiest way to run an application with Knative Serving.

## Pre Requisites

* You should have a Kubernetes cluster with Knative [already installed](../README.md)
* The image that you want to run in an accessible docker registry.

> Note: this demo uses the `paulczar/spring-petclinic` image that is already up on the Docker Registry and should just work.

## Configuring a deployment

Deploying an app with KNative requires that you request a knative serving service. This is done with a Kubernetes manifest that looks like this:

```yaml
apiVersion: serving.knative.dev/v1alpha1 # Current version of Knative
kind: Service
metadata:
  name: petclinic-run # The name of the app
  namespace: default # The namespace the app will use
spec:
  runLatest:
    configuration:
      revisionTemplate:
        spec:
          container:
            image: docker.io/paulczar/spring-petclinic # image to run

```

> Note: You can change the `metdata.name` or `container.image` if you want to run your own application.

## Deploy Petclinic

The required manifest can be found in the `run/` subdirectory so you can simply point `kubectl apply` at it like so:


```bash
$ kubectl apply -f run/petclinic.yaml
service "petclinic-run" created
```

Upon receiving your manifest Knative will:

* Create a new revision of your app.
* Create a route, ingress, service, and loadbalancer for your app.
* Scale your application up and down (to zero!) based on traffic.

## Access Petclinic

Knative manages an ingress that is utilized by all of the apps it manages, you can find the IP Address of this ingress by running `kubectl get svc knative-ingressgateway -n istio-system`. You can save this ouput as an environment variable to use later:

```bash
$ export IP=$(kubectl get svc knative-ingressgateway -n istio-system -o 'jsonpath={.status.loadBalancer.ingress[0].ip}')
```

You also need to find the URL for your service which knative constructs for you

```bash
$ export URL=$(kubectl get services.serving.knative.dev petclinic-run  -o jsonpath='{.status.domain}')
```

You can then access Petstore with CURL by running:

```bash
$ curl -H "Host: ${URL}" http://${IP}
<!DOCTYPE html>
<html>
  <head>
...
...
```

> Note: By setting the `Host:` header you fake out needing to use the URL itself.  If you want to access Petclinic via a web browser you will need to add the IP and URL to your hosts file [/etc/hosts].

## Cleanup

Delete the Petclinic App:

```
kubectl delete -f run/petclinic.yaml
```