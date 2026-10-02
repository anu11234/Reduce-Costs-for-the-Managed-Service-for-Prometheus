# Reduce Costs for the Managed Service for Prometheus || **GSP1027**

**Command:**

```bash
# 1. Fetch project ID and zone dynamically
export PROJECT=$(gcloud config get-value project)
export ZONE=$(gcloud config get-value compute/zone 2>/dev/null)
export ZONE=${ZONE:-us-central1-a}

gcloud config set compute/zone $ZONE

# 2. Task 1: Deploy GKE cluster & get credentials
gcloud beta container clusters create gmp-cluster --num-nodes=1 --zone=$ZONE --enable-managed-prometheus || gcloud container clusters get-credentials gmp-cluster --zone=$ZONE
gcloud container clusters get-credentials gmp-cluster --zone=$ZONE

# 3. Task 2: Deploy managed collection & example app
kubectl -n gmp-system apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/main/examples/self-pod-monitoring.yaml
kubectl -n gmp-system apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/main/examples/example-app.yaml

# 4. Task 5: Create op-config.yaml & upload to Cloud Storage
cat <<'EOF' > op-config.yaml
apiVersion: monitoring.googleapis.com/v1alpha1
collection:
  filter:
    matchOneOf:
    - '{job="prom-example"}'
    - '{__name__=~"job:.+"}'
kind: OperatorConfig
metadata:
  annotations:
    components.gke.io/layer: addon
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"monitoring.googleapis.com/v1alpha1","kind":"OperatorConfig","metadata":{"annotations":{"components.gke.io/layer":"addon"},"labels":{"addonmanager.kubernetes.io/mode":"Reconcile"},"name":"config","namespace":"gmp-public"}}
  creationTimestamp: "2022-03-14T22:34:23Z"
  generation: 1
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
  name: config
  namespace: gmp-public
  resourceVersion: "2882"
  uid: 4ad23359-efeb-42bb-b689-045bd704f295
EOF

kubectl apply -f op-config.yaml

gcloud storage buckets create --project=$PROJECT gs://$PROJECT || true
gcloud storage cp op-config.yaml gs://$PROJECT
gcloud storage buckets add-iam-policy-binding gs://$PROJECT --member=allUsers --role=roles/storage.objectViewer

# 5. Task 7: Create prom-example-config.yaml & upload
cat <<'EOF' > prom-example-config.yaml
apiVersion: monitoring.googleapis.com/v1alpha1
kind: PodMonitoring
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"monitoring.googleapis.com/v1alpha1","kind":"PodMonitoring","metadata":{"annotations":{},"labels":{"app.kubernetes.io/name":"prom-example"},"name":"prom-example","namespace":"gmp-test"},"spec":{"endpoints":[{"interval":"30s","port":"metrics"}],"selector":{"matchLabels":{"app":"prom-example"}}}}
  creationTimestamp: "2022-03-14T22:33:55Z"
  generation: 1
  labels:
    app.kubernetes.io/name: prom-example
  name: prom-example
  namespace: gmp-test
  resourceVersion: "2648"
  uid: c10a8507-429e-4f69-8993-0c562f9c730f
spec:
  endpoints:
  - interval: 60s
    port: metrics
  selector:
    matchLabels:
      app: prom-example
status:
  conditions:
  - lastTransitionTime: "2022-03-14T22:33:55Z"
    lastUpdateTime: "2022-03-14T22:33:55Z"
    status: "True"
    type: ConfigurationCreateSuccess
  observedGeneration: 1
EOF

gcloud storage cp prom-example-config.yaml gs://$PROJECT
gcloud storage buckets add-iam-policy-binding gs://$PROJECT --member=allUsers --role=roles/storage.objectViewer
