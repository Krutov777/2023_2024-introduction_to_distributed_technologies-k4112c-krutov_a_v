University: [ITMO University](https://itmo.ru/ru/)
Faculty: [FICT](https://fict.itmo.ru)
Course: [Introduction to distributed technologies](https://github.com/itmo-ict-faculty/introduction-to-distributed-technologies)
Year: 2023/2024
Group: K4112c
Author: Krutov Alexander Vladimirovich
Lab: Lab3
Date of create: 15.11.2023
Date of finished: <none>



# Лабораторная работа №3 "Сертификаты и "секреты" в Minikube, безопасное хранение данных."

# Ход выполнения работы

## Запуск Minikube:
```
minikube start
```
![minikube start](images/minikube_start.png)

Команда "minikube start" используется для запуска локального кластера Kubernetes с использованием Minikube. После выполнения этой команды произойдет следующее:

Проверка системных требований: Minikube проверит, удовлетворяет ли ваша система минимальным требованиям для запуска Kubernetes. Это включает в себя проверку наличия гипервизора (например, VirtualBox или Hyper-V) и доступных ресурсов на вашем компьютере.

Загрузка образа: Если образ Minikube еще не был загружен на ваш компьютер, команда автоматически загрузит его. Этот образ содержит минимальное окружение Kubernetes, необходимое для работы Minikube.

Создание виртуальной машины: Minikube создаст виртуальную машину (VM) с использованием выбранного гипервизора. Эта виртуальная машина будет служить управляемым окружением Kubernetes.

Установка Kubernetes: Minikube установит выбранную версию Kubernetes внутри виртуальной машины. Версия Kubernetes может быть указана в командной строке, либо будет использована версия по умолчанию.

Запуск кластера: После успешного выполнения всех предыдущих шагов, Minikube запустит кластер Kubernetes внутри виртуальной машины. Кластер будет готов к использованию.

Настройка kubectl: Minikube настроит инструмент командной строки kubectl так, чтобы он указывал на ваш запущенный локальный кластер. Это позволит вам управлять кластером с помощью kubectl.

## Создание сертификата:
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=krutovapp.com"
```

После выполнения этой операции появляется 2 файла `tls.crt` и `tls.key`

## Создание YAML файла:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  REACT_APP_USERNAME: "KAV"
  REACT_APP_COMPANY_NAME: "ITMO"

---
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: itdt-contained-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: itdt-contained-frontend
  template:
    metadata:
      labels:
        app: itdt-contained-frontend
    spec:
      containers:
        - name: itdt-contained-frontend
          image: ifilyaninitmo/itdt-contained-frontend:master
          env:
            - name: REACT_APP_USERNAME
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: REACT_APP_USERNAME
            - name: REACT_APP_COMPANY_NAME
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: REACT_APP_COMPANY_NAME

---
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  type: ClusterIP
  ports:
    - port: 3000
      targetPort: 3000
      protocol: TCP
  selector:
    app: itdt-contained-frontend
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: kubernetes.io/tls
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----

    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN PRIVATE KEY-----

    -----END PRIVATE KEY-----
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  tls:
    - hosts:
        - krutovapp
      secretName: app-secret
  rules:
    - host: krutovapp
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 3000
```

Этот YAML файл описывает различные ресурсы Kubernetes, которые могут быть использованы для развертывания приложения. Вот краткое описание каждого блока:

1. **ConfigMap (Конфигурационная карта)**:
   - **Имя**: `app-config`
   - **Данные**: Содержит ключ-значение для параметров, таких как `REACT_APP_USERNAME` (значение: "KAV") и `REACT_APP_COMPANY_NAME` (значение: "ITMO").

2. **ReplicaSet (Набор реплик)**:
   - **Имя**: `itdt-contained-frontend`
   - **Спецификация**: Создает и управляет репликами приложения `itdt-contained-frontend`.
   - **Контейнеры**: Использует образ `ifilyaninitmo/itdt-contained-frontend:master` и передает переменные среды из ConfigMap для `REACT_APP_USERNAME` и `REACT_APP_COMPANY_NAME`.

3. **Service (Сервис)**:
   - **Имя**: `app-service`
   - **Тип**: ClusterIP (доступен только внутри кластера)
   - **Порт**: Пробрасывает трафик с порта 3000 на поды, соответствующие Selector'у `app: itdt-contained-frontend`.

4. **Secret (Секрет)**:
   - **Имя**: `app-secret`
   - **Тип**: `kubernetes.io/tls`
   - **Данные**: Содержит TLS-сертификат и приватный ключ для использования в безопасной связи.

5. **Ingress (Вход)**:
   - **Имя**: `app-ingress`
   - **Спецификация TLS**: Ассоциирует секрет `app-secret` с хостом `krutovapp.com` для обеспечения безопасного соединения.
   - **Правила маршрутизации**: Настроены для перенаправления трафика с хоста `krutovapp.com` на сервис `app-service` через путь `/`.

## Применим файл конфигурации:
```
minikube kubectl apply -f app.yaml
```

Команда minikube kubectl apply -f app.yaml представляет собой команду для развертывания приложения в локальном кластере Kubernetes, управляемом Minikube.



## Включим ingress
```
minikube addons enable ingress
```

# Подстановка ip адреса:
```
echo "$(minikube ip) krutovapp.com" | sudo tee -a /etc/hosts
```

![substiotution ip](images/substitution_ip.png)

# Интерфейс приложения:
![WEB SITE](images/app.png)


# Сертификат:
![sertificate](images/serificate.png)

# Схема
![Schema](images/image9.jpg)
