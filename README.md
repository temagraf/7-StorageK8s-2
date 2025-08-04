# Домашнее задание к занятию «Хранение в K8s. Часть 2»

### Цель задания
В тестовой среде Kubernetes нужно создать PV и продемонстрировать запись и хранение файлов.

### Чеклист готовности к домашнему заданию
1. Установленное K8s-решение (MicroK8S)
2. Установленный локальный kubectl
3. Редактор YAML-файлов с подключенным git-репозиторием

## Задание 1: Создание локального PV

### Подготовка окружения

1. Создание директории для данных:
   
```bash
sudo mkdir /mnt/data
sudo chmod 777 /mnt/data
```
### Манифесты

Все манифесты находятся в директории [manifests/task1](https://github.com/temagraf/7-StorageK8s-2/tree/main/manifests/task1)  
StorageClass [storage-class.yaml](https://github.com/temagraf/7-StorageK8s-2/blob/main/manifests/task1/storage-class.yaml)    
PersistentVolume [pv.yaml](https://github.com/temagraf/7-StorageK8s-2/blob/main/manifests/task1/pv.yaml)  
PersistentVolumeClaim [pvc.yaml](https://github.com/temagraf/7-StorageK8s-2/blob/main/manifests/task1/pvc.yaml)   
Deployment [deployment.yaml](https://github.com/temagraf/7-StorageK8s-2/blob/main/manifests/task1/deployment.yaml)  

Проверка работоспособности

### 1 Создание и проверка StorageClass и PV:

```bash
kubectl apply -f manifests/task1/storage-class.yaml
kubectl apply -f manifests/task1/pv.yaml
kubectl get pv
```
![image](https://github.com/temagraf/7-StorageK8s-2/blob/main/1-1images%3A1-pv-status.png)

### 2 Создание PVC и проверка:

```bash
kubectl apply -f manifests/task1/pvc.yaml
kubectl get pvc
```
![image](https://github.com/temagraf/7-StorageK8s-2/blob/main/1-2%20статус%20PVC.png)  

### 3 Запуск Deployment и проверка подов:

```bash
kubectl apply -f manifests/task1/deployment.yaml
kubectl get pods -l app=storage-test
```
![image](https://github.com/temagraf/7-StorageK8s-2/blob/main/1-3%20статус%20подов.png)

### 4 Проверка записи данных:
```bash
POD_NAME=$(kubectl get pods -l app=storage-test -o jsonpath="{.items[0].metadata.name}")
kubectl exec -it $POD_NAME -c multitool -- cat /data/output.txt
```
![image](https://github.com/temagraf/7-StorageK8s-2/blob/main/1-4%20Проверяем%20данные.png)

![image](https://github.com/temagraf/7-StorageK8s-2/blob/main/1-5%20статус.png)

![image](https://github.com/temagraf/7-StorageK8s-2/blob/main/1-6%20создание%20нового%20пода.png)

![image](https://github.com/temagraf/7-StorageK8s-2/blob/main/1-7данные%20в%20новом%20поде.png) 

### Проверка сохранности данных

### 1 Удаление Deployment и PVC:
```
kubectl delete -f manifests/task1/deployment.yaml
kubectl delete -f manifests/task1/pvc.yaml
kubectl get pv
```
![image](https://github.com/temagraf/7-StorageK8s-2/blob/main/1-8%20сохранность%20файла%20на%20ноде.png)


### 2 Проверка данных на ноде:

```bash
sudo cat /mnt/data/output.txt
```
![image](https://github.com/temagraf/7-StorageK8s-2/blob/main/1-9%20после%20удаления%20PV.png) 


### В результате имеем:  

- PersistentVolume успешно создан и подключен  
- PersistentVolumeClaim успешно привязан к PV  
- Pod с контейнерами busybox и multitool успешно запущен  
- Данные успешно записываются в общую директорию  
- Данные сохраняются при пересоздании пода  
- Данные остаются на ноде после удаления deployment и PVC  

### Объяснение сохранности данных
После удаления deployment и PVC данные сохранились на ноде, потому что:  
- PV был создан с параметром persistentVolumeReclaimPolicy: Retain  
- Данные физически хранятся на ноде в директории /mnt/data  
- Удаление PVC не влияет на физические данные при политике Retain  
- Файлы остаются доступными на ноде даже после удаления всех объектов Kubernetes, так как они хранятся в локальной файловой системе ноды.  

## Задание 2: Создание Deployment с NFS

### Подготовка окружения

Все манифесты находятся в директории [manifests/task2](https://github.com/temagraf/7-StorageK8s-2/tree/main/manifests/task2)  
[nfs-deployment.yaml](https://github.com/temagraf/7-StorageK8s-2/blob/main/manifests/task2/nfs-deployment.yaml)    
[nfs-pvc.yaml](https://github.com/temagraf/7-StorageK8s-2/blob/main/manifests/task2/nfs-pvc.yaml)  
 

### 1 Установка и настройка NFS:

```bash
sudo apt install -y nfs-kernel-server
sudo mkdir -p /srv/nfs/kubedata
sudo chown -R nobody:nogroup /srv/nfs/kubedata
sudo chmod 777 /srv/nfs/kubedata
```

### 2 Настройка NFS-сервера:

```bash
echo "/srv/nfs/kubedata *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
sudo exportfs -ra
```

### Проверка работоспособности

### 1 Статус NFS-сервера:

```bash 
sudo systemctl status nfs-kernel-server
```
![image](https://github.com/temagraf/7-StorageK8s-2/blob/main/2-1.png)

### 2 Проверка работы NFS:

- После установки NFS:

```bash
sudo systemctl status nfs-kernel-server
```

- После установки NFS-provisioner:
```bash
kubectl get sc
```
- После создания PVC:
```bash
kubectl get pvc
```
- После создания пода:
```bash
kubectl get pods -l app=multitool-nfs
```

- После проверки работы:
```bash
kubectl exec -it $POD_NAME -- cat /data/test.txt
sudo cat /srv/nfs/kubedata/test.txt
```

![image](https://github.com/temagraf/7-StorageK8s-2/blob/main/2-2.png)


### Результаты
- NFS-сервер успешно настроен
- PVC успешно создан и подключен
- Pod с multitool успешно запущен
- Данные успешно записываются и читаются




# Ответ на факультативный вопрос: в чём преимущества kustomize вместо helm?  
  
### ✅ Ну насколько я понял разницу между Helm и Kustomize, то Helm - это как готовый продукт со всей начинкой с которым мы получаем полный набор инструкций и компонентов, можем настраивать параметры, но где структура в целом фиксирована и хорошо подходит для варианта когда нужно быстро развернуть готовое решение. 
### А Kustomize - это как своего рода конструктор, где конечно есть база, но можем изменять/добавлять компоненты, не нужно учить новый язык шаблонов, работать с чистыми YAML-файлами, ну и при умении конечно легче контролировать каждый аспект конфигурации. 
### Если называть преимущества, то я бы наверное назвал основными это то что он проще в освоении (хотя для меня всё тяжко :) ), более гибкий и легче контролировать каждый аспект конфигурации, использует стандартные YAML-файлы Kubernetes, хорошо подходит для CI/CD, так как изменения более прозрачны (это я на форуме одном нашел), ну и меньше шансов "сломать" что-то, так как нет сложной логики шаблонов. 
### Примерно так собрался образ различий, что удалось увидеть, прочитать, понять. 



