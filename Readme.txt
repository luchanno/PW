1) Установить в k8s ingress-nginx (опционально, если отсутствует)

kubectl create namespace m && helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx/ && helm repo update && helm install nginx ingress-nginx/ingress-nginx --namespace m -f nginx/0_OPT_nginx_ingress-25239-20146a.yaml

2) Установить сервис auth в отдельный ns

kubectl create ns auth && kubectl config set-context --current --namespace=auth && cd auth && skaffold run && kubectl delete -A ValidatingWebhookConfiguration ingress-nginx-admission && cd ..

3) Создать отдельный namespace для прочих сервисоа
 
kubectl create ns msusers-ns

4) Установить Kafka

kubectl create -f ./kafka/resources/zookeeper.yml && kubectl create -f ./kafka/resources/kafka.yml

5) Установить Prometheus&GrafanaPrometheus и пробросить к ним порты

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack --namespace=msusers-ns --create-namespace -f prometheus.yaml

kubectl port-forward service/prometheus-operated 9090 -n msusers-ns
kubectl port-forward service/prometheus-grafana  9000:80 -n msusers-ns

6) Развернуть БД в k8s

helm install ./DB/helm-chart --generate-name

7) Провести миграцию данных в БД (в первый раз или при перезапуске)

kubectl apply -f DB/migr

8) Добавить ingress для auth

kubectl apply -f auth_ingress

9) Развернуть основные сервисы

kubectl apply -f MSUSERS/k8s && kubectl apply -f Order/k8s && kubectl apply -f Billing/k8s && kubectl apply -f Inventory/k8s && kubectl apply -f Delivery/k8s

Использование

1) prometheus UI:  http://127.0.0.1:9000/
   graphana UI: http://127.0.0.1:9090/   admin/prom-operator














