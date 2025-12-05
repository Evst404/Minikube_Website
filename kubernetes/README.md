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

Сервис будет доступен через `minikube service django --url` (для Docker-драйвера на macOS) или по NodePort/Ingress в других конфигурациях.

### Ingress

1. Включите ingress-контроллер в minikube:
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
