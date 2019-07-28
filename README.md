# Выпускной проект курса "DevOps практики и инструменты"
## Общая информация
Участники:
* Кушнерчук Сергей Юрьевич [skushnerchuk](https://github.com/skushnerchuk)
* Бабушкин Владимир Сергеевич [4babushkin](https://github.com/4babushkin)
* Беляков Игорь Вячеславович [weisdd](https://github.com/weisdd)

Задачи проекта:
* автоматическое развертывание микросервисного приложения crawler ([backend](https://github.com/express42/search_engine_crawler), [frontend](https://github.com/express42/search_engine_ui));
* организация мониторинга, логгирования;
* выстраивание процессов CI/CD.

Использованные продукты и технологии:
* контейнеризация: Docker;
* оркестрация: GKE;
* provisioning: terraform (+ Terraform Cloud Remote State Management);
* СУБД: MongoDB;
* брокер сообщений: RabbitMQ;
* CI/CD: Gitlab CI.

## Запуск и остановка проекта
### Запуск
Note: Поскольку в проекте используется Terraform Cloud Remote State Management, перед первым запуском terraform необходимо получить токен на https://app.terraform.io/ и сохранить его в ~/.terraformrc:
```
credentials "app.terraform.io" {
    token = "mhVn15hHLylFvQ.atlasv1.jAH..."
}
```

Набор команд для запуска проекта:
```bash
cd terraform
terraform init
terraform apply
cd ../kubernetes/
helm upgrade prometheus charts/prometheus/ --install
helm upgrade grafana charts/grafana/ --install
helm dep update charts/efk
helm upgrade efk charts/efk/ --install
helm dep update charts/crawler
helm upgrade crawler charts/crawler/ --install
```
### Ссылки на приложение и служебные сервисы
* http://nginx.weisdd.space - production-ui для бота;
* http://review.weisdd.space - ui для тестирования бота;
* http://grafana.weisdd.space - grafana;
* http://prometheus-server.weisdd.space - prometheus.

### Остановка проекта
```bash
cd terraform
terraform destroy
```

### Настройка GitLab для работы с развернутым кластером

После того, как кластер подготовлен к работе, необходимо настроить интеграцию проектов приложения с ним. Для этого следует в GitLab:

* перейти в раздел Operations/Kubernetes в проекте приложения
* нажать Add Kubernetes cluster, выбрать вкладку Existing Kebernetes cluster
* Задать параметры интеграции

*Cluster name* - имя кластера

*API URL* - ссылка на Kubernetes API. Для ее получения выполните команду:
```bash
kubectl cluster-info | grep 'Kubernetes master' | awk '/http/ {print $NF}'
```
и полученный адрес укажите в этом поле

*CA Certificate* - клиентский сертификат. Для его получения выполните команды:
```bash
kubectl get secrets
kubectl get secret default-token-xxxxx -o jsonpath="{['data']['ca\.crt']}" | base64 --decode
```
где default-token-xxxxx - имя секрета, выдаваемого первой командой.

*Service Token* - токен, используемый для доступа к кластеру. Для его получения необходимо:
* создайте файл *sa.yaml* с содержимым
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: gitlab-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: gitlab-admin
  namespace: kube-system
```
Примените этот манифест и получите токен:
```bash
kubectl apply -f sa.yaml
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep gitlab-admin | awk '{print $1}')
```
