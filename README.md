# Setup Knative for Development with Minikube

## Setup Minikube

```
minikube start
```

In a new terminal run
```
minikube tunnel
```

## Install Istio (Lean)

```bash
kubectl apply -f https://raw.githubusercontent.com/knative/serving/master/third_party/istio-1.4.2/istio-crds.yaml
```

```bash
kubectl apply -f https://raw.githubusercontent.com/knative/serving/master/third_party/istio-1.4.2/istio-minimal.yaml
```

```bash
kubectl apply -f https://raw.githubusercontent.com/knative/serving/master/third_party/istio-1.4.2/istio-knative-extras.yaml
```

Verify Istio is Running
```bash
kubectl get pods --namespace istio-system -w
```

Output should be:
```
NAME                                     READY   STATUS      RESTARTS   AGE
cluster-local-gateway-866d94b5f5-2ht7h   1/1     Running     0          17m
istio-ingressgateway-54589b686-bhcmz     2/2     Running     0          19m
istio-init-crd-10-1.4.2-dr4zb            0/1     Completed   0          20m
istio-init-crd-11-1.4.2-dks6s            0/1     Completed   0          20m
istio-init-crd-14-1.4.2-5p62d            0/1     Completed   0          20m
istio-pilot-7b5967465c-gfhrf             1/1     Running     0          19m
```

Get the `EXTERNAL-IP` for the istio-ingressgateway
```bash
kubectl get svc istio-ingressgateway -n istio-system
```

Or to get only the IP value run:
```bash
kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

It will print the IP Address, for example
```
10.96.176.21
```

### Install Knative


Select the version of Knative Serving to install
```bash
export KNATIVE_VERSION="0.11.0"
```

Install crds
```bash
kubectl apply --filename https://github.com/knative/serving/releases/download/v$KNATIVE_VERSION/serving-crds.yaml
```

Install the controller
```bash
kubectl apply \
--selector networking.knative.dev/certificate-provider!=cert-manager \
--filename https://github.com/knative/serving/releases/download/v$KNATIVE_VERSION/serving.yaml
```

Verify that app pods for Knative serving are Running
```
kubectl get pods --namespace knative-serving -w
```

Output should be:
```
NAME                                READY   STATUS    RESTARTS   AGE
activator-7db6679666-fwtxh          1/1     Running   0          8m15s
autoscaler-ffc9f79b4-qtpgv          1/1     Running   0          8m15s
autoscaler-hpa-5994dfdb67-tfbrr     1/1     Running   0          8m15s
controller-6797f99458-9qxql         1/1     Running   0          8m14s
networking-istio-85484dc749-fnc2p   1/1     Running   0          8m14s
webhook-6f97457cbf-sxxxq            1/1     Running   0          8m14s
```



## Configure Knative

Setup domain name to use the External IP Address of the ingressgateway above

```bash
export KNATIVE_DOMAIN="10.96.176.21.nip.io"
```

```bash
kubectl patch configmap -n knative-serving config-domain -p "{\"data\": {\"$KNATIVE_DOMAIN\": \"\"}}"
```

## Deploy Knative Application

```bash
cat <<EOF | kubectl apply -f -
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  template:
    spec:
      containers:
        - image: gcr.io/knative-samples/helloworld-go
          env:
            - name: TARGET
              value: "Go Sample v1"
EOF
```


Verify status of Knative Service
```bash
kubectl get ksvc
```

Output should be:
```
NAME    URL                                        LATESTCREATED   LATESTREADY   READY   REASON
hello   http://hello.default.10.96.176.21.nip.io   hello-kpkxt     hello-kpkxt   True
```

Note: If the service is not ready and RevisionMissing then wait a minute or two.


Test the App
```bash
curl http://hello.default.10.96.176.21.nip.io
```

Output should be
```
Hello Go Sample v1!
```

Check the knative pods that scaled from zero
```
kubectl get pod -l serving.knative.dev/service=hello
```

Output should be
```
NAME                                      READY   STATUS    RESTARTS   AGE
hello-kpkxt-deployment-78c9b8c9cf-zrht8   2/2     Running   0          6s
```