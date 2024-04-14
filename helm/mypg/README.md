# Мой Helm-чарт для развёртывания кластера CloudNativePG

Развёртывание кластера PostgreSQL под управлением оператора CloudNativePG в локальном черновике выполнялась применением отдельных манифестов (в app/local_draft/mypg). 

Там же в директории app/local_draft/mypg есть разные вариации кластера. 

Для большей управляемости и гибкости я преобразил эти отдельные манифесты в Helm-чарт. 

## Установка cert-manager и создание необходимых секретов (до выполнения установки чарта), если они все понадобятся:

```bash
# Cert-Manager:
helm repo add jetstack https://charts.jetstack.io
helm repo list
helm repo update
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.14.4/cert-manager.crds.yaml
helm upgrade --install cert-manager jetstack/cert-manager --atomic \
--namespace=cert-manager --create-namespace --version=1.14.4
helm list -n cert-manager
kubectl get -n cert-manager secrets
kubectl get -n cert-manager all
# Secrets and ClusterIssuers:
kubectl apply -f myca-key-pair-secret.yaml -n cert-manager
kubectl apply -f clusterissuers.yaml -n cert-manager
kubectl get clusterissuers
kubectl create namespace mypg
kubectl config set-context --current --namespace=mypg
kubectl apply -f s3-secret.yaml
kubectl apply -f post-sql-secret.yaml
kubectl apply -f application-user-secret.yaml
kubectl apply -f postgres-user-secret.yaml
```

## Установка:

```bash
helm upgrade --install mypg ./ --namespace mypg
```

## После установки для настройки удалённых подключений (при наличии нашего корневого сертификата и ключа и при установке значения owncertificates_on):

* Пример для пользователя streaming_replica (для пользователей myappuser, myappuser2 и postgres аналогично автозаменой):

```bash
# Cert and key:
openssl req -new -nodes -text -out ./streaming_replica.csr -keyout ./streaming_replica.key -subj "/CN=streaming_replica"
openssl x509 -req -in ./streaming_replica.csr -text -days 3650 -CA ./tls.crt -CAkey ./tls.key -CAcreateserial -out ./streaming_replica.crt
openssl x509 -in streaming_replica.crt -text -noout
openssl x509 -in tls.crt -text -noout
# Переменные среды:
export PGUSER=streaming_replica
export PGSSLCERT=streaming_replica.crt
export PGSSLKEY=streaming_replica.key
export PGSSLROOTCERT=tls.crt
# Пример подключения:
psql 'host=<host> port=<port> dbname=myappdb sslmode=verify-ca user=streaming_replica'
```

## Проверка сертификата клиента streaming_replica (с пода):

```bash
# Connection to leader, examples:
psql 'host=<your-cluster>-rw port=5432 dbname=postgres sslmode=verify-full user=streaming_replica sslrootcert=/controller/certificates/client-ca.crt sslcert=/controller/certificates/streaming_replica.crt sslkey=/controller/certificates/streaming_replica.key'
psql 'host=<your-cluster>-rw port=5432 dbname=postgres sslmode=verify-full user=streaming_replica sslrootcert=/controller/certificates/server-ca.crt sslcert=/controller/certificates/streaming_replica.crt sslkey=/controller/certificates/streaming_replica.key'
# Connection to replica, examples:
psql 'host=<your-cluster>-ro port=5432 dbname=postgres sslmode=verify-full user=streaming_replica sslrootcert=/controller/certificates/client-ca.crt sslcert=/controller/certificates/streaming_replica.crt sslkey=/controller/certificates/streaming_replica.key'
psql 'host=<your-cluster>-ro port=5432 dbname=postgres sslmode=verify-full user=streaming_replica sslrootcert=/controller/certificates/server-ca.crt sslcert=/controller/certificates/streaming_replica.crt sslkey=/controller/certificates/streaming_replica.key'
```

