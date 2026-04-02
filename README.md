Note: View Readme file in code format. Will change the format in future

**Prometheus Sharding & Thanos Setup
**
This project demonstrates:
1. Manual Sharding
2. Automatic Sharding (hashmod)
3. Manual Kubernetes Sharding
4. Automatic Kubernetes Sharding
5. Thanos in Docker/baremetal
6. Thanos in Kubernetes

**Sharding (Manual - Docker)

Prerequisites
Docker
Docker Compose

▶️ Start Applications
cd ../../applications
docker compose up -d --build go-application
docker compose up -d --build python-application
docker compose up -d --build dotnet-application
docker compose up -d --build nodejs-application

▶️ Start Prometheus
cd ../sharding/manual
docker compose up -d prometheus-00
docker compose up -d prometheus-01

👉 Targets are manually split between instances
👉 Each Prometheus has ~2 targets
👉 Here we spllitted scrape targets manually for each prometheus instances. Check each prometheus instance targets, it have 2 targets each.

**⚙️ Sharding (Automatic - Hashmod)
**

▶️ Start Applications
docker compose up -d --build ../../applications/go-application
docker compose up -d --build ../../applications/python-application
docker compose up -d --build ../../applications/dotnet-application
docker compose up -d --build ../../applications/nodejs-application

▶️ Start Prometheus
cd ../sharding/automated
docker compose up -d prometheus-00
docker compose up -d prometheus-01

👉 Targets are split using hashmod relabeling
👉 Distribution is automatic but not perfectly equal
👉 Here we spllited scrape targets using hasmod. Hashmod does not gurantee equal targets for each shard. Here see prometheus instance they have targets not 50% each.

☸️ Kubernetes Sharding (Manual)

Prerequisites
Kubernetes cluster
kubectl
Minikube
helm

▶️ Setup Cluster
minikube start --driver=docker

▶️ Install helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh

▶️ Install Prometheus Stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
CHART_VERSION=75.4.0
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --version ${CHART_VERSION} \
  --namespace monitoring \
  --create-namespace

▶️ Deploy Applications
cd ../../applications
docker compose build go-application python-application dotnet-application nodejs-application
minikube image load go-application:latest
minikube image load python-application:latest
minikube image load dotnet-application:latest
minikube image load nodejs-application:latest

kubectl apply -f go-application/deployment.yaml
kubectl apply -f dotnet-application/deployment.yaml
kubectl apply -f python-application/deployment.yaml
kubectl apply -f nodejs-application/deployment.yaml

▶️ Create service monitor for each application 
cd ../kubernetes/sharding/
kubectl apply -f service-monitors.yaml
kubectl get servicemonitor

▶️ Create Service Account for promethues 
kubectl apply -f serviceaccount.yaml
kubectl get serviceaccount

▶️ Create prometheus instances with different serviceMonitorSelector
cd manual
kubectl apply -f prometheus-00.yaml
kubectl apply -f prometheus-01.yaml

▶️ Check prometheus
kubectl port-forward prometheus-prometheus-00-0 9090:9090
kubectl port-forward prometheus-prometheus-01-0 9091:9090

👉 Targets split using ServiceMonitor selectors
👉 Each instance scrapes ~2 targets
👉 Here we created two prometheus instances with it can select servicemonitor based on labels. So check prometheus instances, each has 2 scrape targets

⚙️ Kubernetes Sharding (Automatic)

Prerequisites
Kubernetes cluster
kubectl
Minikube
helm

▶️ Setup Cluster
minikube start --driver=docker

▶️ Install helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh

▶️ Install Prometheus Stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
CHART_VERSION=75.4.0
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --version ${CHART_VERSION} \
  --namespace monitoring \
  --create-namespace

▶️ Deploy Applications
cd ../../applications
docker compose build go-application python-application dotnet-application nodejs-application
minikube image load go-application:latest
minikube image load python-application:latest
minikube image load dotnet-application:latest
minikube image load nodejs-application:latest

kubectl apply -f go-application/deployment.yaml
kubectl apply -f dotnet-application/deployment.yaml
kubectl apply -f python-application/deployment.yaml
kubectl apply -f nodejs-application/deployment.yaml

▶️ Create service monitor for each application 
cd ../kubernetes/sharding/
kubectl apply -f service-monitors.yaml
kubectl get servicemonitor

▶️ Create Service Account for promethues 
kubectl apply -f serviceaccount.yaml
kubectl get serviceaccount

▶️ Deploy prometheus instance with shards
kubectl apply -f automated/prometheus.yaml

👉 Uses Prometheus Operator shards field
👉 Automatic distribution (not perfectly equal)
👉 Here we used shards in prometheus operator. It does not gurantee 50% split scrape targets to each prometheus instanes. See prometheus instances.


📦 Thanos (Docker/Baremetal)

▶️ Start Applications
cd applications
docker compose up -d --build go-application python-application dotnet-application nodejs-application

▶️ Start Prometheus
cd ../Thanos/prometheus
docker compose up -d prometheus-00 prometheus-01

▶️ Start Thanos Stack (thanos sidecar, thanos query, grafana, minio services)
cd ..
docker compose up -d

👉 Query all metrics via Thanos Query
👉 Grafana + MinIO included
👉 Centralized view of all shards
👉 See each prometheus instances have 2 scrape targets. Also We can access 4 application metrics using thanos query, as well as grafana. Also check minio bucket is created. We have added datasource automatically here.

☸️ Thanos (Kubernetes)

▶️ Install Stack
minikube start --driver=docker

▶️ Install helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh

▶️ Install kube-prometheus-stack
cd Thanos
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --version ${CHART_VERSION} \
  --namespace monitoring \
  --create-namespace \
  --values values.yaml

▶️ Deploy Applications
cd ../../applications
docker compose build go-application python-application dotnet-application nodejs-application

minikube image load go-application:latest
minikube image load python-application:latest
minikube image load dotnet-application:latest
minikube image load nodejs-application:latest

kubectl apply -f */deployment.yaml

▶️ Create servicemonitors for each application
cd ../kubernetes/Thanos/
kubectl apply -f servicemonitors.yaml
kubectl get servicemonitor

▶️ Create minio instance
kubectl apply -f minio.yaml
kubectl get pods

▶️ Create Secret that allows thanos to connect to this minio s3
kubectl apply -f thanos-secret.yaml
kubectl get secret

▶️ Create serviceaccount used for prometheus
kubectl apply -f serviceaccount.yaml
kubectl get serviceaccount

▶️ Deploy Prometheus + Thanos sidecars using prometheus-operator
kubectl apply -f prometheus-00.yaml
kubectl apply -f prometheus-01.yaml

▶️ Deploy Thanos Query
kubectl apply -f thanos-query.yaml

▶️ Access Grafana
kubectl -n monitoring port-forward svc/grafana 3000:80

👉 Centralized metrics using Thanos Query
👉 Metrics stored in MinIO (S3)
👉 Grafana integrated
👉 See each prometheus instances have 2 scrape targets. Also We can access 4 application metrics using thanos query, as well as grafana. Also check minio bucket is created. We have to added datasource.
