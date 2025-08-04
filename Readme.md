# Проектная работа

## Описание

Создано 5 сервисов:
* Сервис пользователей (MSUSERS)
* Сервис заказов (Order)
* Сервис биллинга (Billing)
* Сервис склада (Inventory)
* Сервис доставки (Delivery)

Формирование заказа идет в следующей последовательности:
* Создается заказ в статусе NEW (сервис Order)
* Снимаются деньги со счета (сервис Billing)
* Снимается позиция на складе (сервис Inventory)
* Создается запись о доставке (сервис Delivery)
* Заказ переводится в статус CONFIRMED

В случае любой из проблем происходит откат предыдущих шагов

Реализовано по паттерну SAGA-хореография, с использованием брокера сообщений Kafka

См. диаграммы в папке Diagrams

## UPD: Добавлена индемпотентость создания заказа

Метод создания заказа /orders/createOrder требует обязательного параметра в HTTP Header "rqid" типа UUID

При попытке создать заказ в течение 30 секунд с таким же "rqid" вместо создания заказа будет выдаваться созданный ранее.


# Установка
## 1) Установить в k8s ingress-nginx (опционально, если отсутствует)
```kubectl create namespace m && helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx/ && helm repo update && helm install nginx ingress-nginx/ingress-nginx --namespace m -f nginx/0_OPT_nginx_ingress-25239-20146a.yaml```

## 2) Установить сервис auth в отдельный ns

```kubectl create ns auth && kubectl config set-context --current --namespace=auth && cd auth && skaffold run && kubectl delete -A ValidatingWebhookConfiguration ingress-nginx-admission && cd ..```

## 3) Создать отдельный namespace для прочих сервисов

```kubectl create ns msusers-ns```

## 4) Установить Kafka

kubectl create -f ./kafka/resources/zookeeper.yml && kubectl create -f ./kafka/resources/kafka.yml

## 5) Установить Prometheus&GrafanaPrometheus и пробросить к ним порты

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack --namespace=msusers-ns --create-namespace -f prometheus.yaml

kubectl port-forward service/prometheus-operated 9090 -n msusers-ns
kubectl port-forward service/prometheus-grafana  9000:80 -n msusers-ns

## 6) Развернуть БД в k8s

```helm install ./DB/helm-chart --generate-name```

## 7) Провести миграцию данных в БД (опционально, в первый раз!)

```kubectl apply -f DB/migr```

## 8) Добавить ingress для auth

```kubectl apply -f auth_ingress```

## 9) Развернуть основные сервисы в k8s

```kubectl apply -f Delivery/k8s && kubectl apply -f Inventory/k8s && kubectl apply -f Billing/k8s && kubectl apply -f Order/k8s && kubectl apply -f MSUSERS/k8s```

## 10) Подождать окончания деплоя 10 минут

# Tecтирование

## Postman

Рекомендуется установить расширение для newman `npm install -g newman-reporter-htmlextra`

* Коллекция: https://www.postman.com/alexkorkhov/public/collection/794gnl5/otus-saga-8
* Сценарий: https://www.postman.com/alexkorkhov/public/collection/ii5u5x6/otus-idpt-9-tests (в сценарии вставлены искуственные задержки порядка 40 секунд суммарно)
* Проверка коллекции: `newman run testing/OTUS-IDPT-9-TESTS.postman_collection.json -r htmlextra` 
* Результат выполнения: `newman/OTUS-IDPT-9-TESTS-2025-07-23-00-06-25-281-0.html`

## Метрики

## 1) prometheus UI

```http://127.0.0.1:9000/```

## 2) Graphana UI

```http://127.0.0.1:9090/```
Креды ```admin/prom-operator```
