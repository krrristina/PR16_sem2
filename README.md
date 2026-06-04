# Практическая работа №16 (семестр 2)

## Выполнила: Сорокина К.С., ЭФМО-01-25

## Тема: Публикация приложения в Kubernetes (минимальный манифест)

### Цель:

Освоить базовую публикацию контейнеризированного backend-приложения в Kubernetes, научиться описывать Deployment и Service, передавать конфигурацию через ConfigMap, настраивать readiness и liveness probes, применять манифесты через kubectl и проверять состояние Pod и Service.

## Технологии

- **Go** — язык реализации сервиса
- **Docker** — контейнеризация
- **Kubernetes (minikube v1.35.1)** — оркестрация контейнеров
- **kubectl** — CLI для управления кластером
- **ConfigMap, Deployment, Service** — Kubernetes-ресурсы

## Kubernetes-стенд

Использован **minikube** — локальный однонодовый Kubernetes-кластер для разработки и учебных целей.

```
minikube start
kubectl cluster-info
kubectl get nodes
```

Нода: `minikube Ready control-plane v1.35.1`

![](https://github.com/krrristina/PR16_sem2/blob/main/screenshots/установка%20minkube.png)

## Структура проекта

```
PR16_sem2/
├── services/
│   └── tasks/
│       ├── Dockerfile
│       ├── go.mod
│       └── cmd/server/main.go    ← сервис с /health endpoint
├── deploy/
│   └── k8s/
│       ├── configmap.yaml        ← конфигурация приложения
│       ├── deployment.yaml       ← Deployment с probes
│       └── service.yaml          ← ClusterIP Service
└── README.md
```

---

## Подготовка Docker-образа

Образ собирается внутри Docker-демона minikube, чтобы кластер мог его использовать без внешнего registry:

```bash
eval $(minikube docker-env)
docker build -t techip-tasks:0.1 services/tasks/
```

Проверка:
```
techip-tasks:0.1    ea743d61e20e    17.1MB
```

`imagePullPolicy: IfNotPresent` в Deployment гарантирует что Kubernetes не будет пытаться скачать образ из внешнего registry — он возьмёт его из локального Docker minikube.

![](https://github.com/krrristina/PR16_sem2/blob/main/screenshots/запуск%20контейнеров.png)

![](https://github.com/krrristina/PR16_sem2/blob/main/screenshots/проверка%20образа.png)

## Манифесты

### ConfigMap (deploy/k8s/configmap.yaml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tasks-config
data:
  TASKS_PORT: "8082"
  AUTH_BASE_URL: "http://auth:8081"
  LOG_LEVEL: "info"
  INSTANCE_ID: "tasks-k8s"
```

ConfigMap хранит несекретную конфигурацию приложения отдельно от образа. Один и тот же образ можно запускать в разных окружениях, передавая разные ConfigMap. Параметры подгружаются в контейнер через `envFrom`.

### Deployment (deploy/k8s/deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tasks
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tasks
  template:
    metadata:
      labels:
        app: tasks
    spec:
      containers:
        - name: tasks
          image: techip-tasks:0.1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8082
          envFrom:
            - configMapRef:
                name: tasks-config
          readinessProbe:
            httpGet:
              path: /health
              port: 8082
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8082
            initialDelaySeconds: 10
            periodSeconds: 10
```

**Пояснение параметров:**

| Параметр | Значение |
|---|---|
| `replicas: 1` | Один экземпляр приложения |
| `imagePullPolicy: IfNotPresent` | Использовать локальный образ |
| `envFrom.configMapRef` | Подгрузить все переменные из ConfigMap |
| `readinessProbe` | Проверка готовности: GET /health каждые 5 сек |
| `livenessProbe` | Проверка жизнеспособности: GET /health каждые 10 сек |

**Readiness probe** — пока не проходит, Kubernetes не направляет трафик на Pod. Важно при медленном старте приложения.

**Liveness probe** — если начинает стабильно проваливаться, Kubernetes перезапускает контейнер. Защита от зависших процессов.

### Service (deploy/k8s/service.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: tasks
spec:
  type: ClusterIP
  selector:
    app: tasks
  ports:
    - protocol: TCP
      port: 8082
      targetPort: 8082
```

Service типа `ClusterIP` обеспечивает стабильную точку доступа к Pod внутри кластера. IP-адрес Pod может меняться, но IP Service остаётся неизменным.

---

## Применение манифестов

```bash
kubectl apply -f deploy/k8s/configmap.yaml
kubectl apply -f deploy/k8s/deployment.yaml
kubectl apply -f deploy/k8s/service.yaml
```

Результат:
```
configmap/tasks-config created
deployment.apps/tasks created
service/tasks created
```

![](https://github.com/krrristina/PR16_sem2/blob/main/screenshots/проверка%20манифестов.png)

## Проверки

### Состояние Pod, Deployment, Service

```bash
kubectl get pods
kubectl get deployment
kubectl get svc
```

Результат:
```
NAME                    READY   STATUS    RESTARTS   AGE
tasks-7d6cdd64bc-lp48x  1/1     Running   0          31s

NAME    READY   UP-TO-DATE   AVAILABLE
tasks   1/1     1            1

NAME         TYPE        CLUSTER-IP       PORT(S)
tasks        ClusterIP   10.101.43.181    8082/TCP
```

![](https://github.com/krrristina/PR16_sem2/blob/main/screenshots/проверки.png)

### Логи контейнера

```bash
kubectl logs tasks-7d6cdd64bc-lp48x
```

```
2026/06/04 07:46:07 tasks service started on :8082 instance = tasks-k8s
```


### Доступ через port-forward

```bash
kubectl port-forward svc/tasks 8082:8082
```

```
Forwarding from 127.0.0.1:8082 -> 8082
```

![](https://github.com/krrristina/PR16_sem2/blob/main/screenshots/проверка%20port-forward%201.png)

Проверка:
```bash
curl -i http://localhost:8082/health
```

Ответ:
```
HTTP/1.1 200 OK
X-Instance-Id: tasks-k8s
{"instance":"tasks-k8s","status":"ok"}
```
![](https://github.com/krrristina/PR16_sem2/blob/main/screenshots/curl.png)

## Масштабирование

```bash
kubectl scale deployment tasks --replicas=2
kubectl get pods
```

```
NAME                     READY   STATUS    RESTARTS   AGE
tasks-7d6cdd64bc-lp48x   1/1     Running   0          7m39s
tasks-7d6cdd64bc-nnjxj   1/1     Running   0          11s
```

Kubernetes автоматически создал второй Pod. Возврат к одной реплике:

```bash
kubectl scale deployment tasks --replicas=1
```

![](https://github.com/krrristina/PR16_sem2/blob/main/screenshots/масштабирование.png)

## Удаление ресурсов

```bash
kubectl delete -f deploy/k8s/service.yaml
kubectl delete -f deploy/k8s/deployment.yaml
kubectl delete -f deploy/k8s/configmap.yaml
```

---

## Ответы на контрольные вопросы

**Что такое Kubernetes и для чего он используется?**
Kubernetes — система оркестрации контейнеров. Решает задачи запуска, масштабирования, перезапуска при сбоях и управления конфигурацией контейнеризированных приложений в стандартизированной среде.

**Чем Pod отличается от Deployment?**
Pod — минимальная единица запуска, один или несколько контейнеров. Deployment — управляющий объект, который следит за тем чтобы нужное число Pod всегда работало. Если Pod упадёт — Deployment создаст новый.

**Почему приложение публикуют через Deployment, а не через одиночный Pod?**
Одиночный Pod не восстанавливается при сбое — его нужно создавать вручную. Deployment автоматически поддерживает нужное число реплик и управляет обновлениями.

**Зачем нужен Service?**
Pod не имеет стабильного IP — при перезапуске адрес меняется. Service предоставляет постоянную точку доступа и балансирует нагрузку между Pod с нужными метками.

**Что такое ConfigMap?**
Объект Kubernetes для хранения несекретной конфигурации приложения. Позволяет отделить конфигурацию от образа — один образ работает в разных окружениях с разными ConfigMap.

**Чем ConfigMap отличается от Secret?**
ConfigMap хранит данные в открытом виде. Secret — в base64 и с ограниченным доступом. Secret используется для паролей, токенов, ключей. ConfigMap — для несекретных параметров.

**Для чего используется readiness probe?**
Проверяет готовность приложения принимать трафик. Пока probe не проходит — Service не направляет запросы на Pod. Нужна при медленном старте приложения.

**Для чего используется liveness probe?**
Проверяет что приложение не зависло. Если probe стабильно проваливается — Kubernetes перезапускает контейнер.

**Почему важно использовать фиксированный тег образа?**
Тег `latest` не позволяет понять какая именно версия запущена. Фиксированный тег (`0.1` или commit hash) делает деплой воспроизводимым и упрощает откат.

**Зачем нужен kubectl port-forward?**
Пробрасывает порт Service или Pod на локальную машину. Позволяет обратиться к приложению внутри кластера без публичного IP или LoadBalancer.

**Что делает kubectl scale deployment?**
Изменяет число реплик Deployment. Kubernetes автоматически создаёт или удаляет Pod до нужного числа.

**Почему публикация в Kubernetes считается декларативной?**
Вы описываете желаемое состояние системы в YAML-манифестах, а не последовательность действий. Kubernetes сам принимает решения как привести текущее состояние к желаемому.
