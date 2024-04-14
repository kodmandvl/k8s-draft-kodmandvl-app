# Сервисный аккаунт cladmsa (для использования в сборках без gitlab агента)

От данного вариант отказался, здесь оставил его просто для истории (и на всякий случай, если в дальнейшем понадобится по какой-либо причине). 

В данном сценарии использовалась в GitLab CI/CD protected переменная KUBE_CONFIG, содержимое которой - закодированный в base64 файл kubeconfig для подключения сервисного аккаунта cladmsa. 

Пример такого проверочного .gitlab-ci.yml файла: 

```yaml
# (My .gitlab-ci.yml test example without gitlab-agent)
stages:
  - check_conn_to_k8s

check_conn_to_k8s:
  image: dtzar/helm-kubectl:3.13.2
  stage: check_conn_to_k8s
  before_script:
    - mkdir -p ~/.kube/
    - echo $KUBE_CONFIG | base64 -d > ~/.kube/config
    - export KUBECONFIG=~/.kube/config
  script:
    - kubectl get nodes -o wide
```

В манифесте cladmsa.yaml описан сервисный аккаунт cladmsa. 

