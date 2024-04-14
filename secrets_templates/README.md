Вот так необходимо создать наши секреты до инсталляции компонентов (само собой, не шаблоны из этой директории, а настоящие секреты): 

```bash
kubectl apply -f myca-key-pair-secret.yaml -n cert-manager
kubectl apply -f application-user-secret.yaml -n mypg
kubectl apply -f pgadmin-secret.yaml -n mypg
kubectl apply -f postgres-user-secret.yaml -n mypg
kubectl apply -f post-sql-secret.yaml -n mypg
kubectl apply -f s3-secret.yaml -n mypg
```

- myca-key-pair-secret.yaml - сертификат и ключ моего удостоверяющего центра сертификации (CA)
- application-user-secret.yaml - имя пользователя-владельца целевой прикладной БД CNPG и его пароль
- postgres-user-secret.yaml - имя и пароль суперпользователя БД CNPG
- post-sql-secret.yaml - постинсталляционный SQL-скрипт для нашего кластера CNPG, но с чувствительными данными внутри
- s3-secret.yaml - секрета для доступа к объектному хранилищу бэкапов на S3
- pgadmin-secret.yaml - e-mail и пароль административного пользователя нашего statefulset-а PGAdmin4

