# Выполнено ДЗ №1

 - [x] Основное ДЗ
 - [x] Задание со *

## В процессе сделано:
 - Установлен kubectl, minikube
 - minikube запущен, исследован на отказоустойчивость (удаление всех docker-контенеров внутри minikube, удаление pod'ов из namespace kube-system).
 - Исследованы причины автовосстановления pod'ов kube-apiserver и coredns. 
    Можно назвать разные причины для разных уровней абстракций:
    Средствами ОС Linux видим, что pod'ы запускаются с помощью containerd, который контролируется systemd.
    Средствами kubectl describe pods видим, что: 
     - в обоих pod'ах Events события "Created container" и "Started container" приходят From: kubelet
     - у pod'а kube-apiserver-minikube поле Controlled By:  Node/minikube
     - у pod'а coredns поле Controlled By:  ReplicaSet/coredns-74ff55c5b
 - Приложение web: 
     - создан Dockerfile, 
     - с'build'ин image и опубликован на github,
     - создан манифест и скормлен kubectl,
     - создан и запущен pod,
 - Приложение hipster-frontend:
     - собран image hipster-frontend и опубликован на github,
     - в ad-hoc режиме сгенерен манифест,
     - создан исправленный манифест,
     - создан и запущен pod
 
## Как запустить проект:
  ```minikube start &&```
  ```kubectl apply -f web-pod.yaml &&```
  ```kubectl apply -f frontend-pod-healthy.yaml```

## Как проверить работоспособность:
  ```kubectl port-forward --address 0.0.0.0 pod/web 8000:8000```
 - Перейти по ссылке http://localhost:8080
  ```kubectl get pod frontend```
 - frontend должен находиться в статусе Running

## PR checklist:
 - [x] Выставлен label с темой домашнего задания
