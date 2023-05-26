# patroni-consul-k8s
## Deploy Patroni with Consul in k8s

### Добавляем helm chart Consul:
```
helm repo add hashicorp https://helm.releases.hashicorp.com
```
### Устанавливаем Consul:
```
helm install --values consul/values.yaml consul hashicorp/consul --create-namespace --namespace consul --version "1.1.0"
```
### Проверяем pods:
```
kubectl get pods --namespace consul
```
```
NAME                                           READY   STATUS    RESTARTS   AGE
consul-connect-injector-6fc8d669b8-2n82l       1/1     Running   0          2m34s
consul-server-0                                1/1     Running   0          2m34s
consul-webhook-cert-manager-64889c4964-wxc9b   1/1     Running   0          2m34s
```
### Задаем переменные для проверки работоспособности Consul:
```
export CONSUL_HTTP_TOKEN=$(kubectl get --namespace consul secrets/consul-bootstrap-acl-token --template={{.data.token}} | base64 -d)

export CONSUL_HTTP_ADDR=https://127.0.0.1:8501

export CONSUL_HTTP_SSL_VERIFY=false
```
### Открываем новое окно терминала и вводим команду port-forward: 
```
kubectl port-forward svc/consul-ui --namespace consul 8501:443
```
### В основном терминале вводим команду на проверку доступности сервисов:
```
curl -k \
    --header "X-Consul-Token: $CONSUL_HTTP_TOKEN" \
    $CONSUL_HTTP_ADDR/v1/catalog/services
```
### Получаем следующий output: 
```
 {"consul":[]}
```
### Проверяем доступность агентов Consul:
```
curl -k \

   --header "X-Consul-Token: $CONSUL_HTTP_TOKEN" \

   $CONSUL_HTTP_ADDR/v1/agent/members\?pretty
```
### Получаем следующий output:
```
[
    {
        "Name": "consul-server-0",
        "Addr": "10.244.0.13",
        "Port": 8301,
        "Tags": {
            "acls": "1",
            "bootstrap": "1",
            "build": "1.14.0",
            "dc": "dc1",
            "ft_fs": "1",
            "ft_si": "1",
            "grpc_port": "8502",
            "id": "8016fc4d-767f-8552-b018-0812228bd135",
            "port": "8300",
            "raft_vsn": "3",
            "role": "consul",
            "segment": "",
            "use_tls": "1",
            "vsn": "2",
            "vsn_max": "3",
            "vsn_min": "2",
            "wan_join_port": "8302"
        },
        "Status": 1,
        "ProtocolMin": 1,
        "ProtocolMax": 5,
        "ProtocolCur": 2,
        "DelegateMin": 2,
        "DelegateMax": 5,
        "DelegateCur": 4
    }
]
```
### Переходим к деплою Patroni. Заходим в директорию patroni и собираем Docker image:
```
cd patroni && docker build -f Dockerfile.consul . -t patroni:consul
```
### Отправляем Patroni в деплой в k8s:
```
kubectl apply -f patroni_k8s.yaml
```
### Проверяем pods:
```
kubectl get pods 
```
### Output: 
    NAME            READY   STATUS    RESTARTS   AGE   
    patronidemo-0   1/1     Running   0          34s   
    patronidemo-1   1/1     Running   0          30s   
    patronidemo-2   1/1     Running   0          26s   

### Проваливаемся в контейнер и проверяем кластер:
```
kubectl exec -ti patronidemo-0 -- bash
```
postgres@patronidemo-0:~$ 
```
patronictl list
```
### Output: 
    + Cluster: patronidemo (7186662553319358497) ----+----+-----------+
    | Member        | Host       | Role    | State   | TL | Lag in MB |
    +---------------+------------+---------+---------+----+-----------+
    | patronidemo-0 | 10.244.0.5 | Leader  | running |  1 |           |
    | patronidemo-1 | 10.244.0.6 | Replica | running |  1 |         0 |
    | patronidemo-2 | 10.244.0.7 | Replica | running |  1 |         0 |
    +---------------+------------+---------+---------+----+-----------+
 ### Проверяем регистрацию сервисов Patroni в Consul:
 ```
 consul catalog services
 ```
 ### Output:
 ```
 $ consul catalog services
consul
patronidemo
 ```
 ### Источники и документация:
 https://developer.hashicorp.com/consul/tutorials/get-started-kubernetes
 
 https://github.com/zalando/patroni/tree/master


