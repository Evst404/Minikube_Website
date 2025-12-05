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
