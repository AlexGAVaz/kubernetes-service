### Установка NGINX Ingress Controller

Для установки NGINX Ingress Controller с использованием Helm, выполните следующие действия. Эти команды используют официальный Helm чарт NGINX Ingress Controller.

1. **Добавьте репозиторий чарта NGINX Ingress Controller**:
   ```bash
   helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
   helm repo update
   ```

2. **Установите NGINX Ingress Controller**:
   Вы можете установить NGINX Ingress Controller в пространстве имен `nginx-ingress`. Если вы хотите использовать другое пространство имен, замените `nginx-ingress` на желаемое имя пространства имен.
   ```bash
   helm install nginx-ingress ingress-nginx/ingress-nginx \
     --create-namespace \
     --namespace nginx-ingress \
     --set controller.replicaCount=2 \
     --set controller.nodeSelector."kubernetes\.io/os"=linux \
     --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux
   ```

   В этом примере устанавливается NGINX Ingress Controller с двумя репликами и задается node selector для Linux нод. Вы можете адаптировать параметры `--set` в соответствии с вашими требованиями.

3. **Проверка установки NGINX Ingress Controller**:
   Проверьте, что поды NGINX Ingress Controller запущены:
   
   ```bash
   kubectl get pods -n nginx-ingress
   ```

### Настройка Ingress для сервисов

- Создайте манифест `ingress.yaml`, указывая правила маршрутизации для ваших сервисов. Пример:

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: bsai-ingress
    spec:
      ingressClassName: nginx
      rules:
        - http:
            paths:
              - path: /bsai/core/
                pathType: Prefix
                backend:
                  service:
                    name: backstage-ai
                    port:
                      number: 8080
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: monitoring
                    port:
                      number: 8080
    ```

### Проверка установки NGINX Ingress Controller

После установки проверьте статус подов NGINX Ingress Controller:

```bash
kubectl get pods -n nginx-ingress
```

И убедитесь, что сервис NGINX Ingress Controller получил внешний IP-адрес (если вы используете MetalLB или другой механизм для предоставления внешних IP-адресов для сервисов типа `LoadBalancer`):

```bash
kubectl get svc -n nginx-ingress
```
