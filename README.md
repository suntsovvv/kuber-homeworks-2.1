# Домашнее задание к занятию «Хранение в K8s. Часть 1»

### Цель задания

В тестовой среде Kubernetes нужно обеспечить обмен файлами между контейнерам пода и доступ к логам ноды.

------

### Задание 1 



Создал манифест storage.yaml деплоймента , состоящего из двух контейнеров и обменивающихся данными:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: storage-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: storage
  template:
    metadata:
      labels:
        app: storage
    spec:
      containers:
      - name: busybox
        image: busybox:latest
        command: ['sh', '-c', 'while true; do date >> /data/output.txt; sleep 5; done']
        volumeMounts:
        - name: safedata
          mountPath: /data
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 8080
        env:
        - name: HTTP_PORT
          value: "8080"
        volumeMounts:
        - name: safedata
          mountPath: /data
      volumes:
      - name: safedata
        emptyDir: {}

```
Применил:
```bash
user@microk8s:~/kuber-homeworks-2.1$ kubectl apply -f storage.yaml 
deployment.apps/storage-deployment created
user@microk8s:~/kuber-homeworks-2.1$ kubectl get deployments.apps 
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
storage-deployment   1/1     1            1           9s
user@microk8s:~/kuber-homeworks-2.1$ kubectl get pod
NAME                                  READY   STATUS    RESTARTS   AGE
storage-deployment-798d65b7fc-bncxj   2/2     Running   0          20s
```
Проверил создался ли файл output.txt в заданном расположении контейнера busybox:
```bash
user@microk8s:~/kuber-homeworks-2.1$ kubectl exec -it storage-deployment-798d65b7fc-bncxj -c busybox -- sh
/ # cat /data/output.txt 
Wed Nov  6 10:15:46 UTC 2024
Wed Nov  6 10:15:51 UTC 2024
Wed Nov  6 10:15:56 UTC 2024
Wed Nov  6 10:16:01 UTC 2024
Wed Nov  6 10:16:06 UTC 2024
Wed Nov  6 10:16:11 UTC 2024
Wed Nov  6 10:16:16 UTC 2024
Wed Nov  6 10:16:21 UTC 2024
Wed Nov  6 10:16:26 UTC 2024
Wed Nov  6 10:16:31 UTC 2024
Wed Nov  6 10:16:36 UTC 2024
Wed Nov  6 10:16:41 UTC 2024
Wed Nov  6 10:16:46 UTC 2024
Wed Nov  6 10:16:51 UTC 2024
/ # 
```
Проверил есть ли файл output.txt с содержимым в заданном расположении контейнера multitool:
```bash
user@microk8s:~/kuber-homeworks-2.1$ kubectl exec -it storage-deployment-798d65b7fc-bncxj -c multitool -- sh
/ # tail -f /data/output.txt 
Wed Nov  6 10:16:46 UTC 2024
Wed Nov  6 10:16:51 UTC 2024
Wed Nov  6 10:16:56 UTC 2024
Wed Nov  6 10:17:01 UTC 2024
Wed Nov  6 10:17:06 UTC 2024
Wed Nov  6 10:17:11 UTC 2024
Wed Nov  6 10:17:16 UTC 2024
Wed Nov  6 10:17:21 UTC 2024
Wed Nov  6 10:17:26 UTC 2024
Wed Nov  6 10:17:31 UTC 2024
Wed Nov  6 10:17:36 UTC 2024
Wed Nov  6 10:17:41 UTC 2024
Wed Nov  6 10:17:46 UTC 2024
^C
/ # 
```

------

### Задание 2


Создал манифест read-logs.yaml-DaemonSet приложения, которое может прочитать логи ноды:
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: read-logs
spec:
  selector:
    matchLabels:
      name: read-logs
  template:
    metadata:
      labels:
        name: read-logs
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        ports:
        volumeMounts:
        - name: varlog
          mountPath: /input
          readOnly: true
        securityContext:

      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```
Запустил:
```bash
user@microk8s:~/kuber-homeworks-2.1$ kubectl apply -f read-logs.yaml 
daemonset.apps/read-logs created
user@microk8s:~/kuber-homeworks-2.1$ kubectl get deployments.apps 
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
storage-deployment   1/1     1            1           84m
user@microk8s:~/kuber-homeworks-2.1$ kubectl get pod
NAME                                  READY   STATUS    RESTARTS   AGE
read-logs-6f2fb                       1/1     Running   0          22s
storage-deployment-798d65b7fc-bncxj   2/2     Running   0          84m
```
Проверил читается ли файл syslog ноды из пода:
```bash
user@microk8s:~/kuber-homeworks-2.1$ kubectl exec -it daemonsets/read-logs -c multitool -- sh
/ # tail -f /input/syslog
2024-11-06T11:39:57.279177+00:00 microk8s systemd[1]: run-containerd-runc-k8s.io-801eaba2fca3768b60ccadbbbeb6bab586ff8a2f7e6980ad5ad00379ea7fb601-runc.qhdV7M.mount: Deactivated successfully.
2024-11-06T11:39:58.967696+00:00 microk8s systemd[1]: Cannot find unit for notify message of PID 2994599, ignoring.
2024-11-06T11:39:59.146381+00:00 microk8s systemd[1]: Cannot find unit for notify message of PID 2994668, ignoring.
2024-11-06T11:40:02.794983+00:00 microk8s systemd[1]: Starting sysstat-collect.service - system activity accounting tool...
2024-11-06T11:40:02.800051+00:00 microk8s systemd[1]: sysstat-collect.service: Deactivated successfully.
2024-11-06T11:40:02.800389+00:00 microk8s systemd[1]: Finished sysstat-collect.service - system activity accounting tool.
2024-11-06T11:40:09.375738+00:00 microk8s systemd[1]: Cannot find unit for notify message of PID 2994885, ignoring.
2024-11-06T11:40:09.547596+00:00 microk8s systemd[1]: Cannot find unit for notify message of PID 2994953, ignoring.
2024-11-06T11:40:14.753213+00:00 microk8s systemd[1]: Cannot find unit for notify message of PID 2995089, ignoring.
2024-11-06T11:40:30.171702+00:00 microk8s systemd[1]: Cannot find unit for notify message of PID 2995525, ignoring.
2024-11-06T11:40:37.278038+00:00 microk8s systemd[1]: run-containerd-runc-k8s.io-801eaba2fca3768b60ccadbbbeb6bab586ff8a2f7e6980ad5ad00379ea7fb601-runc.M8F76B.mount: Deactivated successfully.
2024-11-06T11:40:40.591118+00:00 microk8s systemd[1]: Cannot find unit for notify message of PID 2995817, ignoring.
^C
/ # 
```

------
