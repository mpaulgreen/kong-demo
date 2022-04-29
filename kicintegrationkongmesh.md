- Deploy the mesh first

- In Progress deployment steps of kic
```
helm repo add kong https://charts.konghq.com
helm repo update

oc new-project kong
oc adm policy add-scc-to-group nonroot system:serviceaccounts:kong
oc annotate namespace kong kuma.io/sidecar-injection=enabled
oc policy add-role-to-group system:image-puller system:serviceaccounts:kong --namespace=kong-image-registry

helm install kong kong/kong -n kong \
--set ingressController.installCRDs=false \
--set ingressController.image.tag=2.3.1 \
--set podAnnotations."kuma\.io/mesh"=default \
--set podAnnotations."kuma\.io/gateway"=enabled


oc get pod -n kong
oc get service -n kong
oc expose service kong-kong-proxy -n kong

oc adm policy add-scc-to-group anyuid system:serviceaccounts:kuma-demo
oc policy add-role-to-group system:image-puller system:serviceaccounts:kuma-demo --namespace=kong-image-registry
oc apply -f kic/sample.yaml
oc apply -f ~/kuma-demo/kubernetes/kuma-demo-aio.yaml
```

```
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: route1
  namespace: kuma-demo
  annotations:
    konghq.com/strip-path: "true"
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - http:
      paths:
        - path: /route1
          pathType: Prefix
          backend:
            service:
              name: sample
              port:
                number: 5000
EOF
```

```
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-app-ingress
  namespace: kuma-demo
  annotations:
    konghq.com/strip-path: "true"
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - http:
      paths:
      - path: /demo-app
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 8080 
EOF
```

```
oc get ingress -n kuma-demo
oc describe ingress route1 -n kuma-demo
http kong-kong-proxy-kong.apps.mpkongdemo.51ty.p1.openshiftapps.com/route1/hello
http kong-kong-proxy-kong.apps.mpkongdemo.51ty.p1.openshiftapps.com/demo-app
```

- Uninstall 
```
oc delete ingress route1
oc delete ingress demo-app-ingress
oc delete -f kic/sample.yaml 
oc policy remove-role-from-group system:image-puller system:serviceaccounts:kuma-demo --namespace=kong-image-registry
oc adm policy remove-scc-from-group anyuid system:serviceaccounts:kuma-demo
helm uninstall kong -n kong
oc delete project kong
kubectl delete -f https://bit.ly/kong-ingress-enterprise
```


