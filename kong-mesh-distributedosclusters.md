# Install control plane on kong-mesh-system namespace
```
./kumactl install control-plane --cni-enabled --mode=global --license-path=./license | oc apply -f -
```

- Expose the service
```{bash}
oc expose svc/kong-mesh-control-plane -n kong-mesh-system --port http-api-server
```

- Verify the Installation
```{bash}
$ http -h `oc get route kong-mesh-control-plane -n kong-mesh-system -ojson | jq -r .spec.host`/gui/
HTTP/1.1 200 OK
cache-control: private
content-length: 295
content-type: application/json
date: Tue, 05 Apr 2022 14:54:17 GMT
set-cookie: 559045d469a6cf01d61b4410371a08e0=35991f4fa47508ce861e82fa7d63d40a; path=/; HttpOnly
```

- Setting the mTLS on
```
cat <<EOF | oc apply -f -
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  mtls:
    enabledBackend: ca-1
    backends:
    - name: ca-1
      type: builtin
EOF
```
```
GLOBAL_SYNC_HOST=$(oc -n kong-mesh-system get service --output=json | jq -r ".items[1].status.loadBalancer.ingress[0].hostname")

GLOBAL_SYNC_PORT=$(oc -n kong-mesh-system get service --output=json | jq -r ".items[1].spec.ports[0].port")

GLOBAL_REMOTE_SYNC=${GLOBAL_SYNC_HOST}:${GLOBAL_SYNC_PORT}

echo $GLOBAL_REMOTE_SYNC
```
# Data Plane 1 installation
```
./kumactl install control-plane --cni-enabled --mode=zone --zone=dp1 --ingress-enabled --license-path=./license  --kds-global-address grpcs://a8cdfbb57304f47bf9a9acb7b67a1ece-1503957949.ca-central-1.elb.amazonaws.com:5685 | oc apply -f -
```

# Data Plane 2 installation
```
./kumactl install control-plane --cni-enabled --mode=zone --zone=dp2 --ingress-enabled --license-path=./license  --kds-global-address grpcs://a8cdfbb57304f47bf9a9acb7b67a1ece-1503957949.ca-central-1.elb.amazonaws.com:5685 | oc apply -f -
```

# Data Plane 3 installation
```
./kumactl install control-plane --cni-enabled --mode=zone --zone=dp3 --ingress-enabled --license-path=./license  --kds-global-address grpcs://a8cdfbb57304f47bf9a9acb7b67a1ece-1503957949.ca-central-1.elb.amazonaws.com:5685 | oc apply -f -
```
# Deploy Microservices(magnanimo) in Data Plane 1
```
oc new-project kuma-app
oc adm policy add-scc-to-group nonroot system:serviceaccounts:kuma-app
oc annotate namespace kuma-app kuma.io/sidecar-injection=enabled
```

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: magnanimo
  namespace: kuma-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: magnanimo
  template:
    metadata:
      annotations:
        kuma.io/gateway: enabled
      labels:
        app: magnanimo
    spec:
      containers:
      - name: magnanimo
        image: claudioacquaviva/magnanimo_kuma
        ports:
        - containerPort: 4000
---
apiVersion: v1
kind: Service
metadata:
  name: magnanimo
  namespace: kuma-app
  annotations:
    ingress.kubernetes.io/service-upstream: "true"
  labels:
    app: magnanimo
spec:
  type: ClusterIP
  ports:
    - port: 4000
      name: http
  selector:
    app: magnanimo
EOF

```

```
oc expose service magnanimo --port 4000 -n kuma-app
http http://magnanimo-kuma-app.apps.dp1.51ty.p1.openshiftapps.com/hello

```

# Deploy Microservices(Benigno - v1) in Data Plane 2

```
oc new-project kuma-app
oc adm policy add-scc-to-group nonroot system:serviceaccounts:kuma-app
oc annotate namespace kuma-app kuma.io/sidecar-injection=enabled
```

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: benigno-v1
  namespace: kuma-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: benigno
  template:
    metadata:
      labels:
        app: benigno
        version: v1
    spec:
      containers:
      - name: benigno
        image: claudioacquaviva/benigno
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: benigno
  namespace: kuma-app
  labels:
    app: benigno
spec:
  type: ClusterIP
  ports:
    - port: 5000
      name: http
  selector:
    app: benigno
EOF

```

- From Data Plane1 call magnanimo service that in turn calls benigno service
```
http http://magnanimo-kuma-app.apps.dp1.51ty.p1.openshiftapps.com/hw3
```

# Deploy Microservices(Benigno - v2) in Data Plane 3
```
oc new-project kuma-app
oc adm policy add-scc-to-group nonroot system:serviceaccounts:kuma-app
oc annotate namespace kuma-app kuma.io/sidecar-injection=enabled
```

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: benigno
  namespace: kuma-app
  labels:
    app: benigno
spec:
  type: ClusterIP
  ports:
  - port: 5000
    name: http
  selector:
    app: benigno
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: benigno-v2
  namespace: kuma-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: benigno
  template:
    metadata:
      labels:
        app: benigno
        version: v2
    spec:
      containers:
      - name: benigno
        image: claudioacquaviva/benigno_rc
        ports:
        - containerPort: 5000
EOF
```

# Validating canary release from control plane
```
while [ 1 ]; do curl http://magnanimo-kuma-app.apps.dp1.51ty.p1.openshiftapps.com/hw3; sleep 1; echo; done
```

# Implement Traffic Route in control plane
```
cat <<EOF | kubectl apply -f -
apiVersion: kuma.io/v1alpha1
kind: TrafficRoute
mesh: default
metadata:
  namespace: kuma-app
  name: magnanimo-benigno
spec:
  sources:
    - match:
        kuma.io/service: magnanimo_kuma-app_svc_4000
  destinations:
    - match:
        kuma.io/service: benigno_kuma-app_svc_5000
  conf:
    split:
      - weight: 10
        destination:
          kuma.io/service: benigno_kuma-app_svc_5000
          version: v1
      - weight: 90
        destination:
          kuma.io/service: benigno_kuma-app_svc_5000
          version: v2
EOF

```

# Apply traffic permission in control plane

- Delete the ```allow-all-default``` trafficpermission
```
kubectl delete trafficpermission allow-all-default
```

- Apply the new trafficpermission(This will make all traffic to v2 of beningo fail)
```
cat <<EOF | kubectl apply -f -
apiVersion: kuma.io/v1alpha1
kind: TrafficPermission
mesh: default
metadata:
  namespace: kuma-app
  name: magnanimo-benigno
spec:
  sources:
    - match:
        kuma.io/service: magnanimo_kuma-app_svc_4000
  destinations:
    - match:
        kuma.io/service: benigno_kuma-app_svc_5000
        version: v1
EOF

```
- If you want to allow the communication with both releases again run
```
cat <<EOF | kubectl apply -f -
apiVersion: kuma.io/v1alpha1
kind: TrafficPermission
mesh: default
metadata:
  namespace: kuma-app
  name: magnanimo-benigno
spec:
  sources:
    - match:
        kuma.io/service: magnanimo_kuma-app_svc_4000
  destinations:
    - match:
        kuma.io/service: benigno_kuma-app_svc_5000
EOF
```
