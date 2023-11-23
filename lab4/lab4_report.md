University: [ITMO University](https://itmo.ru/ru/) \
Faculty: [FICT](https://fict.itmo.ru) \
Course: [Introduction to distributed technologies](https://github.com/itmo-ict-faculty/introduction-to-distributed-technologies) \
Year: 2023/2024 \
Group: K4112с \
Author: Krutov Alexander Krutov \
Lab: Lab4 \
Date of create: 23.11.2023 \
Date of finished: <none>

## Устанавливаем `Calico` для Windows
    $ Invoke-WebRequest -Uri "https://github.com/projectcalico/calico/releases/download/v3.26.4/calicoctl-windows-amd64.exe" -OutFile kubectl-calico.exe
    $ copy .\kubectl-calico.exe 'C:\Program Files\Kubernetes\Minikube\'

## Запускаем 2 ноды `minikube` с `Calico`
    $ minikube start --network-plugin=cni --cni=calico --nodes 2 -p multinode

## Проверяем количество нод
    $ kubectl get nodes
```
NAME            STATUS   ROLES           AGE   VERSION
multinode       Ready    control-plane   23h   v1.27.4
multinode-m02   Ready    <none>          69s   v1.27.4
```

## Проверяем количество подов `Calico`
    $ kubectl get pods -l k8s-app=calico-node -A
```
NAMESPACE     NAME                READY   STATUS    RESTARTS       AGE
kube-system   calico-node-vb8th   1/1     Running   8 (86s ago)    23h
kube-system   calico-node-wq6mb   1/1     Running   10 (36s ago)   23h
```
## Укажем `label` для нод
    $ kubectl label node multinode rack=rack1 && kubectl label node multinode-m02 rack=rack2
```
node/multinode labeled
node/multinode-m02 labeled
```

## Проверим указанные `label` для нод
    $ kubectl get nodes -l rack=rack2
```
NAME            STATUS   ROLES    AGE     VERSION
multinode-m02   Ready    <none>   5m15s   v1.27.4
```

## Создаем `Calico` манифест
``` yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: node-1-ippool
spec:
  cidr: 192.168.20.0/24
  ipipMode: Always
  natOutgoing: true
  nodeSelector: rack == "rack1"

---

apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: node-2-ippool
spec:
  cidr: 192.168.30.0/24
  ipipMode: Always
  natOutgoing: true
  nodeSelector: rack == "rack2"
```

## Удалим `IPPOOL` по умолчанию
    $ kubectl-calico delete ippools default-ipv4-ippool

## Применяем манифест
    $ kubectl-calico apply -f -< calico.yaml --allow-version-mismatch
```
Successfully applied 2 'IPPool' resource(s)
```

## Проверяем `IPPOOL`
    $ kubectl-calico get ippools --allow-version-mismatch
```
NAME            CIDR              SELECTOR
node-1-ippool   192.168.30.0/24   rack == "rack1"
node-2-ippool   192.168.40.0/24   rack == "rack2"
```

## Создаем `configMap`, `deployment` и `service`
``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: react-env
data:
  REACT_APP_USERNAME: "Krutov777"
  REACT_APP_COMPANY_NAME: "FICT ITMO"

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: react
spec:
  replicas: 2
  selector:
    matchLabels:
      app: react
  template:
    metadata:
      labels:
        app: react
    spec:
      containers:
        - name: react
          image: ifilyaninitmo/itdt-contained-frontend:master
          - name: react-port
            containerPort: 3000
          env:
            - name: REACT_APP_USERNAME
              valueFrom:
                configMapKeyRef:
                  name: react-env
                  key: REACT_APP_USERNAME
            - name: REACT_APP_COMPANY_NAME
              valueFrom:
                configMapKeyRef:
                  name: react-env
                  key: REACT_APP_COMPANY_NAME

---

apiVersion: v1
kind: Service
metadata:
  name: react-service
spec:
  type: LoadBalancer
  selector:
    app: react
  ports:
    - port: 777
      name: react-port
      targetPort: react-port
      protocol: TCP
```

## Применяем `deployment` и `service`
    $ kubectl apply -f dep.yaml
```
configmap/react-env created
deployment.apps/react created
service/react-service created
```

## Узнаем запущенные ноды
    $ kubectl get pods -o wide
```
NAME                         READY   STATUS    RESTARTS      AGE   IP               NODE            NOMINATED NODE   READINESS GATES
react-74987d8dc6-g877z       1/1     Running   0             33s   192.168.30.193   multinode       <none>           <none>
react-74987d8dc6-pcd7k       1/1     Running   0             33s   192.168.40.128   multinode-m02   <none>           <none>
```

## Включаем туннелирование
    $ minikube service react-service -p multinode

## Приложение
![1](images/image1.png)
![2](images/image2.png)

## Container и Container IP
Они могут меняться, так как у нас есть два контейнера и `LoadBalancer` перенаправляет нас на один из них

## Узнаем имя соседнего пода
    $ kubectl exec react-74987d8dc6-g877z -- nslookup 192.168.40.128
```
Server:         10.96.0.10
Address:        10.96.0.10:53

128.40.168.192.in-addr.arpa     name = 192-168-40-128.react-service.default.svc.cluster.local
```

## Ping соседнего пода
    $ kubectl exec react-74987d8dc6-g877z -- ping 192-168-40-128.react-service.default.svc.cluster.local
```
PING 192-168-40-128.react-service.default.svc.cluster.local (192.168.40.128): 56 data bytes
64 bytes from 192.168.40.128: seq=0 ttl=62 time=0.092 ms
64 bytes from 192.168.40.128: seq=1 ttl=62 time=0.123 ms
64 bytes from 192.168.40.128: seq=2 ttl=62 time=0.162 ms
64 bytes from 192.168.40.128: seq=3 ttl=62 time=0.102 ms
```

## Схема
![3](images/image3.drawio.png)
