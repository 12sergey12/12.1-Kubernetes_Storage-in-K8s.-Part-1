### Домашнее задание к занятию «Хранение в K8s. Часть 1» Баранов Сергей

### Цель задания

В тестовой среде Kubernetes нужно обеспечить обмен файлами между контейнерам пода и доступ к логам ноды.

------

### Чеклист готовности к домашнему заданию

1. Установленное K8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключенным GitHub-репозиторием.

------

### Дополнительные материалы для выполнения задания

1. [Инструкция по установке MicroK8S](https://microk8s.io/docs/getting-started).
2. [Описание Volumes](https://kubernetes.io/docs/concepts/storage/volumes/).
3. [Описание Multitool](https://github.com/wbitt/Network-MultiTool).

------

### Задание 1 

**Что нужно сделать**

Создать Deployment приложения, состоящего из двух контейнеров и обменивающихся данными.

1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool.


[deployment.yaml]()

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-01
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apps
  template:
    metadata:
      labels:
        app: apps
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["/bin/sh", "-c", "while true;do date>>/data/date.txt;sleep 5;done"]
        volumeMounts:
          - mountPath: "/data"
            name: volume-01
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 80
          name: http
        - containerPort: 443
          name: https
        volumeMounts:
          - mountPath: "/data"
            name: volume-01
      volumes:
        - name: volume-01
          emptyDir: {}

```

2. Сделать так, чтобы busybox писал каждые пять секунд в некий файл в общей директории.

В контейнере busybox запущена команда while true;do date>>/data/date.txt;sleep 5;done.

3. Обеспечить возможность чтения файла контейнером multitool.

Был создан общий том volume-01, и подмонтирован в оба контейнера.

4. Продемонстрировать, что multitool может читать файл, который периодоически обновляется.

```
root@baranov:/home/baranovsa/kube-2.1# kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
deployment-01-f7fd9f79d-ntl6r   2/2     Running   0          67s
root@baranov:/home/baranovsa/kube-2.1#

root@baranov:/home/baranovsa/kube-2.1# kubectl exec -it deployment-01-f7fd9f79d-ntl6r -c multitool -->
/ # tail -f /data/date.txt
Fri Mar 15 15:00:10 UTC 2024
Fri Mar 15 15:00:15 UTC 2024
Fri Mar 15 15:00:20 UTC 2024
Fri Mar 15 15:00:25 UTC 2024
Fri Mar 15 15:00:30 UTC 2024
Fri Mar 15 15:00:35 UTC 2024
Fri Mar 15 15:00:40 UTC 2024
Fri Mar 15 15:00:45 UTC 2024
Fri Mar 15 15:00:50 UTC 2024
Fri Mar 15 15:00:55 UTC 2024
Fri Mar 15 15:01:00 UTC 2024
Fri Mar 15 15:01:05 UTC 2024
Fri Mar 15 15:01:10 UTC 2024
Fri Mar 15 15:01:15 UTC 2024
```

5. Предоставить манифесты Deployment в решении, а также скриншоты или вывод команды из п. 4.

[deployment.yaml]()


```
root@baranov:/home/baranovsa/kube-2.1# kubectl exec -it deployment-01-f7fd9f79d-ntl6r -c multitool -->
/ # tail -f /data/date.txt
Fri Mar 15 15:00:10 UTC 2024
Fri Mar 15 15:00:15 UTC 2024
Fri Mar 15 15:00:20 UTC 2024
Fri Mar 15 15:00:25 UTC 2024
Fri Mar 15 15:00:30 UTC 2024
Fri Mar 15 15:00:35 UTC 2024
Fri Mar 15 15:00:40 UTC 2024
Fri Mar 15 15:00:45 UTC 2024
Fri Mar 15 15:00:50 UTC 2024
Fri Mar 15 15:00:55 UTC 2024
Fri Mar 15 15:01:00 UTC 2024
Fri Mar 15 15:01:05 UTC 2024
Fri Mar 15 15:01:10 UTC 2024
Fri Mar 15 15:01:15 UTC 2024
```

------

### Задание 2

**Что нужно сделать**

Создать DaemonSet приложения, которое может прочитать логи ноды.

1. Создать DaemonSet приложения, состоящего из multitool.

[DaemonSet_multitool.yml]()

```
apiVersion: apps/v1 # Версия api изменилось и теперь выглядит так
kind: DaemonSet # Тип уже не деплоймент, а демонсет
metadata:
  name: logs-multitool # Имя демонсета
  namespace: default
spec:
  selector: # Селектор на наши метки
    matchLabels:
      app: logs-multitool
  template: # Шаблон ПОДа
    metadata:
      labels: # Метки, которые будут установлены при запуске ПОДа из этого шаблона
        app: logs-multitool
    spec:
      containers:
        - name: multitool
          image: wbitt/network-multitool
          volumeMounts: # Монтируем вольюм с именем log-volume в контейнер multitool
            - mountPath: "/log_data"
              name: log-volume
      volumes:
        - name: log-volume # Описываем вольюм хостпас с именем log-volume
          hostPath:
            path: /var/log

```

2. Обеспечить возможность чтения файла `/var/log/syslog` кластера MicroK8S.

```
root@baranov:/home/baranovsa/kube-2.1#  kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
logs-multitool-8vpwr             1/1     Running   0          26m
root@baranov:/home/baranovsa/kube-2.1# 
```
3. Продемонстрировать возможность чтения файла изнутри пода.

```
root@baranov:/home/baranovsa/kube-2.1# kubectl exec -n default logs-multitool-8vpwr -it -- bash
logs-multitool-8vpwr:/# 
logs-multitool-8vpwr:/# 
logs-multitool-8vpwr:/# ls -lah /
total 84K    
drwxr-xr-x    1 root     root        4.0K Mar 19 15:47 .
drwxr-xr-x    1 root     root        4.0K Mar 19 15:47 ..
drwxr-xr-x    1 root     root        4.0K Sep 14  2023 bin
drwx------    2 root     root        4.0K Sep 14  2023 certs
drwxr-xr-x    5 root     root         360 Mar 19 15:47 dev
drwxr-xr-x    1 root     root        4.0K Sep 14  2023 docker
drwxr-xr-x    1 root     root        4.0K Mar 19 15:47 etc
drwxr-xr-x    2 root     root        4.0K Aug  7  2023 home
drwxr-xr-x    1 root     root        4.0K Sep 14  2023 lib
drwxr-xr-x   14 root     root        4.0K Mar 18 17:02 log_data
drwxr-xr-x    5 root     root        4.0K Aug  7  2023 media
drwxr-xr-x    2 root     root        4.0K Aug  7  2023 mnt
drwxr-xr-x    2 root     root        4.0K Aug  7  2023 opt
dr-xr-xr-x  372 root     root           0 Mar 19 15:47 proc
drwx------    1 root     root        4.0K Mar 19 15:50 root
drwxr-xr-x    1 root     root        4.0K Mar 19 15:47 run
drwxr-xr-x    1 root     root        4.0K Sep 14  2023 sbin
drwxr-xr-x    2 root     root        4.0K Aug  7  2023 srv
dr-xr-xr-x   13 root     root           0 Mar 19 15:47 sys
drwxrwxrwt    2 root     root        4.0K Aug  7  2023 tmp
drwxr-xr-x    1 root     root        4.0K Aug  7  2023 usr
drwxr-xr-x    1 root     root        4.0K Sep 14  2023 var
logs-multitool-8vpwr:/# ls -lah /log_data/
total 72M    
drwxr-xr-x   14 root     root        4.0K Mar 18 17:02 .
drwxr-xr-x    1 root     root        4.0K Mar 19 15:47 ..
-rw-r--r--    1 root     root           0 Mar  2 12:51 alternatives.log
-rw-r--r--    1 root     root        3.7K Feb 23 09:45 alternatives.log.1
-rw-r--r--    1 root     root         492 Mar  8  2023 alternatives.log.2.gz
-rw-r--r--    1 root     root        3.6K Jan 19  2023 alternatives.log.3.gz
drwxr-xr-x    2 root     root        4.0K Mar 12 14:27 apt
-rw-r-----    1 root     adm        48.5K Mar 19 15:30 auth.log
-rw-r-----    1 root     adm       139.5K Mar 18 17:01 auth.log.1
-rw-r-----    1 root     adm         6.9K Mar 12 12:12 auth.log.2.gz
-rw-r-----    1 root     adm        23.7K Mar  2 17:02 auth.log.3.gz
-rw-r-----    1 root     adm         3.1K Feb 23 09:49 auth.log.4.gz
-rw-------    1 root     root       13.3K Mar 19 14:25 boot.log
-rw-------    1 root     root        6.6K Mar 18 12:54 boot.log.1
-rw-------    1 root     root        6.7K Mar 15 11:29 boot.log.2
-rw-------    1 root     root        6.6K Mar 13 12:27 boot.log.3
-rw-------    1 root     root       13.7K Mar 12 12:11 boot.log.4
-rw-------    1 root     root        6.7K Mar  8 05:25 boot.log.5
-rw-------    1 root     root        6.7K Mar  5 11:54 boot.log.6
-rw-------    1 root     root        2.8K Mar  4 12:24 boot.log.7
-rw-rw----    1 root     43           384 Mar 18 12:58 btmp
-rw-rw----    1 root     43          2.6K Feb 29 11:46 btmp.1
drwxr-xr-x    2 root     root        4.0K Mar 19 16:00 containers
drwxr-xr-x    2 root     root        4.0K Mar 18 17:01 cups
-rw-r-----    1 root     adm         2.9M Mar 19 16:15 daemon.log
-rw-r-----    1 root     adm        26.4M Mar 18 17:02 daemon.log.1
-rw-r-----    1 root     adm         1.3M Mar 12 12:12 daemon.log.2.gz
-rw-r-----    1 root     adm         3.4M Mar  2 17:02 daemon.log.3.gz
-rw-r-----    1 root     adm       141.3K Feb 25 08:32 daemon.log.4.gz
-rw-r-----    1 root     adm        21.9K Mar 19 16:14 debug
-rw-r-----    1 root     adm        73.4K Mar 18 16:57 debug.1
-rw-r-----    1 root     adm         5.4K Mar 12 12:11 debug.2.gz
-rw-r-----    1 root     adm        10.6K Mar  2 17:02 debug.3.gz
-rw-r-----    1 root     adm         5.1K Feb 25 08:32 debug.4.gz
-rw-r--r--    1 root     root         741 Mar 12 14:28 dpkg.log
-rw-r--r--    1 root     root      201.0K Feb 29 12:08 dpkg.log.1
-rw-r--r--    1 root     root        4.4K Mar  8  2023 dpkg.log.2.gz
-rw-r--r--    1 root     root       80.6K Jan 19  2023 dpkg.log.3.gz
-rw-r--r--    1 root     root       31.3K Feb 25 16:36 faillog
-rw-r--r--    1 root     root        4.3K Feb 23 09:47 fontconfig.log
drwx--x--x    2 root     124         4.0K Nov 23  2022 gdm3
drwxr-xr-x    3 root     root        4.0K Nov 23  2022 installer
drwxr-sr-x    3 root     nginx       4.0K Feb 25 09:23 journal
-rw-r-----    1 root     adm       123.3K Mar 19 15:59 kern.log
-rw-r-----    1 root     adm       238.4K Mar 18 17:01 kern.log.1
-rw-r-----    1 root     adm        56.0K Mar 12 12:12 kern.log.2.gz
-rw-r-----    1 root     adm       149.7K Mar  2 17:02 kern.log.3.gz
-rw-r-----    1 root     adm        46.6K Feb 25 08:32 kern.log.4.gz
-rw-rw-r--    1 root     43        285.4K Feb 25 16:36 lastlog
-rw-r-----    1 root     adm       148.0K Mar 19 15:59 messages
-rw-r-----    1 root     adm       495.1K Mar 18 17:01 messages.1
-rw-r-----    1 root     adm        75.6K Mar 12 12:12 messages.2.gz
-rw-r-----    1 root     adm       200.0K Mar  2 17:02 messages.3.gz
-rw-r-----    1 root     adm        61.0K Feb 25 08:32 messages.4.gz
drwxr-xr-x    2 119      127         4.0K Sep 23  2020 ntpstats
drwxr-xr-x   13 root     root        4.0K Mar 19 15:59 pods
drwx------    2 root     root        4.0K Nov 23  2022 private
drwxr-xr-x    3 root     root        4.0K Nov 23  2022 runit
drwx------    2 111      root        4.0K Sep 19  2021 speech-dispatcher
-rw-r-----    1 root     adm         3.0M Mar 19 16:15 syslog
-rw-r-----    1 root     adm        26.9M Mar 18 17:02 syslog.1
-rw-r-----    1 root     adm         1.4M Mar 12 12:12 syslog.2.gz
-rw-r-----    1 root     adm         3.7M Mar  2 17:02 syslog.3.gz
-rw-r-----    1 root     adm       214.8K Feb 25 08:32 syslog.4.gz
drwxr-x---    2 root     adm         4.0K Mar  2 12:51 unattended-upgrades
-rw-r-----    1 root     adm        32.3K Mar 19 14:40 user.log
-rw-r-----    1 root     adm       272.2K Mar 18 16:58 user.log.1
-rw-r-----    1 root     adm        19.8K Mar 12 12:12 user.log.2.gz
-rw-r-----    1 root     adm        41.3K Mar  2 17:02 user.log.3.gz
-rw-r-----    1 root     adm        11.1K Feb 25 08:32 user.log.4.gz
-rw-rw-r--    1 root     43         54.4K Mar 19 14:26 wtmp
logs-multitool-8vpwr:/# tail /log_data/syslog
Mar 19 23:15:30 baranov systemd[673]: run-containerd-runc-k8s.io-5ee543d51a4153ea259c0160c1bb668663bf91f7bf78a9f23969729022a67189-runc.7q5WHG.mount: Succeeded.
Mar 19 23:15:31 baranov systemd[1]: run-containerd-runc-k8s.io-20d5700238ddd086e580871687576c18f98990d7c685ae3b0a6bf5a313700db5-runc.KLGZw1.mount: Succeeded.
Mar 19 23:15:35 baranov systemd[1]: run-containerd-runc-k8s.io-20d5700238ddd086e580871687576c18f98990d7c685ae3b0a6bf5a313700db5-runc.Bdvw9F.mount: Succeeded.
Mar 19 23:15:35 baranov systemd[673]: run-containerd-runc-k8s.io-20d5700238ddd086e580871687576c18f98990d7c685ae3b0a6bf5a313700db5-runc.Bdvw9F.mount: Succeeded.
Mar 19 23:15:37 baranov gnome-shell[1325]: libinput error: client bug: timer event4 debounce: scheduled expiry is in the past (-121ms), your system is too slow
Mar 19 23:15:37 baranov gnome-shell[1325]: libinput error: client bug: timer event4 debounce short: scheduled expiry is in the past (-134ms), your system is too slow
Mar 19 23:15:38 baranov gnome-shell[1325]: libinput error: client bug: timer event4 debounce short: scheduled expiry is in the past (-9ms), your system is too slow
Mar 19 23:15:39 baranov systemd[1]: run-containerd-runc-k8s.io-5ee543d51a4153ea259c0160c1bb668663bf91f7bf78a9f23969729022a67189-runc.10SPX8.mount: Succeeded.
Mar 19 23:15:40 baranov systemd[1]: run-containerd-runc-k8s.io-5ee543d51a4153ea259c0160c1bb668663bf91f7bf78a9f23969729022a67189-runc.FK2Xe2.mount: Succeeded.
Mar 19 23:15:40 baranov systemd[673]: run-containerd-runc-k8s.io-5ee543d51a4153ea259c0160c1bb668663bf91f7bf78a9f23969729022a67189-runc.FK2Xe2.mount: Succeeded.
logs-multitool-8vpwr:/# exit

```

4. Предоставить манифесты Deployment, а также скриншоты или вывод команды из п. 2.


[DaemonSet_multitool.yml]()

```
root@baranov:/home/baranovsa/kube-2.1# kubectl exec -n default logs-multitool-8vpwr -it -- bash
logs-multitool-8vpwr:/# 
logs-multitool-8vpwr:/# tail /log_data/syslog
Mar 19 23:15:30 baranov systemd[673]: run-containerd-runc-k8s.io-5ee543d51a4153ea259c0160c1bb668663bf91f7bf78a9f23969729022a67189-runc.7q5WHG.mount: Succeeded.
Mar 19 23:15:31 baranov systemd[1]: run-containerd-runc-k8s.io-20d5700238ddd086e580871687576c18f98990d7c685ae3b0a6bf5a313700db5-runc.KLGZw1.mount: Succeeded.
Mar 19 23:15:35 baranov systemd[1]: run-containerd-runc-k8s.io-20d5700238ddd086e580871687576c18f98990d7c685ae3b0a6bf5a313700db5-runc.Bdvw9F.mount: Succeeded.
Mar 19 23:15:35 baranov systemd[673]: run-containerd-runc-k8s.io-20d5700238ddd086e580871687576c18f98990d7c685ae3b0a6bf5a313700db5-runc.Bdvw9F.mount: Succeeded.
Mar 19 23:15:37 baranov gnome-shell[1325]: libinput error: client bug: timer event4 debounce: scheduled expiry is in the past (-121ms), your system is too slow
Mar 19 23:15:37 baranov gnome-shell[1325]: libinput error: client bug: timer event4 debounce short: scheduled expiry is in the past (-134ms), your system is too slow
Mar 19 23:15:38 baranov gnome-shell[1325]: libinput error: client bug: timer event4 debounce short: scheduled expiry is in the past (-9ms), your system is too slow
Mar 19 23:15:39 baranov systemd[1]: run-containerd-runc-k8s.io-5ee543d51a4153ea259c0160c1bb668663bf91f7bf78a9f23969729022a67189-runc.10SPX8.mount: Succeeded.
Mar 19 23:15:40 baranov systemd[1]: run-containerd-runc-k8s.io-5ee543d51a4153ea259c0160c1bb668663bf91f7bf78a9f23969729022a67189-runc.FK2Xe2.mount: Succeeded.
Mar 19 23:15:40 baranov systemd[673]: run-containerd-runc-k8s.io-5ee543d51a4153ea259c0160c1bb668663bf91f7bf78a9f23969729022a67189-runc.FK2Xe2.mount: Succeeded.
logs-multitool-8vpwr:/# exit

```
------

### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl`, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

------
