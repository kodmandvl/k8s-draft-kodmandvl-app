# Мой Helm-чарт для развёртывания графического инструмента PGAdmin

Развёртывание пулера в локальном черновике выполнялась применением отдельных манифестов (в app/local_draft/mypg). 

Для большей управляемости и гибкости я преобразил эти отдельные манифесты в Helm-чарт. 

## Установка Ingress-Nginx перед установкой PGAdmin:

[Ссылка](https://yandex.cloud/ru/docs/managed-kubernetes/tutorials/ingress-cert-manager?from=int-console-help-center-or-nav). 

Установка ingress-nginx: 

```text
$ helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace
NAME: ingress-nginx
LAST DEPLOYED: Thu Apr  4 23:10:09 2024
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the load balancer IP to be available.
You can watch the status by running 'kubectl get service --namespace ingress-nginx ingress-nginx-controller --output wide --watch'

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```

## Установка PGAdmin:

```bash
helm upgrade --install pgadmin ./ --namespace mypg
```

