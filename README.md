# run_single_instance_statefulset_for_mysql

這個 repo 主要參考 [Deploy a single instance Mysql use StatefulSet Task](https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/) 

## Prequest

已有 k8s 執行個體

在此將會使用 minikube

## 佈署目標

1 建立對應到本機硬碟的 PersistentVolume 與可用的 PersistentVolumeClaim 給 Mysql

2 給 Mysql 的 Secret 來存放 MYSQL_ROOT_PASSWORD

3 建立一個 headless Service 給 Mysql 來存取與一個 mysql 的 Deployment

## 建立 PersistentVolume 遇 PersistentVolumeClaim

建立一個 mysql-pv.yaml 檔案如下：

```yaml=
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

第1部份 建立一個 PersistentVolume

設定容量為 20Gi

掛載點為 /mnt/data

讀取模式是 ReadWriteOnce

第2部份 建立一個 PersistentVolumeClaim

名稱設定為 mysql-pv-claim

讀取模式是 ReadWriteOnce

設定需求容量為 20Gi

建立指令如下：

```shell=
kubectl apply -f mysql-pv.yaml
```
![](https://i.imgur.com/U832SEX.png)

查訊指令如下：

```shell=
kubectl describe pvc mysql-pv-claim
```
![](https://i.imgur.com/Qzyyynk.png)

## 給 Mysql 的 Secret 來存放 MYSQL_ROOT_PASSWORD

建立 mysql-secret.yaml

```yaml=
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  mysql-root-password: cGFzc3dvcmQ=
```

建立一個 Secret 名稱為 mysql-secret

設定型態為 key value Pair 且使用 Base64 編碼

設定值為 mysql-root-password: cGFzc3dvcmQ=

這邊 mysql-root-password 的值是使用以下指令產生

```shell=
echo -n 'password' | base64
```

建立指令如下

```shell=
kubectl apply -f mysql-secret.yaml 
```

![](https://i.imgur.com/sMilfbv.png)

## 建立一個 headless Service 給 Mysql 來存取與一個 mysql 的 Deployment

```yaml=
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-root-password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

第1部份 建立一個 headless Service

設定名稱為 mysql

設定對接的 port 3306

設定 ClusterIP 為 None 

設定 selector 為 app: mysql

上面這個設定讓 mysql 這個 headless Service 對應到 labels 為 app: mysql 的 Pod

第2部份 建立一個 Deployment

設定名稱為 mysql

設定 selector 對應的條件是找到 labels 為 app: mysql 的 Pod

重點的部份在於

1 template container 的環境參數 MYSQL_ROOT_PASSWORD 使用 mysql-secret 這個 Secret 內部的 mysql-root-password 值 

2 template container 的 containerPort 設立為 3306

3 volumes 從 mysql-pv-name 這個 Persistent-Volume-Claim 取得並設立名稱為 mysql-persistent-storage

4 template container volumeMount 把 mysql-persistent-storage 掛載到 /var/lib/mysql

建立指令如下：

```yaml=
kubectl apply -f mysql-deployment.yaml 
```

![](https://i.imgur.com/nfivEpV.png)

查看 deployment 指令

```shell=
kubectl describe deployment mysql
```

![](https://i.imgur.com/1PXd2TR.png)


## 驗證連接

因為是 headless Service 所以 ClusterIP 會變成 Pod IP

並且 Service 名稱在 Cluster 內也會解析成 Pod IP

所以目前可以在 Cluster 內透過 Service Name mysql 連線到 Pod

連線驗證指令如下

```shell=
kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -ppassword
```

![](https://i.imgur.com/4vibU1H.png)

## 刪除佈署

```shell=
kubectl delete deployment,svc mysql
kubectl delete pvc mysql-pv-claim
kubectl delete pv mysql-pv-volume
kubectl delete secret mysql-secret
```

## 後記

這邊很特別的是因為 PersistentVolume 是本機的 Disk

所以無法做 scale up

因為當 scale up 這類的 PersistentVolume 無法掛載的 Pod上

另外一點是這邊的 Deployment strategy 設定成 Recreate 

這樣一來在更新這個 Pod 的時候, 就不會採用 Rolling Update

而會先刪除舊的 Pod 再產生一個新的 Pod