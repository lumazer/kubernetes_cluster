# Репоз для проектной работы OTUS Kuber-2024-02

Шаги предпринятые для поднятия кластера:
0. Подготовил три VM с пробросом на одну из них порта 443
1. Установка кластера через kubespray (один controlplane+worker node и два worker node)
  - <code>git pull https://github.com/kubernetes-sigs/kubespray.git && cd kubespray</code>
  - <code>cp -rfp inventory/sample inventory/kubespray_k8s</code>
  - <code>declare -a IPS=(192.168.55.100 192.168.55.101 192.168.55.102)</code>
  - <code>CONFIG_FILE=inventory/kubespray_k8s/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}</code>
  - <code>ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root reset.yml</code>
3. Установка argocd
  - подклчюился к controlplane с пробросом порта 8080 - <code>ssh root@192.168.55.100 -L8080:127.0.0.1:8080</code>
  - Создал namespace argocd - <code>kubectl create namespace argocd</code>
  - скачал yaml argocd install для HA - <code>wget https://raw.githubusercontent.com/argoproj/argo-cd/master/manifests/ha/install.yaml -O argocd_ha_install.yaml</code>
  - применил в namespace argocd - <code>kubectl apply -n argocd -f argocd_ha_install.yaml</code>
  - Убедился что argocd поднялся
    - Поды в статусе Running - <code>kubectl get pods -n argocd</code>
    - Сервисы подняты и имеют внутренние IP - <code>kubectl get service -n argocd</code>
  - Делаем проброс сервиса Argo CD на локальный порт CP -<code> kubectl port-forward -n argocd svc/argocd-server 8080:8080</code>
  - Получаем пароль в base64 через - <code>kubectl get secrets argocd-initial-admin-secret -n argocd -o yaml</code>
  - Декодируем пароль - <code>echo !ПАРОЛЬ! | base64 --decode</code>
  - Авторизуемся в argocd для просмотра статуса
  - ingress для ArgoCD. Применяем из репозитория https://github.com/lumazer/kubernetes_cluster - application-argocd-ingress.yaml
    - Для добавления tls сертификата в кластер
    - для настройки ingress
3. Деплоим приложение online boutique
  - Добавляем в Argo CD приложение из репозитория https://github.com/lumazer/kubernetes_cluster - application-online-boutique.yaml
  - ingress для online boutique. Применяем из репозитория https://github.com/lumazer/kubernetes_cluster - application-online-boutique-ingress.yaml
    - Для добавления tls сертификата в кластер
    - для настройки ingress
4. Деплоим prometheus+grafana через helm
  - C настройками values для мониторинга ресурсов без меток и для e-mail оповещений - https://github.com/lumazer/kubernetes_cluster/blob/main/application-kube-prometheus-stack.yaml (использовать полученный пароль SMTP от yandex)
  - ingress для prometheus и grafana. Применяем из репозитория https://github.com/lumazer/kubernetes_cluster - application-monitoring-ingresess.yaml
    - Для добавления tls сертификата в кластер
    - для настройки ingress
5. Деплоим loki для сбора логов через helm
  - На каждой ноде kubernetes устанавливаем драйвер nfs - <code>sudo apt install cifs-utils nfs-common</code>
  - Создадим PV для loki - <code>kubectl apply -f https://github.com/lumazer/kubernetes_cluster/blob/main/pv002_loki.yaml</code>
  - Деплоим loki с настройками values для отключения авторизации, для использования "coredns" как основного DNS сервиса в kubernetes и для подключения PV Filesystem - https://github.com/lumazer/kubernetes_cluster/blob/main/application-loki-helm.yaml
  - Деплоим promtail для получения логов со всего кластера - https://github.com/lumazer/kubernetes_cluster/blob/main/application-promtail-daemonset.yaml
  - Убеждаемся, что "Logging" появился в Grafana и логи собираются со всех namespace-ов
