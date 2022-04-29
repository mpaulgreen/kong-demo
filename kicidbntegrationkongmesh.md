# Deploy the mesh
  - Download Kong Mesh
  ```
  curl -L https://docs.konghq.com/mesh/installer.sh | sh -
  ```

  - Install control plane on kong-mesh-system namespace

  ```
  ./kumactl install control-plane --cni-enabled --license-path=./license | oc apply -f -
  oc get pod -n kong-mesh-system
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

# Install demo-app
  ```  
  oc adm policy add-scc-to-group anyuid system:serviceaccounts:kuma-demo
  
  oc apply -f kic/sample.yaml
  oc apply -f ~/kuma-demo/kubernetes/kuma-demo-aio.yaml
  ```



# Deploy kong gateway control plane
  - Create kong project
  ```
  oc new-project kong
  ```

  - Create the secrets for license key
  ```
  kubectl create secret generic kong-enterprise-license -n kong --from-file=./license
  ```

  - Find the Opsnshift project scc uid
  ```
  oc describe project kong # uid will be used in the helm chart
  ```

  - Generating Private Key and Digital Certificate
  ```
  openssl req -new -x509 -nodes -newkey ec:<(openssl ecparam -name secp384r1) \
    -keyout ./cluster.key -out ./cluster.crt \
    -days 1095 -subj "/CN=kong_clustering"

  kubectl create secret tls kong-cluster-cert --cert=./cluster.crt --key=./cluster.key -n kong
  ```

  - Creating session conf for Kong Manager and Kong DevPortal
  ```
  echo '{"cookie_name":"admin_session","cookie_samesite":"off","secret":"kong","cookie_secure":false,"storage":"kong"}' > admin_gui_session_conf
  echo '{"cookie_name":"portal_session","cookie_samesite":"off","secret":"kong","cookie_secure":false,"storage":"kong"}' > portal_session_conf
  kubectl create secret generic kong-session-config -n kong --from-file=admin_gui_session_conf --from-file=portal_session_conf
  ```

  - Creating Kong Manager password
  ```
  kubectl create secret generic kong-enterprise-superuser-password -n kong --from-literal=password=kong
  ```

  - Deploy the chart
  ```
  helm install kong kong/kong -n kong \
  --set env.database=postgres \
  --set env.password.valueFrom.secretKeyRef.name=kong-enterprise-superuser-password \
  --set env.password.valueFrom.secretKeyRef.key=password \
  --set env.role=control_plane \
  --set env.cluster_cert=/etc/secrets/kong-cluster-cert/tls.crt \
  --set env.cluster_cert_key=/etc/secrets/kong-cluster-cert/tls.key \
  --set cluster.enabled=true \
  --set cluster.tls.enabled=true \
  --set cluster.tls.servicePort=8005 \
  --set cluster.tls.containerPort=8005 \
  --set clustertelemetry.enabled=true \
  --set clustertelemetry.tls.enabled=true \
  --set clustertelemetry.tls.servicePort=8006 \
  --set clustertelemetry.tls.containerPort=8006 \
  --set image.repository=kong/kong-gateway \
  --set image.tag=2.8.0.0-alpine \
  --set admin.enabled=true \
  --set admin.http.enabled=true \
  --set admin.type=NodePort \
  --set proxy.enabled=true \
  --set ingressController.enabled=true \
  --set ingressController.installCRDs=false \
  --set ingressController.image.repository=kong/kubernetes-ingress-controller \
  --set ingressController.image.tag=2.2.1 \
  --set ingressController.env.kong_admin_token.valueFrom.secretKeyRef.name=kong-enterprise-superuser-password \
  --set ingressController.env.kong_admin_token.valueFrom.secretKeyRef.key=password \
  --set ingressController.env.enable_reverse_sync=true \
  --set ingressController.env.sync_period="1m" \
  --set postgresql.enabled=true \
  --set postgresql.postgresqlUsername=kong \
  --set postgresql.postgresqlDatabase=kong \
  --set postgresql.postgresqlPassword=kong \
  --set postgresql.securityContext.runAsUser=1000830000 \
  --set postgresql.securityContext.fsGroup= \
  --set enterprise.enabled=true \
  --set enterprise.license_secret=kong-enterprise-license \
  --set enterprise.rbac.enabled=true \
  --set enterprise.rbac.session_conf_secret=kong-session-config \
  --set enterprise.rbac.admin_gui_auth_conf_secret=admin-gui-session-conf \
  --set enterprise.smtp.enabled=false \
  --set enterprise.portal.enabled=true \
  --set manager.enabled=true \
  --set manager.type=NodePort \
  --set portal.enabled=true \
  --set portal.http.enabled=true \
  --set env.portal_gui_protocol=http \
  --set portal.type=NodePort \
  --set portalapi.enabled=true \
  --set portalapi.http.enabled=true \
  --set portalapi.type=NodePort \
  --set secretVolumes[0]=kong-cluster-cert
  ```

  - Create the routes
  ```
  oc expose service kong-kong-admin -n kong
  oc expose service kong-kong-manager -n kong
  oc expose service kong-kong-portal -n kong
  oc expose service kong-kong-portalapi -n kong
  ```

  - Checking the Admin API
  ```
  http kong-kong-admin-kong.apps.mpkongdemo.51ty.p1.openshiftapps.com kong-admin-token:kong | jq -r .version
  ```

  - Configuring Kong Manager Service
  ```
  kubectl patch deployment -n kong kong-kong -p "{\"spec\": { \"template\" : { \"spec\" : {\"containers\":[{\"name\":\"proxy\",\"env\": [{ \"name\" : \"KONG_ADMIN_API_URI\", \"value\": \"kong-kong-admin-kong.apps.mpkongdemo.51ty.p1.openshiftapps.com\" }]}]}}}}"
  ```

  - Configuring Kong Dev Portal
  ```
  kubectl patch deployment -n kong kong-kong -p "{\"spec\": { \"template\" : { \"spec\" : {\"containers\":[{\"name\":\"proxy\",\"env\": [{ \"name\" : \"KONG_PORTAL_API_URL\", \"value\": \"http://kong-kong-portalapi-kong.apps.mpkongdemo.51ty.p1.openshiftapps.com\" }]}]}}}}"
  kubectl patch deployment -n kong kong-kong -p "{\"spec\": { \"template\" : { \"spec\" : {\"containers\":[{\"name\":\"proxy\",\"env\": [{ \"name\" : \"KONG_PORTAL_GUI_HOST\", \"value\": \"kong-kong-portal-kong.apps.mpkongdemo.51ty.p1.openshiftapps.com\" }]}]}}}}"
  ```

# Deploy kong gateway data plane plane

  ```
  oc new-project kong-dp
  oc adm policy add-scc-to-group nonroot system:serviceaccounts:kong-dp
  oc annotate namespace kong-dp kuma.io/sidecar-injection=enabled
  kubectl create secret generic kong-enterprise-license -n kong-dp --from-file=./license
  kubectl create secret tls kong-cluster-cert --cert=./cluster.crt --key=./cluster.key -n kong-dp

  helm install kong-dp kong/kong -n kong-dp \
  --set ingressController.enabled=false \
  --set image.repository=kong/kong-gateway \
  --set image.tag=2.8.0.0-alpine \
  --set env.database=off \
  --set env.role=data_plane \
  --set env.cluster_cert=/etc/secrets/kong-cluster-cert/tls.crt \
  --set env.cluster_cert_key=/etc/secrets/kong-cluster-cert/tls.key \
  --set env.lua_ssl_trusted_certificate=/etc/secrets/kong-cluster-cert/tls.crt \
  --set env.cluster_control_plane=kong-kong-cluster.kong.svc.cluster.local:8005 \
  --set env.cluster_telemetry_endpoint=kong-kong-clustertelemetry.kong.svc.cluster.local:8006 \
  --set proxy.enabled=true \
  --set proxy.type=NodePort \
  --set enterprise.enabled=true \
  --set enterprise.license_secret=kong-enterprise-license \
  --set enterprise.portal.enabled=false \
  --set enterprise.rbac.enabled=false \
  --set enterprise.smtp.enabled=false \
  --set manager.enabled=false \
  --set portal.enabled=false \
  --set portalapi.enabled=false \
  --set env.status_listen=0.0.0.0:8100 \
  --set secretVolumes[0]=kong-cluster-cert \
  --set podAnnotations."kuma\.io/mesh"=default \
  --set podAnnotations."kuma\.io/gateway"=enabled
  ```

# Test the gateway and service
  - Checking the Data Plane from the Control Plane
  ```
  http kong-kong-admin-kong.apps.mpkongdemo.51ty.p1.openshiftapps.com/clustering/status kong-admin-token:kong
  ```
  - Expose the proxy
  ```
  oc expose service kong-dp-kong-proxy
  ```

  - Checking the Proxy
  ```
  http kong-dp-kong-proxy-kong-dp.apps.mpkongdemo.51ty.p1.openshiftapps.com
  ```

  - Defining a Service and a Route

  ```
  http kong-kong-admin-kong.apps.mpkongdemo.51ty.p1.openshiftapps.com/services name=sampleservice url='http://sample.kuma-demo.svc.cluster.local:5000' kong-admin-token:kong
  http kong-kong-admin-kong.apps.mpkongdemo.51ty.p1.openshiftapps.com/services/sampleservice/routes name='httpbinroute' paths:='["/sample"]' kong-admin-token:kong
  ```

  - Test the service and route
  ```
  http kong-dp-kong-proxy-kong-dp.apps.mpkongdemo.51ty.p1.openshiftapps.com/sample/hello
  while [ 1 ]; do curl kong-dp-kong-proxy-kong-dp.apps.mpkongdemo.51ty.p1.openshiftapps.com/sample/hello; echo; done
  ```


  - Ingress Controller Policies
```
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sampleroute
  namespace: kuma-demo
  annotations:
    konghq.com/strip-path: "true"
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - http:
      paths:
        - path: /sampleroute
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
  oc describe ingress sampleroute -n kuma-demo
  oc describe ingress demo-app-ingress -n kuma-demo
  http kong-dp-kong-proxy-kong-dp.apps.mpkongdemo.51ty.p1.openshiftapps.com/sampleroute/hello
  http kong-dp-kong-proxy-kong-dp.apps.mpkongdemo.51ty.p1.openshiftapps.com/demo-app
  ```
- Uninstall 
```
oc delete ingress sampleroute -n kuma-demo
oc delete ingress demo-app-ingress -n kuma-demo
oc delete -f kic/sample.yaml 
oc delete -f ~/kuma-demo/kubernetes/kuma-demo-aio.yaml
oc adm policy remove-scc-from-group anyuid system:serviceaccounts:kuma-demo
helm uninstall kong -n kong
helm uninstall kong-dp -n kong-dp
oc delete project kong
oc delete project kong-dp
kubectl delete -f https://bit.ly/kong-ingress-enterprise
```


