---
layout: post
title: 2018-4-11-基于Kubernetes部署kong 0.13.0
date: 2018-4-11 17:4:5 +0386
categories: 环境配置
tags: API_Gateway Kubernetes
description: 在Kubernetes中部署kong的新手指南。
---

# （一） 一些准备工作

### 1. Kubernetes部署完成并且支持DNS解析；
### 2. 准备运行数据库（Postgres或者Cassandra）容器的必要的后端存储，这里我使用的heketi-glusterfs;
### 3. 特别说明，本文kong应用部署在default namespace中
# （二）Kong部署过程
## 1. 创建StorageClass，为运行数据库Postgres或者Cassandra做准备；

``` yaml?linenums
# cat glusterfs-storageclass.yaml    
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: glusterfs-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://heketi-sever:port"
  restuser: "username"
  restuserkey: "password"
```
使用glusterfs-storageclass.yaml在Kubernetes中创建StorageClass glusterfs-sc。
``` bash
kubectl create -f glusterfs-storageclass.yaml 
```
## 2. 运行并初始化数据库
官方给出了支持的两种数据库：Postgres和Cassandra，在此我选择了Cassandra。
###  1. 运行Cassandra数据库
Cassandra服务以及StatefulSet的描述文件如下：

``` YAML?linenums
# cat cassandra.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: cassandra
  name: cassandra
spec:
  clusterIP: None
  ports:
    - port: 9042
  selector:
    app: cassandra
---
apiVersion: "apps/v1beta1"
kind: StatefulSet
metadata:
  name: cassandra
spec:
  serviceName: cassandra
  replicas: 1
  template:
    metadata:
      labels:
        app: cassandra
    spec:
      containers:
      - name: cassandra
        # image: yujinyu/cassandra:v12
        image: cassandra
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 7000
          name: intra-node
        - containerPort: 7001
          name: tls-intra-node
        - containerPort: 7199
          name: jmx
        - containerPort: 9042
          name: cql
        env:
          - name: MAX_HEAP_SIZE
            value: 512M
          - name: HEAP_NEWSIZE
            value: 100M
          - name: CASSANDRA_SEEDS
            # 根据部署应用所在的namespace修改以下的值
            value: "cassandra-0.cassandra.default.svc.cluster.local"
          - name: CASSANDRA_CLUSTER_NAME
            value: "K8Demo"
          - name: CASSANDRA_DC
            value: "DC1-K8Demo"
          - name: CASSANDRA_RACK
            value: "Rack1-K8Demo"
          - name: CASSANDRA_AUTO_BOOTSTRAP
            value: "false"
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
        volumeMounts:
        - name: cassandra-data
          mountPath: /cassandra_data
  volumeClaimTemplates:
 1. metadata:
      name: cassandra-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: glusterfs-sc
      resources:
        requests:
          storage: 10Gi
```
使用以上描述文件部署Cassandra服务以及StatefulSet，
``` bash
kubectl create -f cassandra.yaml
```
###  2. 准备Cassandra数据库

``` yaml?linenums
# cat kong_migration_cassandra.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kong-migration
spec:
  template:
    metadata:
      name: kong-migration
    spec:
      containers:
      - name: kong-migration
        image: kong:0.13.0-centos
        env:
          - name: KONG_NGINX_DAEMON
            value: 'off'
          - name: KONG_DATABASE
            value: cassandra
          - name: KONG_CASSANDRA_CONTACT_POINTS
            value: cassandra
          - name: KONG_CASSANDRA_KEYSPACE
            value: kong
        command: [ "/bin/sh", "-c", "kong migrations up" ]
      restartPolicy: Never
```
运行kong-migration job准备并更新数据库，

``` bash
kubectl create -f kong_migration_cassandra.yaml
```
一旦job完成即在使用命令“kubectl get job kong-migration"显示内容中success下面显示为“1”，通过以下命令移除该job,

``` bash
kubectl delete -f kong_migration_cassandra.yaml
```
###  3.运行kong
kong服务以及Deploy描述文件如下，
``` yaml?linenums
# cat kong_cassandra.yaml
apiVersion: v1
kind: Service
metadata:
  name: kong-proxy
spec:
  type: LoadBalancer
  loadBalancerSourceRanges:
  - 0.0.0.0/0
  ports:
  - name: kong-proxy
    port: 8000
    targetPort: 8000
    protocol: TCP
  selector:
    app: kong

---
apiVersion: v1
kind: Service
metadata:
  name: kong-proxy-ssl
spec:
  type: LoadBalancer
  loadBalancerSourceRanges:
  - 0.0.0.0/0
  ports:
  - name: kong-proxy-ssl
    port: 8443
    targetPort: 8443
    protocol: TCP
  selector:
    app: kong

---
apiVersion: v1
kind: Service
metadata:
  name: kong-admin
spec:
  type: LoadBalancer
  loadBalancerSourceRanges:
  - 0.0.0.0/0
  ports:
  - name: kong-admin
    port: 8001
    targetPort: 8001
    protocol: TCP
  selector:
    app: kong

---
apiVersion: v1
kind: Service
metadata:
  name: kong-admin-ssl
spec:
  type: LoadBalancer
  loadBalancerSourceRanges:
  - 0.0.0.0/0
  ports:
  - name: kong-admin-ssl
    port: 8444
    targetPort: 8444
    protocol: TCP
  selector:
    app: kong

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kong-rc
spec:
  replicas: 2
  template:
    metadata:
      labels:
        name: kong-rc
        app: kong
    spec:
      containers:
      - name: kong
        image: kong:0.13.0-centos
        env:
          - name: KONG_ADMIN_LISTEN
            value: "0.0.0.0:8001, 0.0.0.0:8444 ssl"
          - name: KONG_DATABASE
            value: cassandra
          - name: KONG_CASSANDRA_CONTACT_POINTS
            value: cassandra
          - name: KONG_CASSANDRA_KEYSPACE
            value: kong
          - name: KONG_CASSANDRA_REPL_FACTOR
            value: "2"
          - name: KONG_PROXY_ACCESS_LOG
            value: "/dev/stdout"
          - name: KONG_ADMIN_ACCESS_LOG
            value: "/dev/stdout"
          - name: KONG_PROXY_ERROR_LOG
            value: "/dev/stderr"
          - name: KONG_ADMIN_ERROR_LOG
            value: "/dev/stderr"
        ports:
        - name: admin
          containerPort: 8001
          protocol: TCP
        - name: proxy
          containerPort: 8000
          protocol: TCP
        - name: proxy-ssl
          containerPort: 8443
          protocol: TCP
        - name: admin-ssl
          containerPort: 8444
          protocol: TCP
```
使用描述文件部署kong，

``` bash
kubectl create -f kong_cassandra.yaml
```

###  4.【可选，只适用于Cassandra】保证数据库的高可靠性
为保证Cassandra的高可靠性扩展Cassandra StatefulSet的副本至3个，

``` bash
kubectl scale --replicas=3 statefulset/cassandra
```

# （三） 验证并使用kong
## 1. 直接访问kong

``` bash
curl http://${kubernetes_master_ip}:$(kong_admin_nodeport} | python -m json.tool
```
如成功部署kong，运行以上命令不出错并有数据显示。
## 2. 运行Nginx实例测试
### 1. 部署测试应用Nginx

``` yaml?linenums
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
```
### 2. 在kong中添加api

``` bash
curl -i -X POST --url http://${kubernetes_master_ip}:${kong_admin_nodeport}/apis/ --data 'name=nginx-example-api'   --data 'hosts=nginx-example.com'   --data 'upstream_url=http://my-nginx'
```
说明：如果部署Nginx应用和Kong不在同一namespace，则需修改命令中“
upstream_url”的值，加入kong在defaul而Nginx在test命名空间，那么upstream_url则应修改为“upstream_url=http://my_nginx.test.svc.cluster.local"（cluster-domain默认为cluster.local）。

### 3. 使用api访问后端Nginx服务

``` bash
curl -i -X GET  --url http://${kubernetes_master_ip}:$(kong_proxy_nodeport}/  --header 'Host: nginx-example.com'
```
结果如图所示，
![enter description here](https://www.github.com/yujinyu/markdown/raw/master/images/2018-4-11-基于Kubernetes部署kong 0.13.0/clipboard.png)

# （四） 到此已成功运行kong。


----------

参考资料：
[1]. https://github.com/Kong/kong-dist-kubernetes
[2]. https://getkong.org/install/kubernetes/
[3]. https://kubernetes.io/docs/concepts/storage/storage-classes/
[4]. https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
[5]. https://github.com/IBM/Scalable-Cassandra-deployment-on-Kubernetes
[6]. https://getkong.org/docs/0.13.x/getting-started/configuring-a-service/