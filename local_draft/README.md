# Local Draft

## Web (my nginx server, просто для проверки жизнеспособности кластера, ingress-а и metallb некое пустое демо-приложение)

```bash
kubectl config use-context minikube
kubectl apply -f web-ns.yaml
kubectl config set-context --current --namespace=web
kubectl config get-contexts
kubectl apply -f web-deploy.yaml 
kubectl apply -f web-svc-lb.yaml
kubectl apply -f web-svc-headless.yaml
kubectl apply -f web-ingress.yaml
kubectl get all
kubectl get ingresses
curl 192.168.118.1
curl `minikube ip`/web
```

## MyPG (my CloudNativePG installation)

```bash
kubectl config use-context minikube
wget https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/main/releases/cnpg-1.22.1.yaml
# Ставим больше одной реплики для cnpg-controller-manager, например, 3:
nano cnpg-1.22.1.yaml 
kubectl apply -f cnpg-1.22.1.yaml
kubectl apply -f mypg-ns.yaml
kubectl config set-context --current --namespace=mypg
kubectl config get-contexts
kubectl apply -f mypg-cluster.yaml
kubectl get all
kubectl get pods
# Для MetalLB:
kubectl apply -f mypg-svc-lb.yaml
kubectl get svc
```

* Попасть на под-primary: 

```bash
kubectl exec -it pods/`kubectl get pods -l role=primary,cnpg.io/cluster=mypg | grep -v ^NAME | cut -d' ' -f1` -- /bin/bash
```

* Попасть в PG через MetalLB: 

```bash
psql postgres://192.168.118.2:5432/myappdb -U myappuser
```

Здесь дальше рассмотрел, исследовал, разобрал и реализовал в черновике дальнейшие настройки для нашей сущности "Кластер": 

* OK: регулярные бэкапы на S3 (minio в локальном черновом миникубе, S3 Storage в Yandex Cloud для кластера Kubernetes в Yandex Cloud)
* OK: восстановление из бэкапов
* OK: создание кластера копии как бы для тестового стенда
* OK: расширения
* OK: вообще свой докер-образ постгреса (добавить barman и проверить требования https://cloudnative-pg.io/documentation/1.22/container_images/ )
* OK: добавить докерфайл в свой проект
* OK: сделать чтобы мой кастомный образ cnpg собирался пайпом и хранился в Registry GitLab-а (это сделал уже не в локальном черновике на minikube, а дальше при интеграции кластера K8s и GitLab)
* OK: добавить количество реплик контроллера cnpg больше одной (2 или 3)
* OK: тюнинг параметров
* OK: постнастройки (SQL-команды и скрипты настройки, в т.ч. таблица-история размера БД по крону, пользователи, расширения)
* OK: env для инсталляции (например, TZ) https://cloudnative-pg.io/documentation/1.22/cluster_conf/ 
* OK: ресурсы и лимиты для нашей инсталляции https://cloudnative-pg.io/documentation/1.22/resource_management/ 
* OK: мониторинг (Prometheus+Grafana)
* OK: правила алертов
* OK: pooler (pgbouncer) https://cloudnative-pg.io/documentation/1.22/connection_pooling/ 
* OK: UI для PostgreSQL (PGAdmin4)
* OK: в локальном черновике какое-то приложение на PostgreSQL (просто для проверки работы реальных приложений на нашей инсталляции), а именно: Vault
* OK: Ingress Controller (в minikube само собой OK)
* OK: Load Balancer (в minikube само собой OK)
* OK: cert-manager
* OK: registry => встроенный в GitLab (там собранный образ будем хранить)

В т.ч. удовлетворены данные требования: 

- Берете базу данных (OK)
- показываете ее отказоустойчивость (OK)
- ее бэкапы (OK)
- ее тонкий тюнинг параметров (OK)
- тюнинг различных балансировщиков для баз данных (OK)

## Prometheus и Grafana

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update
helm repo ls
# Создаем новый NS и делаем его дефолтным:
kubectl apply -f mon-ns.yaml 
kubectl config set-context --current --namespace=mon
kubectl config get-contexts
wget https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/main/docs/src/samples/monitoring/kube-stack-config.yaml 
helm upgrade --install \
     -f kube-stack-config.yaml \
     prometheus-community prometheus-community/kube-prometheus-stack \
     --namespace=mon
kubectl --namespace mon get pods -l "release=prometheus-community"
kubectl get crds
kubectl get svc
wget https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/main/docs/src/samples/monitoring/prometheusrule.yaml
kubectl apply -f prometheusrule.yaml
kubectl apply -f prom-svc-lb.yaml
kubectl apply -f graf-svc-lb.yaml
kubectl get all
kubectl get svc
# Если бы не было MetalLB, то, например: you can port-forward: kubectl port-forward svc/prometheus-community-grafana 3000:80
# credentials: admin as username, prom-operator as password (defined in kube-stack-config.yaml)
# Prometheus: http://192.168.118.3:9090
# Grafana: http://192.168.118.4:3000
wget https://raw.githubusercontent.com/cloudnative-pg/charts/main/charts/cloudnative-pg/monitoring/grafana-dashboard.json
# And then import dashboard to Grafana
```

## Нагрузка (смотрим дашборд в графане):

```bash
# (нужно, чтобы кластер PostgreSQL был создан после настрйоки Prometheus и Grafana, поэтому я его пересоздал перед подачей нагрузки)
pgbench postgres://192.168.118.2:5432/myappdb -U myappuser -i
pgbench postgres://192.168.118.2:5432/myappdb -U myappuser -P5 -T 300
```

## Регулярные бэкапы (локально):

```bash
kubectl apply -f mypg-backup.yaml
kubectl describe scheduledbackups
kubectl get scheduledbackups -o yaml
kubectl get scheduledbackups
kubectl get clusters
kubectl get backups
kubectl exec -it -n mypg pods/mypg-1 -- /bin/bash
kubectl exec -it -n minio pods/minio-0 -- /bin/bash
```

## Kubectl-CNPG:

```bash
wget https://github.com/cloudnative-pg/cloudnative-pg/releases/download/v1.22.2/kubectl-cnpg_1.22.2_linux_x86_64.tar.gz
tar -xzvf kubectl-cnpg_1.22.2_linux_x86_64.tar.gz
sudo install -o root -g root -m 0755 kubectl-cnpg /usr/local/bin/kubectl-cnpg
sudo install -o root -g root -m 0755 kubectl-cnpg /usr/local/bin/kcnpg
kubectl-cnpg status mypg
kubectl-cnpg backup mypg
kubectl-cnpg report cluster mypg
```

## Regular jobs with PG_CRON:

Добавляем в постинициализационные скрипты наполнение таблицы size_hist 

## PGBench:

```bash
psql postgres://192.168.49.2:30432/myappdb -U myappuser -c "select * from size_hist order by select_time desc limit 10;"
pgbench postgres://192.168.49.2:30432/myappdb -U myappuser -i -s2 -n
pgbench postgres://192.168.49.2:30432/myappdb -U myappuser -T60 -P5 -j8 -c16 -n
```

Состояние (после PGBench и после ежеминутного добавления по строчке в таблицу истории): 

```bash
psql postgres://192.168.49.2:30432/myappdb -U myappuser
select count(*), min(select_time), max(select_time), min(size), max(size), cluster_name from size_hist group by cluster_name order by 3 desc;
select count(*), min(abalance), max(abalance), avg(abalance) from pgbench_accounts ;
```

## Восстановление работоспособности после удаления пода

```bash
kubectl delete pods/mypg-1
kubectl logs pods/mypg-1
kubectl get pods/mypg-1 -w
psql postgres://192.168.49.2:30432/myappdb -U myappuser
select * from size_hist order by select_time desc limit 10;
select count(*), min(select_time), max(select_time), min(size), max(size), cluster_name from size_hist group by cluster_name order by 3 desc;
select count(*), min(abalance), max(abalance), avg(abalance) from pgbench_accounts;
```

Успех! 

## Восстановление из бэкапа

Проверим восстановление из бэкапа (это будет тестовый стенд, копия продакшена). 

```bash
psql postgres://192.168.49.2:30434/myappdb -U myappuser
select * from size_hist order by select_time desc limit 10;
select count(*), min(select_time), max(select_time), min(size), max(size), cluster_name from size_hist group by cluster_name order by 3 desc;
select count(*), min(abalance), max(abalance), avg(abalance) from pgbench_accounts ;
```

## Гибернация кластеров:

```bash
kubectl annotate cluster test-stand --overwrite cnpg.io/hibernation=on
kubectl annotate cluster mypg --overwrite cnpg.io/hibernation=on
kubectl annotate cluster test-stand-yc --overwrite cnpg.io/hibernation=on
kubectl annotate cluster mypg-yc --overwrite cnpg.io/hibernation=on
```

## Pooler (PGBouncer):

Добавил деплоймент Pooler (с тремя репликами), это PGBouncer, который является пулером сессий, а также (благодаря трем репликам) является одновременно еще и балансировщиком. 

```bash
psql postgres://`minikube ip`:30543/myappdb -U myappuser -c "select * from size_hist order by select_time desc limit 10;"
pgbench postgres://`minikube ip`:30543/myappdb -U myappuser -i -s2 -n
pgbench postgres://`minikube ip`:30543/myappdb -U myappuser -T300 -P5 -j16 -c700 -n
```

В другом окне  можно посмотреть и увидеть, что запросы идут через три разные реплики Pooler (см. IP pooler-ов и client_addr из представления pg_stat_activity, нагрузка идет по трем пулерам приблизительно равномерно, по 149-150 коннектов с каждого из трех подов pooler-а): 

```bash
kubectl get pods -o wide
psql postgres://`minikube ip`:30543/myappdb -U myappuser -c "select count(*) connections, datname, usename, application_name, client_addr, client_hostname, backend_type from pg_stat_activity group by datname, usename, application_name, client_addr, client_hostname, backend_type order by 1 desc, 2,3,4;"
```

Здесь в локальном примере у нас только один экземпляр (реплик нет), но в случае полноценного кластера Kubernetes у нас на одном из рабочих узлов был бы инстанс СУБД, лидер, работающий на запись, и несколько инстансов-реплик в read only. Наличие pooler-ов решает несколько задач: 

- рациональное потребление CPU и RAM (есть пул сессий, физически выделяется меньше сессий, чем запрашивается в приложении)
- возможность подключения большего количества сессий, чем задано в max_connections, благодаря пулу сессий
- благодаря деплойменту с несколькми репликами получаем также и балансировку нагрузки между несколькими рабочими узлами

В примере выше в pgbench задано 700 клиентов (-c700), хотя max_connections в PostgreSQL у нас задано 500. Реальных коннектов в СУБД у нас значительно меньше: 

- pool_mode = transaction, поэтому после транзакции подключение освобождается и отдается обратно в pool
- default_pool_size у нас задан 150, поэтому сессий используется порядка 450 при такой нагрузке от PGBench при наличии трех реплик Pooler-а

Если бы мы с max_connections = 500 попробовали бы подключить к нашему PostgreSQL 700 клиентов от PGBench, такой тест бы не удался: 

```text
$ pgbench postgres://`minikube ip`:30433/myappdb -U myappuser -T300 -P5 -j16 -c700 -n
pgbench (15.5 (Debian 15.5-1.pgdg110+1), server 16.2 (Debian 16.2-1.pgdg110+2))
pgbench: error: connection to server at "192.168.49.2", port 30433 failed: FATAL:  remaining connection slots are reserved for roles with the SUPERUSER attribute
pgbench: error: could not create connection for client 441
```

## UI для работы с СУБД (PGAdmin4)

В неймспейсе mypg создал statefulset для PGAdmin4, в котором монтируется Configmap со списком сохраненных коннектов servers.json, а также монтируется PV, в которой далее сохраняются данные (список коннектов, пользователи приложения PGAdmin4 и т.д.). 

## Vault как пример работающего на нашем постгресе приложения (есть только в локальном черновике, просто как пример такой возможности)

* Подготовка (объекты в нашей БД):

```sql
CREATE TABLE vault_kv_store (
  parent_path TEXT COLLATE "C" NOT NULL,
  path        TEXT COLLATE "C",
  key         TEXT COLLATE "C",
  value       BYTEA,
  CONSTRAINT pkey PRIMARY KEY (path, key)
);

CREATE INDEX parent_path_idx ON vault_kv_store (parent_path);

CREATE TABLE vault_ha_locks (
  ha_key                                      TEXT COLLATE "C" NOT NULL,
  ha_identity                                 TEXT COLLATE "C" NOT NULL,
  ha_value                                    TEXT COLLATE "C",
  valid_until                                 TIMESTAMP WITH TIME ZONE NOT NULL,
  CONSTRAINT ha_key PRIMARY KEY (ha_key)
);
```

* Неймспейс для Vault-а: 

```bash
kubectl apply -f vault-ns.yaml 
```

* Склонируем репозиторий vault: 

```bash
git clone https://github.com/hashicorp/vault-helm.git
```

* Отредактируем параметры установки в vault-values.yaml 

```text
standalone:
enabled: true
....
ha:
enabled: false
...
ui:
enabled: true
serviceType: "LoadBalancer"
...
storage "postgresql" {
connection_url = "postgres://myappuser:MyPass123@mypg-yc-svc-lb.mypg.svc.cluster.local:5433/myappdb"
}
```

* Сохраним в файле vault-values.yaml и запустим установку с параметрами: 

```bash
nano vault-values.yaml 
helm upgrade --install vault vault-helm -f ./vault-values.yaml --atomic --wait -n vault
```

* Провел инициализацию:

```bash
kubectl exec -it vault-0 -- vault operator init --key-shares=1 --key-threshold=1
```

```text
$ kubectl exec -it vault-0 -- vault operator init --key-shares=1 --key-threshold=1
Unseal Key 1: aOMO4LLJSi/Jywbj0tyrNIYvCAuCUBRg2LxC02fNb40=
Initial Root Token: hvs.39Tsnp5vq0AerNPYL5WdmyRg
```

Само собой, приводить токен и ключ, а также пароль от БД в README небезопасно. Но это пример учебный на локальном Minikube, который, к тому же, уже будет не существовать к моменту проверки проекта, т.ч. ничего страшного. 

* Проверил состояние vault'а:

```bash
kubectl logs vault-0
```

* Статус:

```bash
kubectl exec -it vault-0 -- vault status
```

* Распечатал vault:

```bash
kubectl exec -it vault-0 -- vault operator unseal 'aOMO4LLJSi/Jywbj0tyrNIYvCAuCUBRg2LxC02fNb40='
```

>> "Sealed          false" => под-0 распечатали ("распломбировали"). 

* Залогинился в vault (у нас есть root token)

```bash
kubectl exec -it vault-0 -- vault login
kubectl exec -it vault-0 -- vault auth list
```

* Завел пробные секреты

```bash
kubectl exec -it vault-0 -- vault secrets enable --path=test kv
kubectl exec -it vault-0 -- vault secrets list --detailed
kubectl exec -it vault-0 -- vault kv put test/test-ro/config username='testtest' password='passpass'
kubectl exec -it vault-0 -- vault kv put test/test-rw/config username='testtest' password='passpass'
kubectl exec -it vault-0 -- vault read test/test-ro/config
kubectl exec -it vault-0 -- vault kv get test/test-rw/config
```

В локальном черновике Vault на PostgreSQL работает, в БД можно видеть данные в соответствующих созданных выше таблицах. 

## cert-manager

* Добавил репозиторий, в котором хранится актуальный helm chart cert-manager: 

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo list
helm repo update
```

* Также для установки cert-manager предварительно потребуется создать в кластере некоторые CRD ([ссылка](https://github.com/cert-manager/cert-manager/tree/master/deploy/charts/cert-manager) на документацию по установке): 

```bash
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.14.4/cert-manager.crds.yaml
```

* Установил cert-manager: 

```
helm search repo cert-manager
```

```bash
helm upgrade --install cert-manager jetstack/cert-manager --atomic \
--namespace=cert-manager --create-namespace --version=1.14.4
helm list -n cert-manager
kubectl get -n cert-manager secrets
kubectl get -n cert-manager all
```

* Добавил объекты ClusterIssuer (пробовал как acme, так и ca). В дальнейшем использовал свой CA (как для сертификатов подключения к БД, так и для Ingress-ов). 

