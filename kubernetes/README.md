### Секреты для Django

В манифесте Deployment значения `SECRET_KEY` и `DATABASE_URL` берутся из Kubernetes Secret `django-secret`. Перед раскаткой манифестов создайте (или обновите) секрет в кластере:

```bash
kubectl delete secret django-secret --ignore-not-found
kubectl create secret generic django-secret \
  --from-literal=SECRET_KEY=<your_secret_key> \
  --from-literal=DATABASE_URL=postgres://test_k8s:OwOtBep9Frut@host.docker.internal:5432/test_k8s
```

После этого применяйте манифесты:

```bash
kubectl apply -f kubernetes
kubectl wait --for=condition=Available deploy/django-web --timeout=120s
```

После раскатки в списке сервисов останется только `ClusterIP` без публичных портов (`kubectl get service django`).

### Ingress

1. Включите ingress-контроллер (nginx) в minikube:
   ```bash
   minikube addons enable ingress
   ```
2. Добавьте запись в `/etc/hosts`, чтобы домен указывал на IP minikube:
   ```bash
   echo "$(minikube ip) star-burger.test" | sudo tee -a /etc/hosts
   ```
3. Примените манифесты:
   ```bash
   kubectl apply -f kubernetes
   kubectl wait --for=condition=Available deploy/django-web --timeout=120s
   ```
4. Откройте сайт: `http://star-burger.test/` (порт 80, без port-forward).

Примечания:
- Сервис `django` имеет тип `ClusterIP`, внешняя точка входа — только через Ingress.
- В `Deployment` выставлен `DEBUG="False"`. Для временного включения отладки измените значение и снова выполните `kubectl apply -f kubernetes`.
- На Docker-драйвере доступ с хоста на порт 80 удобнее организовать через `minikube tunnel` (запускать в отдельном терминале с правами root); в этом случае External IP ingress-контроллера будет `127.0.0.1`, достаточно записи `127.0.0.1 star-burger.test` в `/etc/hosts`.

### Очистка устаревших сессий

- Разовый под: `kubectl apply -f kubernetes/django-clearsessions-pod.yaml` (вызывает `manage.py clearsessions`, не рестартится).
- Регулярно (cronjob): `kubectl apply -f kubernetes/django-clearsessions-cronjob.yaml` — расписание `0 3 * * *`, `concurrencyPolicy: Forbid`, `startingDeadlineSeconds: 600`, `ttlSecondsAfterFinished: 120`.
- Принудительно создать job из cronjob: `kubectl create job django-clearsessions-once --from=cronjob/django-clearsessions`.
- Проверка: `kubectl get cronjobs`, `kubectl get jobs`, `kubectl logs job/django-clearsessions-once`.

### PostgreSQL через Helm (для minikube)
1. Установите Helm (`brew install helm`), добавьте репозиторий Bitnami:  
   `helm repo add bitnami https://charts.bitnami.com/bitnami && helm repo update`
2. Установите БД:  
   `helm install postgresql bitnami/postgresql --set auth.username=test_k8s,auth.password=OwOtBep9Frut,auth.database=test_k8s --wait`
3. Обновите секрет с новым `DATABASE_URL`:  
   ```
   kubectl delete secret django-secret --ignore-not-found
   kubectl create secret generic django-secret \
     --from-literal=SECRET_KEY=change-me-in-prod \
     --from-literal=DATABASE_URL=postgresql://test_k8s:OwOtBep9Frut@postgresql:5432/test_k8s
   ```
4. Примените миграции:  
   `kubectl delete job django-migrate --ignore-not-found && kubectl apply -f kubernetes/django-migrate-job.yaml && kubectl wait --for=condition=complete job/django-migrate --timeout=120s && kubectl logs job/django-migrate`
5. Перезапустите деплой приложения: `kubectl rollout restart deploy/django-web`.
6. Проверка psql-клиентом:  
   `kubectl run postgresql-client --rm --restart=Never --image registry-1.docker.io/bitnami/postgresql:latest --env="PGPASSWORD=OwOtBep9Frut" --command -- psql -h postgresql -U test_k8s -d test_k8s -p 5432 -c "SELECT 1;"`.

### Проверка
- Сервисы: `kubectl get svc django` — тип `ClusterIP`, никаких NodePort/LoadBalancer.
- Ingress: `kubectl get ingress django` — хост `star-burger.test`, порт 80.
- DEBUG: `kubectl get deploy django-web -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name==\"DEBUG\")].value}'` — должно быть `False`.
- Доступность: запустите временный порт-форвард `kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80 --address 127.0.0.1` и в другом терминале `curl -I -H 'Host: star-burger.test' http://127.0.0.1:8080`. После проверки остановите порт-форвард (`Ctrl+C`).
