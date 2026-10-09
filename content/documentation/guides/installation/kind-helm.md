---
draft: false
title: "On Kind with Helm"
date: 2024-04-30
publishdate: 2024-04-30
lastmod: 2024-06-13
weight: 4
---

## Overview 

This guide will walk you through the different steps of running a full Microcks installation on your laptop using [Kind](https://kind.sigs.k8s.io). Step #4 is actually optional and may only be of interest if you'd like to use the Asynchronous features of Microcks.

The installation notes were run on an Apple MacBook M2, but those steps would sensibly be the same on any Linux machine. 

Let's go 🚀

## 1. Preparation

As being on a Mac, people usually use [brew](https://brew.sh) to install `kind`. However, it is also available from several different package managers out there. You can check the [Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) guide for that. Obviously, you'll also need the [`kubectl`](https://kubernetes.io/docs/tasks/tools/#kubectl) utility to interact with your cluster. 

```sh
  brew install kind
  kind --version
```

```sh
kind version 0.33.0
```

In a dedicated folder, prepare a `cluster-kind.yaml` configuration file like this:

```sh
  cd ~/tmp
  mkdir microcks && cd microcks
  cat > cluster-kind.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 80
        listenAddress: "127.0.0.1"
        protocol: TCP
      - containerPort: 30443
        hostPort: 443
        listenAddress: "127.0.0.1"
        protocol: TCP
EOF
```

## 2. Start and configure a cluster

We're now going to start a Kube cluster. Start your `kind` cluster using the `cluster-kind.yaml` configuration file we just created before:

```sh
  kind create cluster --config=cluster-kind.yaml
```

```sh
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.37.0) 🖼️ 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
``` 

Check that the node is available:

```sh
  kubectl get nodes
```

```sh
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   75s   v1.37.0
```

## 3. Install Envoy Gateway

Envoy Gateway implements Kubernetes Gateway API and provides the proxy that will route traffic to Microcks.

Install Envoy Gateway using its Helm chart:

```sh
  helm install envoy-gateway oci://docker.io/envoyproxy/gateway-helm -n envoy-gateway-system --create-namespace
```

```sh
NAME: envoy-gateway
LAST DEPLOYED: Fri Oct  9 15:02:08 2026
NAMESPACE: envoy-gateway-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
**************************************************************************
*** PLEASE BE PATIENT: Envoy Gateway may take a few minutes to install ***
**************************************************************************

Envoy Gateway is an open source project for managing Envoy Proxy as a standalone or Kubernetes-based application gateway.

Thank you for installing Envoy Gateway! 🎉

Your release is named: envoy-gateway. 🎉

Your release is in namespace: envoy-gateway-system. 🎉

To learn more about the release, try:

  $ helm status envoy-gateway -n envoy-gateway-system
  $ helm get all envoy-gateway -n envoy-gateway-system

To have a quickstart of Envoy Gateway, please refer to https://gateway.envoyproxy.io/latest/tasks/quickstart.

To get more details, please visit https://gateway.envoyproxy.io and https://github.com/envoyproxy/gateway.
```

Wait for the deployment to become ready:

```sh
  kubectl wait \
  --namespace envoy-gateway-system \
  --for=condition=Available \
  deployment/envoy-gateway \
  --timeout=120s
```

This installation also installs the Gateway API CRDs.

Create a GatewayClass:

```sh
  cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
EOF
```

```sh
gatewayclass.gateway.networking.k8s.io/envoy created
```

Verify that the GatewayClass has been accepted:

```sh
  kubectl get gatewayclass envoy
```

```sh
NAME    CONTROLLER                                      ACCEPTED   AGE
envoy   gateway.envoyproxy.io/gatewayclass-controller   True       22s
```

Create the Microcks namespace:

```sh
  kubectl create namespace microcks
```

Configure an Envoy for the Kind cluster:

```sh
  cat <<EOF | kubectl apply -f -
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: microcks-proxy
  namespace: microcks
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        type: NodePort
        patch:
          type: StrategicMerge
          value:
            spec:
              ports:
                - port: 443
                  nodePort: 30443
EOF
```

Create two HTTPS listeners on port 443, one for Microcks and one for Keycloak. Both listeners use the same certificate, which covers both hostnames.

```sh
  cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: microcks-gateway
  namespace: microcks
spec:
  gatewayClassName: envoy

  infrastructure:
    parametersRef:
      group: gateway.envoyproxy.io
      kind: EnvoyProxy
      name: microcks-proxy

  listeners:
    - name: microcks-https
      hostname: microcks.127.0.0.1.nip.io
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: microcks-local-tls
      allowedRoutes:
        namespaces:
          from: Same

    - name: keycloak-https
      hostname: keycloak.127.0.0.1.nip.io
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: microcks-local-tls
      allowedRoutes:
        namespaces:
          from: Same
EOF
```

## 4. Create a local TLS certificate

Generates TLS certificates for Gateway API listeners:

```sh
  openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout microcks-local.key \
  -out microcks-local.crt \
  -subj "/CN=microcks.127.0.0.1.nip.io" \
  -addext "subjectAltName=DNS:microcks.127.0.0.1.nip.io,DNS:keycloak.127.0.0.1.nip.io"
```

```sh
  kubectl create secret tls microcks-local-tls \
  --namespace microcks \
  --key=microcks-local.key \
  --cert=microcks-local.crt
```

Check that the secret is created:

```sh
  kubectl get secret microcks-local-tls -n microcks
```

```sh

NAME                 TYPE                DATA   AGE
microcks-local-tls   kubernetes.io/tls   2      20s
```

Note that the certificate is self-signed, so the browser will display a security warning until you trust it.

## 5. Install Microcks 

We're now going to install Microcks with basic options. We'll do that using the Helm Chart, so you'll also need the [`helm`](https://helm.sh) binary. You can use `brew install helm` on Mac for that.

Add the Microcks Helm repository:

```sh
  helm repo add microcks https://microcks.io/helm/
  helm repo update
```

Install Microcks with Gateway API routes enabled and Ingress resources disabled:

```sh
  helm install microcks microcks/microcks \
  --namespace microcks \
  --create-namespace \
  --set ingresses=false \
  --set gatewayRoutes=true \
  --set gatewayRefName=microcks-gateway \
  --set gatewayRefSectionName=microcks-https \
  --set microcks.url=microcks.127.0.0.1.nip.io \
  --set microcks.gatewayRefSectionName=microcks-https \
  --set microcks.generateCert=false \
  --set keycloak.url=keycloak.127.0.0.1.nip.io \
  --set keycloak.privateUrl=http://microcks-keycloak.microcks.svc.cluster.local:8080 \
  --set keycloak.gatewayRefName=microcks-gateway \
  --set keycloak.gatewayRefSectionName=keycloak-https \
  --set keycloak.generateCert=false
```

```sh

NAME: microcks
LAST DEPLOYED: Fri Oct  9 22:41:23 2026
NAMESPACE: microcks
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing microcks.

Your release is named microcks.

To learn more about the release, try:

  $ helm status microcks
  $ helm get microcks

Microcks is available at https://microcks.127.0.0.1.nip.io.

GRPC mock service is available at "microcks-grpc.127.0.0.1.nip.io".
It has been exposed using TLS passthrough on the Ingress controller, you should extract the certificate for your client using:

  $ kubectl get secret microcks-microcks-grpc-secret -n microcks -o jsonpath='{.data.tls\.crt}' | base64 -d > tls.crt

Keycloak has been deployed on https://keycloak.127.0.0.1.nip.io to protect user access.
You may want to configure an Identity Provider or add some users for your Microcks installation by login in using the
username and password found into 'microcks-keycloak-admin' secret.
```

Wait for the pods to become ready:

```sh
  kubectl get pods -n microcks
```

```sh

NAME                                            READY   STATUS    RESTARTS   AGE
microcks-68657ccb64-l9zkx                       1/1     Running   0          118s
microcks-keycloak-7c995cbf6d-xtb69              1/1     Running   0          118s
microcks-keycloak-postgresql-6dbb9744df-56plm   1/1     Running   0          118s
microcks-mongodb-66c77b5dbd-wjf5f               1/1     Running   0          118s
microcks-postman-runtime-566748c78-f7mzk        1/1     Running   0          118s
```

Inspect the generated routes:

```sh
  kubectl get httproute -n microcks
```

```sh

NAME                HOSTNAMES                       AGE
microcks            ["microcks.127.0.0.1.nip.io"]   2m16s
microcks-keycloak   ["keycloak.127.0.0.1.nip.io"]   2m16s
```

Start opening `https://keycloak.127.0.0.1.nip.io/` in your browser to validate the self-signed certificate. Once done, you can visit `https://microcks.127.0.0.1.nip.io/` in your browser, validate the self-signed certificate and start playing around with Microcks!

The default user/password is `admin/microcks123`

## 6. Install Microcks with asynchronous options

TODO!!!

In this section, we're installing Microcks completely and enabling the asynchronous protocols feature. This requires deploying additional pods and a Kafka cluster. Microcks installs and manages its own cluster using the [Strimzi](https://strimzi.io) project.

To be able to expose the Kafka cluster to the outside of Kind, you’ll need to enable SSL passthrough on nginx: This requires updating the default ingress controller deployment:

```sh
  kubectl patch -n ingress-nginx deployment/ingress-nginx-controller --type='json' \
    -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--enable-ssl-passthrough"}]'
```

Then, you have to install the latest version of Strimzi, which provides an easy way to set up Kafka on Kubernetes:
```sh
  kubectl apply -f 'https://strimzi.io/install/latest?namespace=microcks' -n microcks
```

Now, you can install Microcks using the Helm chart and enable the asynchronous features:

```sh
  helm install microcks microcks/microcks --namespace microcks --set microcks.url=microcks.127.0.0.1.nip.io --set keycloak.url=keycloak.127.0.0.1.nip.io --set keycloak.privateUrl=http://microcks-keycloak.microcks.svc.cluster.local:8080 --set features.async.enabled=true --set features.async.kafka.url=kafka.127.0.0.1.nip.io
```
```sh

NAME: microcks
LAST DEPLOYED: Sun Dec  3 20:14:38 2023
NAMESPACE: microcks
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing microcks.

Your release is named microcks.

To learn more about the release, try:

    helm status microcks
    helm get microcks

Microcks is available at https://microcks.127.0.0.1.nip.io.

GRPC mock service is available at "microcks-grpc.127.0.0.1.nip.io".
It has been exposed using TLS passthrough on the Ingress controller, you should extract the certificate for your client using:

    kubectl get secret microcks-microcks-grpc-secret -n microcks -o jsonpath='{.data.tls\.crt}' | base64 -d > tls.crt

Keycloak has been deployed on https://keycloak.127.0.0.1.nip.io to protect user access.
You may want to configure an Identity Provider or add some users for your Microcks installation by login in using the
username and password found into 'microcks-keycloak-admin' secret.

Kafka broker has been deployed on microcks-kafka.kafka.127.0.0.1.nip.io.
It has been exposed using TLS passthrough on the Ingress controller, you should extract the certificate for your client using:

    kubectl get secret microcks-kafka-cluster-ca-cert -n microcks -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
```

Watch and check the pods you should get in the namespace:

```sh
  kubectl get pods -n microcks
```
```sh

NAME                                              READY   STATUS    RESTARTS        AGE
microcks-6ffcc7dc54-c9h4w                         1/1     Running   0               68s
microcks-async-minion-7f689d9ff7-ptv4c            1/1     Running   2 (40s ago)     48s
microcks-kafka-entity-operator-585dc4cd45-24tvp   3/3     Running   0               2m19s
microcks-kafka-kafka-0                            1/1     Running   0               2m41s
microcks-kafka-zookeeper-0                        1/1     Running   5 (4m56s ago)   6m43s
microcks-keycloak-77447d8957-fwhv6                1/1     Running   0               87s
microcks-keycloak-postgresql-868b7dbdd4-pb52g     1/1     Running   0               2m43s
microcks-mongodb-78888fb67f-7t2vf                 1/1     Running   4 (3m57s ago)   8m2s
microcks-postman-runtime-857c577dfb-d597r         1/1     Running   0               8m2s
strimzi-cluster-operator-95d88f6b5-p8bvs          1/1     Running   0               16m
```

Now you can extract the Kafka cluster certificate using `kubectl get secret microcks-kafka-cluster-ca-cert -n microcks -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt` and apply the checks found at [Async Features with Docker Compose](https://microcks.io/blog/async-features-with-docker-compose/)

Start with loading the [User signed-up API](https://microcks.io/blog/async-features-with-docker-compose/#load-a-sample-and-check-up) sample within your Microcks instance - remember that you have to validate the self-signed certificates like in the basic install first.

Now connect to the Kafka broker pod to check that a topic has been correctly created and that you can consume messages from there:


```sh
  kubectl -n microcks exec microcks-kafka-kafka-0 -it -- /bin/sh
```
```sh
--- INPUT ---
sh-4.4$ cd bin
sh-4.4$ ./kafka-topics.sh --bootstrap-server localhost:9092 --list
UsersignedupAPI-0.1.1-user-signedup
__consumer_offsets
microcks-services-updates

sh-4.4$ ./kafka-console-consumer.sh --bootstrap-server microcks-kafka-kafka-bootstrap:9092 --topic UsersignedupAPI-0.1.1-user-signedup
{"id": "eNc5TNaPlHAKa38XQA8N7HkSRHl7Yvm1", "sendAt": "1703699907417", "fullName": "Laurent Broudoux", "email": "laurent@microcks.io", "age": 41}
{"id":"g9uDUhXPOPtwK9bZYSGmqbxHAC3tTxAz","sendAt":"1703699907428","fullName":"John Doe","email":"john@microcks.io","age":36}
{"id": "kllBuhcv3kTRNg75sFxWH6HGLtSbpXwZ", "sendAt": "1703699917413", "fullName": "Laurent Broudoux", "email": "laurent@microcks.io", "age": 41}
{"id":"YE2ZAdVwSK9JLGEyLFebHxMOVfmYlzs1","sendAt":"1703699917426","fullName":"John Doe","email":"john@microcks.io","age":36}
^CProcessed a total of 4 messages
sh-4.4$ exit
exit
command terminated with exit code 130
```

And finally, from your Mac host, you can install the [`kcat`](https://github.com/edenhill/kcat) utility to consume messages as well. You'll need to refer to the `ca.crt` certificate you previously extracted from there:

```sh
  kcat -b microcks-kafka.kafka.127.0.0.1.nip.io:443 -X security.protocol=SSL -X ssl.ca.location=ca.crt -t UsersignedupAPI-0.1.1-user-signedup
```
```sh

% Auto-selecting Consumer mode (use -P or -C to override)
{"id": "zYcAzFlRoTGvu9Mu4ajg30lr1fBa4Kah", "sendAt": "1703699827456", "fullName": "Laurent Broudoux", "email": "laurent@microcks.io", "age": 41}
{"id":"v0TkDvd1Z7RxynQvi1i0NmXAaLPzuYXE","sendAt":"1703699827585","fullName":"John Doe","email":"john@microcks.io","age":36}
{"id": "JK55813rQ938Hj50JWXy80s5KWC61Uvr", "sendAt": "1703699837416", "fullName": "Laurent Broudoux", "email": "laurent@microcks.io", "age": 41}
{"id":"MZnR6UeKVXMhJET6asTjafPpfldiqXim","sendAt":"1703699837430","fullName":"John Doe","email":"john@microcks.io","age":36}
[...]
% Reached end of topic UsersignedupAPI-0.1.1-user-signedup [0] at offset 30
^C%  
```

## 6. Delete everything and stop the cluster

Deleting the microcks Helm release from your cluster is straightforward. Then you can finally delete the envoy gateway and your Kind cluster to save some resources!

```sh
  helm delete microcks -n microcks
```

```sh

Release "microcks" uninstalled
```

```sh
  helm delete envoy-gateway -n envoy-gateway-system
```

```sh

Release "envoy-gateway" uninstalled
```


```sh
  kind delete cluster
```

```sh

Deleting cluster "kind" ...
Deleted nodes: ["kind-control-plane"]
```

## Wrap-up

You've been through this guide and learned how to install Microcks on a Kubernetes cluster using Helm. Congrats! 🎉

If you'd like to learn more about all the available installation parameters, you can check our [Helm Chart Parameters](/documentation/references/configuration/helm-chart-config) reference documentation.

Happy learning!
