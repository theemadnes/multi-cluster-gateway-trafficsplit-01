# multi-cluster-gateway-trafficsplit-01
playing around with GKE Gateway MCGs and traffic splitting

i'm setting this up across 2 existing GKE Autopilot clusters (`autopilot-cluster-us-central1` and `autopilot-cluster-us-east4`, and the context names match the cluster names). these clusters already have MCS / Gateway enabled on them, so if you want to learn about the basics of setup, go [here](https://github.com/theemadnes/multi-cluster-gateway-asm-01).

in this example, we're just going to demo traffic splitting across two versions (one named `flip`, the other `flop`) of [`whereami`](https://github.com/GoogleCloudPlatform/kubernetes-engine-samples/tree/main/whereami).

### deployment stuff

```
# create namespace
kubectl --context autopilot-cluster-us-central1 create ns whereami
kubectl --context autopilot-cluster-us-east4 create ns whereami

# create whereami base
mkdir whereami
mkdir whereami/base
cat <<EOF > whereami/base/kustomization.yaml 
resources:
  - github.com/GoogleCloudPlatform/kubernetes-engine-samples/whereami/k8s
EOF

# deploy `flip` on both clusters
mkdir whereami/flip-variant
cat <<EOF > whereami/flip-variant/cm-flag.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: whereami
data:
  BACKEND_ENABLED: "False" # assuming you don't want a chain of backend calls
  METADATA:        "flip"
EOF

cat <<EOF > whereami/flip-variant/service-type.yaml 
apiVersion: "v1"
kind: "Service"
metadata:
  name: "whereami"
spec:
  type: ClusterIP
EOF

cat <<EOF > whereami/flip-variant/kustomization.yaml 
nameSuffix: "-flip"
namespace: whereami
commonLabels:
  app: whereami-flip
resources:
- ../base
patches:
- path: cm-flag.yaml
  target:
    kind: ConfigMap
- path: service-type.yaml
  target:
    kind: Service
EOF

kubectl --context=autopilot-cluster-us-central1 apply -k whereami/flip-variant
kubectl --context=autopilot-cluster-us-east4 apply -k whereami/flip-variant

# deploy `flop` on both clusters
mkdir whereami/flop-variant
cat <<EOF > whereami/flop-variant/cm-flag.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: whereami
data:
  BACKEND_ENABLED: "False" # assuming you don't want a chain of backend calls
  METADATA:        "flop"
EOF

cat <<EOF > whereami/flop-variant/service-type.yaml 
apiVersion: "v1"
kind: "Service"
metadata:
  name: "whereami"
spec:
  type: ClusterIP
EOF

cat <<EOF > whereami/flop-variant/kustomization.yaml 
nameSuffix: "-flop"
namespace: whereami
commonLabels:
  app: whereami-flop
resources:
- ../base
patches:
- path: cm-flag.yaml
  target:
    kind: ConfigMap
- path: service-type.yaml
  target:
    kind: Service
EOF

kubectl --context=autopilot-cluster-us-central1 apply -k whereami/flop-variant
kubectl --context=autopilot-cluster-us-east4 apply -k whereami/flop-variant

# create serviceExports
cat <<EOF > flip_svc_export.yaml 
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: whereami-flip
  namespace: whereami
EOF

kubectl --context=autopilot-cluster-us-central1 apply -f flip_svc_export.yaml
kubectl --context=autopilot-cluster-us-east4 apply -f flip_svc_export.yaml

cat <<EOF > flop_svc_export.yaml 
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: whereami-flop
  namespace: whereami
EOF

kubectl --context=autopilot-cluster-us-central1 apply -f flop_svc_export.yaml
kubectl --context=autopilot-cluster-us-east4 apply -f flop_svc_export.yaml

# create multi-cluster gateway
cat <<EOF > gateway.yaml
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: external-http
  namespace: whereami
spec:
  gatewayClassName: gke-l7-global-external-managed-mc
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      kinds:
      - kind: HTTPRoute
EOF

kubectl --context=autopilot-cluster-us-central1 apply -f gateway.yaml

# create httpRoute to flip, to test
cat << EOF > default-httproute.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: default-httproute
  namespace: whereami
spec:
  parentRefs:
  - name: external-http
    namespace: whereami
  rules:
  - backendRefs:
    - group: net.gke.io
      kind: ServiceImport
      name: whereami-flip
      port: 80
      weight: 50
    - group: net.gke.io
      kind: ServiceImport
      name: whereami-flop
      port: 80
      weight: 50
EOF

kubectl --context=autopilot-cluster-us-central1 apply -f default-httproute.yaml

# test
export GATEWAY_IP=$(kubectl --context=autopilot-cluster-us-central1 -n whereami get gateway/external-http -o jsonpath='{.status.addresses[0].value}')
echo $GATEWAY_IP
watch -n 0.2 'curl http://${GATEWAY_IP} -s | jq'
```