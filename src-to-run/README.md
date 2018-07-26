# Use Knative Serving to build a docker image from a git repo

This example shows using Knative's Serving and Build tooling to do the following:

1. Download your source code from Github
1. Push the image to your Docker Registry
1. Run your Application in Kubernetes

## Pre Requisites

1. You should have a Kubernetes cluster with Knative [already installed](../README.md)
2. The Kaniko build template

### Installing the Kaniko build template

The manifest found in [/install/kaniko.yaml](../install/kaniko.yaml) will install the kaniko build template:

```
$ k apply -f install/kaniko.yaml
buildtemplate "kaniko" configured
```

## Configuring the deployment

Knative's Build needs write access to your github registry to push the built image to it. To do this we create a `secret` resource and a `service-account` that uses that secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-user-pass
  annotations:
    build.knative.dev/docker-0: https://index.docker.io/v1/
type: kubernetes.io/basic-auth
data:
  # Use 'echo -n "username" | base64' to generate this string
  username: BASE64_ENCODED_USERNAME
  # Use 'echo -n "password" | base64' to generate this string
  password: BASE64_ENCODED_PASSWORD
```

> Note: The values of `username` and `password` should be `base64` encoded. The above shows the commands you should run to do so.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot
secrets:
- name: basic-user-pass
```

These files already exist in the `src-to-run` path so edit them there to add your encoded username and password. You can then run:

```
$ kubectl apply -f src-to-run/secret.yaml
secret "basic-user-pass" created
$ kubectl apply -f src-to-run/account.yaml
serviceaccount "build-bot" created
```

Knative's Serving knows how to call Build when you need your application built. This is done by adding a build configuration to your manifest that looks like the following snippet:

```yaml
      build:
        serviceAccountName: build-bot
        source:
          git:
            url: https://github.com/paulczar/spring-petclinic.git
            revision: master
        template:
          name: kaniko
          arguments:
          - name: IMAGE
            value: docker.io/paulczar/spring-petclinic:latest

```

> Note: You can find the full manifest in [src-to-run/petclinic.yaml](src-to-run/petclinic.yaml)

## Deploy Petclinic

The required manifest can be found in the `src-to-run/` subdirectory so you can simply point `kubectl apply` at it like so:

```bash
$ kubectl apply -f src-to-run/petclinic.yaml
service "petclinic-src-to-run" created
```

Upon receiving your manifest Knative will:

* Clone your source from github
* Build a docker image using the `/Dockerfile` in your github repo
* Push the docker image to your docker registry
* Create a new revision of your app.
* Create a route, ingress, service, and loadbalancer for your app.
* Scale your application up and down (to zero!) based on traffic.

You can watch the build in progress (once it starts) by doing the following:

```
k get pods
NAME                               READY     STATUS     RESTARTS   AGE
petclinic-src-to-run-00001-lzqjj   0/1       Init:2/3   0          2m

$ kubectl logs -f petclinic-src-to-run-00001-lzqjj -c build-step-build-and-push
time="2018-07-26T20:24:37Z" level=info msg="Unpacking filesystem of maven:3.5-jdk-8-alpine..."
time="2018-07-26T20:24:37Z" level=info msg="Mounted directories: [/kaniko /var/run /proc /dev /dev/pts /sys /sys/fs/cgroup /sys/fs/cgroup/cpuset /sys/fs/cgroup/cpu /sys/fs/cgroup/cpuacct /sys/fs/cgroup/blkio /sys/fs/cgroup/memory /sys/fs/cgroup/devices /sys/fs/cgroup/freezer /sys/fs/cgroup/net_cls /sys/fs/cgroup/perf_event /sys/fs/cgroup/net_prio /sys/fs/cgroup/hugetlb /sys/fs/cgroup/pids /dev/mqueue /workspace /builder/home /dev/termination-log /etc/resolv.conf /etc/hostname /etc/hosts /dev/shm /var/run/secrets/kubernetes.io/serviceaccount /proc/bus /proc/fs /proc/irq /proc/sys /proc/sysrq-trigger /proc/kcore /proc/timer_list /proc/timer_stats /proc/sched_debug /proc/scsi /sys/firmware]"
```

## Access Petclinic

> Note: It will take a considerable amount of time before its ready to receive traffic, but eventually the Petclinic deployment will be complete and accessible.

Knative manages an ingress that is utilized by all of the apps it manages, you can find the IP Address of this ingress by running `kubectl get svc knative-ingressgateway -n istio-system`. You can save this ouput as an environment variable to use later:

```bash
$ export IP=$(kubectl get svc knative-ingressgateway -n istio-system -o 'jsonpath={.status.loadBalancer.ingress[0].ip}')
```

You also need to find the URL for your service which knative constructs for you

```bash
$ export URL=$(kubectl get services.serving.knative.dev petclinic-src-to-run  -o jsonpath='{.status.domain}')
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
kubectl delete -f src-to-run/petclinic.yaml
```