# Using just the Build component of Knative

What if you just want to use the build capabilities of Knative without needing the complexity of Istio and the rest of Knative. Good news, it's totally possible... and pretty damn awesome.

## Install Knative Build

Deploy the Knative build components to the `knative-build` namespace:

```bash
$ kubectl apply -f install
namespace "knative-build" created
clusterrole "knative-build-admin" created
serviceaccount "build-controller" created
clusterrolebinding "build-controller-admin" created
customresourcedefinition "builds.build.knative.dev" created
customresourcedefinition "buildtemplates.build.knative.dev" created
service "build-controller" created
service "build-webhook" created
configmap "config-logging" created
deployment "build-controller" created
deployment "build-webhook" created
```

Check Knative is installed and ready:

```bash
$ kubectl -n knative-build get all
NAME                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/build-controller   1         1         1            1           25s
deploy/build-webhook      1         1         1            1           25s

NAME                             DESIRED   CURRENT   READY     AGE
rs/build-controller-5cb4f5cb67   1         1         1         25s
rs/build-webhook-6b4c65546b      1         1         1         25s

NAME                                   READY     STATUS    RESTARTS   AGE
po/build-controller-5cb4f5cb67-8vdh4   1/1       Running   0          25s
po/build-webhook-6b4c65546b-ww2gs      1/1       Running   0          25s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
svc/build-controller   ClusterIP   10.100.200.41   <none>        9090/TCP   25s
svc/build-webhook      ClusterIP   10.100.200.77   <none>        443/TCP    25s
```

## Build Petclinic

Deploy the components for building your app to the `petclinic-build` namespace:

```bash
$ kubectl apply -f build
namespace "build-petclinic" created
secret "build-petclinic" created
serviceaccount "build-petclinic" configured
buildtemplate "build-petclinic" configured
build "build-petclinic" created
```

Check on the Build:

```bash
$ kubectl -n build-petclinic get pods,build,buildtemplate
NAME                       READY     STATUS     RESTARTS   AGE
po/build-petclinic-prw8j   0/1       Init:2/3   0          17s

NAME                     AGE
builds/build-petclinic   17s

NAME                             AGE
buildtemplates/build-petclinic   53s
```

After a few minutes you should be able to see the build logs (use the pod name from above):

```bash
$ kubectl -n build-petclinic logs -f build-petclinic-prw8j -c build-step-build-and-push
```

## Run Petclinic

Run your freshly build Application:

```bash
$ kubectl apply -f run
deployment "petclinic" created
service "petclinic" created
```

After a short while it should be running and accessible:

```bash
k get all
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/petclinic   1         1         1            1           2m

NAME                    DESIRED   CURRENT   READY     AGE
rs/petclinic-78cddd7d   1         1         1         2m

NAME                          READY     STATUS    RESTARTS   AGE
po/petclinic-78cddd7d-hkglx   1/1       Running   0          2m

NAME             TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)        AGE
svc/kubernetes   ClusterIP      10.100.200.1     <none>          443/TCP        7d
svc/petclinic    LoadBalancer   10.100.200.232   35.192.99.186   80:30560/TCP   2m
```

Use the petclinic service's EXTERNAL-IP to access the application:

```bash
$ curl -s 35.192.99.186 | grep PetClinic
    <title>PetClinic :: a Spring Framework demonstration</title>

```

## Cleanup

```bash
$ kubectl delete -f run
deployment "petclinic" deleted
service "petclinic" deleted

$ kubectl delete -f build
namespace "build-petclinic" deleted
secret "build-petclinic" deleted
serviceaccount "build-petclinic" deleted
buildtemplate "build-petclinic" deleted
build "build-petclinic" deleted

$ kubectl delete -f install
namespace "knative-build" deleted
clusterrole "knative-build-admin" deleted
serviceaccount "build-controller" deleted
clusterrolebinding "build-controller-admin" deleted
customresourcedefinition "builds.build.knative.dev" deleted
customresourcedefinition "buildtemplates.build.knative.dev" deleted
service "build-controller" deleted
service "build-webhook" deleted
configmap "config-logging" deleted
deployment "build-controller" deleted
```