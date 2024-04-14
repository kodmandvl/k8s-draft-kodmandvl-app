# Мой простенький Helm-чарт для оператора CloudNativePG

Установка оператора в локальном черновике выполнялась применением манифеста [cnpg-1.22.1.yaml](https://github.com/cloudnative-pg/cloudnative-pg/releases/download/v1.22.1/cnpg-1.22.1.yaml) из [проекта CloudNativePG на GitHub](https://github.com/cloudnative-pg/cloudnative-pg). 

Для большей отказоустойчивости я увеличил количество реплик для cnpg-controller-manager (там использовалась 1 реплика). 

Поэтому этот параметр (а в перспективе развития нашего проекта, возможно, и другие) нужно параметризировать. 

Поэтому создал для установки оператора простенький Helm Chart. 

Также параметризировал целевой неймспейс. 

## Установка:

```bash
kubectl create namespace cnpg-system
kubectl label namespaces/cnpg-system app.kubernetes.io/name=cloudnative-pg
helm upgrade --install cloudnative-pg ./ --namespace cnpg-system
```

