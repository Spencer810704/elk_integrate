
Table of contents
- [介紹](#介紹)
- [架構圖](#架構圖)
- [系統版本](#系統版本)
- [機器規格](#機器規格)
  - [Elasticsearch Cluster](#elasticsearch-cluster)
  - [Kafka Cluster](#kafka-cluster)
  - [Consumer Group](#consumer-group)
- [部署服務](#部署服務)
  - [事前準備](#事前準備)
  - [Elasticsearch Cluster](#elasticsearch-cluster-1)
    - [設置LVM](#設置lvm)
    - [安裝](#安裝)
    - [修改配置](#修改配置)
    - [設定開機啟動及啟動服務](#設定開機啟動及啟動服務)
    - [驗證](#驗證)
  - [Kibana](#kibana)
    - [安裝](#安裝-1)
    - [修改配置](#修改配置-1)
    - [啟動服務及開機啟動](#啟動服務及開機啟動)
  - [Kafka Cluster](#kafka-cluster-1)
    - [設置LVM](#設置lvm-1)
    - [安裝](#安裝-2)
    - [配置](#配置)
    - [服務啟動及開機啟動](#服務啟動及開機啟動)
    - [驗證](#驗證-1)
    - [常用指令](#常用指令)
  - [Kafka Eagle (Kafka Web介面)](#kafka-eagle-kafka-web介面)
  - [Kafka Producer](#kafka-producer)
    - [配置](#配置-1)
  - [Kafka Consumer](#kafka-consumer)
    - [安裝](#安裝-3)
    - [配置](#配置-2)
    - [啟動服務＆開機啟動](#啟動服務開機啟動)
    - [驗證](#驗證-2)
- [驗證](#驗證-3)
- [Reference](#reference)

# 介紹
紀錄 ElasticSearch Cluster + Kafka  + Logstash 整合過程

<br>

# 架構圖

![architecture](img/architecture.png)

<br>

# 系統版本

- Java：`openjdk15.0.2_7`
- Kibana  : `8.4.2`
- Elasticsearch ：`8.4.2`
- Kafka ：`kafka_2.13-3.2.0`

<br>

# 機器規格

<br>

## Elasticsearch Cluster

| hostname | IP | CPU | Memory | Disk (LVM) | 備註 |
| --- | --- | --- | --- | --- | --- |
| fet-es-master01.pf | 10.64.5.101 | 2 | 32 | 30 |  |
| fet-es-master02.pf | 10.64.5.102 | 2 | 32 | 30 |  |
| fet-es-master03.pf | 10.64.5.103 | 2 | 32 | 30 |  |
| fet-es-coordinating01.pf | 10.64.5.104 | 2 | 32 | 30 | Kibana也安裝在此VM |
| fet-es-coordinating02.pf | 10.64.5.105 | 2 | 32 | 30 | Kibana也安裝在此VM |
| fet-es-data01.pf | 10.64.5.106 | 8 | 64 | 200 |  |
| fet-es-data02.pf | 10.64.5.107 | 8 | 64 | 200 |  |
| fet-es-data03.pf | 10.64.5.108 | 8 | 64 | 200 |  |
| fet-es-data04.pf | 10.64.5.109 | 8 | 64 | 200 |  |
| fet-es-data05.pf | 10.64.5.110 | 8 | 64 | 200 |  |
| fet-es-data06.pf | 10.64.5.111 | 8 | 64 | 200 |  |

<br>

## Kafka Cluster

Kafka 機器磁碟空間需要較大  , 因日誌會儲存在Kafka內等消費者進行消化傳送至Elasticsearch 

日誌的刪除策略會設置 保留最近48hours

| hostname | IP | CPU | Memory | Disk (LVM) | 備註 |
| --- | --- | --- | --- | --- | --- |
| fet-es-kfk01.pf | 10.64.5.115 | 4 | 8 | 200 | kafka + zookeeper |
| fet-es-kfk02.pf | 10.64.5.116 | 4 | 8 | 200 | kafka + zookeeper |
| fet-es-kfk03.pf | 10.64.5.117 | 4 | 8 | 200 | kafka + zookeeper |

<br>

## Consumer Group

| hostname | IP | CPU | Memory | Disk (LVM) | 備註 |
| --- | --- | --- | --- | --- | --- |
| fet-es-kfk-consumer01.pf | 10.64.5.121 | 4 | 8 | 30 | logstash |
| fet-es-kfk-consumer02.pf | 10.64.5.122 | 4 | 8 | 30 | logstash |
| fet-es-kfk-consumer03.pf | 10.64.5.123 | 4 | 8 | 30 | logstash |

<br>

# 部署服務

<br>

## 事前準備

CoreDNS 設定

```bash
# Master Node
/coredns/nexiosoft/pf/fet-es-master01 {"host": "10.64.5.101", "ttl": 60}
/coredns/nexiosoft/pf/fet-es-master02 {"host": "10.64.5.102", "ttl": 60}
/coredns/nexiosoft/pf/fet-es-master03 {"host": "10.64.5.103", "ttl": 60}

# Coordinating Node
/coredns/nexiosoft/pf/fet-es-coordinating01 {"host": "10.64.5.104", "ttl": 60}
/coredns/nexiosoft/pf/fet-es-coordinating02 {"host": "10.64.5.105", "ttl": 60}

# Data Node
/coredns/nexiosoft/pf/fet-es-data01 {"host": "10.64.5.106", "ttl": 60}
/coredns/nexiosoft/pf/fet-es-data02 {"host": "10.64.5.107", "ttl": 60}
/coredns/nexiosoft/pf/fet-es-data03 {"host": "10.64.5.108", "ttl": 60}
/coredns/nexiosoft/pf/fet-es-data04 {"host": "10.64.5.109", "ttl": 60}
/coredns/nexiosoft/pf/fet-es-data05 {"host": "10.64.5.110", "ttl": 60}
/coredns/nexiosoft/pf/fet-es-data06 {"host": "10.64.5.111", "ttl": 60}

# Elasticsearch Kafka
/coredns/nexiosoft/pf/fet-es-kfk01 {"host": "10.64.5.115", "ttl": 60}
/coredns/nexiosoft/pf/fet-es-kfk02 {"host": "10.64.5.116", "ttl": 60}
/coredns/nexiosoft/pf/fet-es-kfk03 {"host": "10.64.5.117", "ttl": 60}

# Kafka消費者(Logstash)
/coredns/nexiosoft/pf/fet-es-kfk-consumer01 {"host": "10.64.5.121", "ttl": 60}
/coredns/nexiosoft/pf/fet-es-kfk-consumer02 {"host": "10.64.5.122", "ttl": 60}
/coredns/nexiosoft/pf/fet-es-kfk-consumer03 {"host": "10.64.5.123", "ttl": 60}
```

<br>

## Elasticsearch Cluster

[https://learnku.com/articles/40718](https://learnku.com/articles/40718)

<br>

### 設置LVM

請依照實際狀況修改裝置位置(/dev/sd<x>)

```bash

# 建立 PV
root@ubuntu:~# pvcreate /dev/sdb
# 建立（擴充) VG
root@ubuntu:~# vgcreate data /dev/sdb
# 建立 LV
root@ubuntu:~# lvcreate  -l +100%FREE -n data data
# 建立一個 ext4 檔案系統
root@ubuntu:~# mkfs.ext4 /dev/data/data
# 暫時掛載 
root@ubuntu:~# mkdir /data
root@ubuntu:~# mount /dev/data/data /data/
# 設置永久掛載
root@ubuntu:~# echo "/dev/mapper/data-data     /data        ext4    defaults 0 1" >> /etc/fstab

```

<br>

### 安裝

在所有 Elasticsearch 節點中下載檔案以及安裝

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.4.2-amd64.deb
dpkg -i elasticsearch-8.4.2-amd64.deb
```

<br>

### 修改配置

**設定 `jvm.options` 配置** (在 /etc/elasticsearch/jvm.options.d/ 下新增 jvm.options)

(依照實際節點配置)

```bash
# Data Node
-Xms32g
-Xmx32g

# Master Node
-Xms16g
-Xmx16g

# Coordinating Node
-Xms16g
-Xmx16g

```

**設定 `sysctl.conf`** 

```bash
# =========================== Master Node =========================== 
kernel.shmmax=30250893312
kernel.shmall=7385472
kernel.msgmax=65536
kernel.msgmnb=65536
vm.swappiness=1
vm.max_map_count=262144
net.core.rmem_max=33554432
net.core.wmem_max=33554432
net.core.somaxconn=65535
net.ipv4.tcp_tw_reuse=0
net.ipv4.tcp_fin_timeout=5
net.ipv4.tcp_keepalive_time=300
net.core.netdev_max_backlog=30000
net.ipv4.ip_forward=0
net.ipv4.icmp_echo_ignore_broadcasts=1
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
fs.file-max=1000000

# =========================== Data Node =========================== 
kernel.shmmax=60646175539
kernel.shmall=14806195
kernel.msgmax=65536
kernel.msgmnb=65536
vm.swappiness=1
vm.max_map_count=262144
net.core.rmem_max=33554432
net.core.wmem_max=33554432
net.core.somaxconn=65535
net.ipv4.tcp_tw_reuse=0
net.ipv4.tcp_fin_timeout=5
net.ipv4.tcp_keepalive_time=300
net.core.netdev_max_backlog=30000
net.ipv4.ip_forward=0
net.ipv4.icmp_echo_ignore_broadcasts=1
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
fs.file-max=1000000

# =========================== Coordinating Node =========================== 
kernel.shmmax=30250893312
kernel.shmall=7385472
kernel.msgmax=65536
kernel.msgmnb=65536
vm.swappiness=1
vm.max_map_count=262144
net.core.rmem_max=33554432
net.core.wmem_max=33554432
net.core.somaxconn=65535
net.ipv4.tcp_tw_reuse=0
net.ipv4.tcp_fin_timeout=5
net.ipv4.tcp_keepalive_time=300
net.core.netdev_max_backlog=30000
net.ipv4.ip_forward=0
net.ipv4.icmp_echo_ignore_broadcasts=1
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
fs.file-max=1000000
```

上面參數只有針對目前最上面規劃的系統資源配置計算  ( 不太了解內核參數實際作用 , 是抓原本ansible專案內的優化playbook進行修改 ) , 附上Ansible playbook

```bash
---
- name: "update elasticsearch config"
  hosts: "fet_elasticsearch"
  tasks:
    # update sysctl | elasticsearch_conf
    - name: update sysctl | elasticsearch_conf
      sysctl:
        name: "{{ item.name }}"
        value: "{{ item.value }}"
        sysctl_set: "{{ item.sysctl_set }}"
        sysctl_file: "{{ item.sysctl_file }}"
        state: "{{ item.state }}"
        reload: "{{ item.reload }}"
      with_items:
        # memory
        - name: kernel.shmmax
          value: "{{ (ansible_memtotal_mb * 1024 * 1024 * 0.9) | int }}"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        - name: kernel.shmall
          value: "{{ (ansible_memtotal_mb * 1024 * 1024 * 0.9 / 4096) | int }}"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        - name: kernel.msgmax
          value: "65536"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        - name: kernel.msgmnb
          value: "65536"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        - name: vm.swappiness
          value: "1"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        - name: vm.max_map_count
          value: "262144"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        # netowrk
        - name: net.core.rmem_max
          value: "33554432"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        - name: net.core.wmem_max
          value: "33554432"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        - name: net.core.somaxconn
          value: "65535"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        - name: net.ipv4.tcp_tw_reuse
          value: "0"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        - name: net.ipv4.tcp_fin_timeout
          value: "5"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        - name: net.ipv4.tcp_keepalive_time
          value: "300"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        - name: net.core.netdev_max_backlog
          value: "30000"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        - name: net.ipv4.ip_forward
          value: "0"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        - name: net.ipv4.icmp_echo_ignore_broadcasts
          value: "1"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        # disable ipv6
        - name: net.ipv6.conf.all.disable_ipv6
          value: "1"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        - name: net.ipv6.conf.default.disable_ipv6
          value: "1"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
        # filesystem
        - name: fs.file-max
          value: "1000000"
          sysctl_set: yes
          sysctl_file: /etc/sysctl.d/elasticsearch.conf
          state: present
          reload: yes
```

**設定 `Systemd  override`**

```bash
mkdir -p /etc/systemd/system/elasticsearch.service.d
cat << EOF > /etc/systemd/system/elasticsearch.service.d/override.conf
[Service]
LimitNOFILE=200000
LimitMEMLOCK=infinity
EOF
```

**修改目錄權限**

```bash
chown -R elasticsearch:elasticsearch /data
```

### 設定開機啟動及啟動服務

```bash
systemctl daemon-reload
systemctl enable elasticsearch.service --now
```

設定 elasticsearch.yml

```bash
# =========================== Master Node =========================== 
############################################
#
# This file is managed by Ansible.
#
# Please do not edit this file manually!!
#
# date: 2022-11-28 14:36:02
#
############################################

# ---------------------------------- Cluster -----------------------------------
cluster.name: fet_es_cluster

# ------------------------------------ Node ------------------------------------
node.name: fet-es-master01.pf

# ----------------------------------- Paths ------------------------------------
path.data: /data/elasticsearch/data
path.logs: /data/elasticsearch/logs

# ----------------------------------- Memory -----------------------------------
bootstrap.memory_lock: true

# ---------------------------------- Network -----------------------------------
network.host: 0.0.0.0
http.port: 9200

# --------------------------------- Discovery ----------------------------------
discovery.seed_hosts: ['fet-es-master01.pf', 'fet-es-master02.pf', 'fet-es-master03.pf', 'fet-es-coordinating01.pf', 'fet-es-coordinating02.pf', 'fet-es-data01.pf', 'fet-es-data02.pf', 'fet-es-data03.pf', 'fet-es-data04.pf', 'fet-es-data05.pf', 'fet-es-data06.pf']
cluster.initial_master_nodes: ['fet-es-master01.pf', 'fet-es-master02.pf', 'fet-es-master03.pf']

# ---------------------------------- Various -----------------------------------
action.auto_create_index: "+*"

node.roles: [master]

# ---------------------------------- Security -----------------------------------
# Enable security features
xpack.security.enabled: false
xpack.security.enrollment.enabled: false

xpack.security.transport.ssl.enabled: false
xpack.security.http.ssl.enabled: false

## Enable encryption for HTTP API client connections, such as Kibana, Logstash, and Agents
#xpack.security.http.ssl:
#  enabled: true
#  keystore.path: certs/http.p12
#
## Enable encryption and mutual authentication between cluster nodes
#xpack.security.transport.ssl:
#  enabled: true
#  verification_mode: certificate
#  keystore.path: certs/transport.p12
#  truststore.path: certs/transport.p12

# =========================== Data Node ===============================

############################################
#
# This file is managed by Ansible.
#
# Please do not edit this file manually!!
#
# date: 2022-11-28 14:36:02
#
############################################

# ---------------------------------- Cluster -----------------------------------
cluster.name: fet_es_cluster

# ------------------------------------ Node ------------------------------------
node.name: fet-es-data01.pf

# ----------------------------------- Paths ------------------------------------
path.data: /data/elasticsearch/data
path.logs: /data/elasticsearch/logs

# ----------------------------------- Memory -----------------------------------
bootstrap.memory_lock: true

# ---------------------------------- Network -----------------------------------
network.host: 0.0.0.0
http.port: 9200

# --------------------------------- Discovery ----------------------------------
discovery.seed_hosts: ['fet-es-master01.pf', 'fet-es-master02.pf', 'fet-es-master03.pf', 'fet-es-coordinating01.pf', 'fet-es-coordinating02.pf', 'fet-es-data01.pf', 'fet-es-data02.pf', 'fet-es-data03.pf', 'fet-es-data04.pf', 'fet-es-data05.pf', 'fet-es-data06.pf']
cluster.initial_master_nodes: ['fet-es-master01.pf', 'fet-es-master02.pf', 'fet-es-master03.pf']

# ---------------------------------- Various -----------------------------------
action.auto_create_index: "+*"

node.roles: [data]

# ---------------------------------- Security -----------------------------------
# Enable security features
xpack.security.enabled: false
xpack.security.enrollment.enabled: false

xpack.security.transport.ssl.enabled: false
xpack.security.http.ssl.enabled: false

## Enable encryption for HTTP API client connections, such as Kibana, Logstash, and Agents
#xpack.security.http.ssl:
#  enabled: true
#  keystore.path: certs/http.p12
#
## Enable encryption and mutual authentication between cluster nodes
#xpack.security.transport.ssl:
#  enabled: true
#  verification_mode: certificate
#  keystore.path: certs/transport.p12
#  truststore.path: certs/transport.p12

# =========================== Coordinating Node =========================== 

############################################
#
# This file is managed by Ansible.
#
# Please do not edit this file manually!!
#
# date: 2022-11-28 14:36:02
#
############################################

# ---------------------------------- Cluster -----------------------------------
cluster.name: fet_es_cluster

# ------------------------------------ Node ------------------------------------
node.name: fet-es-coordinating02.pf

# ----------------------------------- Paths ------------------------------------
path.data: /data/elasticsearch/data
path.logs: /data/elasticsearch/logs

# ----------------------------------- Memory -----------------------------------
bootstrap.memory_lock: true

# ---------------------------------- Network -----------------------------------
network.host: 0.0.0.0
http.port: 9200

# --------------------------------- Discovery ----------------------------------
discovery.seed_hosts: ['fet-es-master01.pf', 'fet-es-master02.pf', 'fet-es-master03.pf', 'fet-es-coordinating01.pf', 'fet-es-coordinating02.pf', 'fet-es-data01.pf', 'fet-es-data02.pf', 'fet-es-data03.pf', 'fet-es-data04.pf', 'fet-es-data05.pf', 'fet-es-data06.pf']
cluster.initial_master_nodes: ['fet-es-master01.pf', 'fet-es-master02.pf', 'fet-es-master03.pf']

# ---------------------------------- Various -----------------------------------
action.auto_create_index: "+*"

node.roles: []

# ---------------------------------- Security -----------------------------------
# Enable security features
xpack.security.enabled: false
xpack.security.enrollment.enabled: false

xpack.security.transport.ssl.enabled: false
xpack.security.http.ssl.enabled: false

## Enable encryption for HTTP API client connections, such as Kibana, Logstash, and Agents
#xpack.security.http.ssl:
#  enabled: true
#  keystore.path: certs/http.p12
#
## Enable encryption and mutual authentication between cluster nodes
#xpack.security.transport.ssl:
#  enabled: true
#  verification_mode: certificate
#  keystore.path: certs/transport.p12
#  truststore.path: certs/transport.p12

```

<br>

### 驗證

使用 elasticvue 連線 coordinating , 如果能看到節點資訊表示cluster已經建立完成

<br>

## Kibana

<br>

### 安裝

```bash
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.4.2-amd64.deb
dpkg -i kibana-8.4.2-amd64.deb
```

<br>

### 修改配置

```bash
server.port: 5601
# 02 一樣改成 fet-es-coordinating02.pf
server.host: "fet-es-coordinating01.pf"
elasticsearch.hosts: ["http://localhost:9200"]
elasticsearch.requestTimeout: 30000
logging:
  appenders:
    file:
      type: file
      fileName: /var/log/kibana/kibana.log
      layout:
        type: json
  root:
    appenders:
      - default
      - file
pid.file: /run/kibana/kibana.pid
```

<br>

### 啟動服務及開機啟動

```bash
systemctl enable kibana.service --now
```

<br>

## Kafka Cluster

參考文章: [https://acloudguru.com/hands-on-labs/installing-a-kafka-cluster-and-creating-a-topic](https://acloudguru.com/hands-on-labs/installing-a-kafka-cluster-and-creating-a-topic)

<br>

### 設置LVM

```bash
# 建立 PV
root@ubuntu:~# pvcreate /dev/sdb
# 建立（擴充) VG
root@ubuntu:~# vgcreate data /dev/sdb
# 建立 LV
root@ubuntu:~# lvcreate  -l +100%FREE -n data data
# 建立一個 ext4 檔案系統
root@ubuntu:~# mkfs.ext4 /dev/data/data
# 暫時掛載 
root@ubuntu:~# mkdir /data
root@ubuntu:~# mount /dev/data/data /data/
# 設置永久掛載
root@ubuntu:~# echo "/dev/mapper/data-data     /data        ext4    defaults 0 1" >> /etc/fstab

```

<br>

### 安裝

```bash
# Kafka 內有包含 Zookeeper , 不需另外下載
wget https://archive.apache.org/dist/kafka/3.2.0/kafka_2.13-3.2.0.tgz

mv /opt/
tar -xvf kafka_2.13-3.2.0.tgz
mv kafka_2.13-3.2.0 kafka
mv kafka /opt
```

<br>

### 配置

關閉SWAP (全部節點進行配置)

```bash
sed -i '/ swap / s/^/#/' /etc/fstab
```

建立Kafka Data目錄 (全部節點進行配置)

```bash
mkdir -p /data/kafka
mkdir -p /data/zookeeper
```

設定zookeeper id (依照各節點進行配置)

```bash
# 在Kafka對應機器中給予相應ID 
# kafka01就給1
echo "1" > /data/zookeeper/myid

# kafka02就給2
echo "2" > /data/zookeeper/myid

# kafka03就給3
echo "3" > /data/zookeeper/myid
```

修改 Kafka 配置 (依照各節點進行配置)

```bash
# kafka01
# 先刪除原本的預設配置
rm /opt/kafka/config/server.properties
cat << EOF > /opt/kafka/config/server.properties
# broker 的 ID，每一個 broker 的 ID 必須是唯一的、不能和其他重複。
broker.id=1

# kafka 節點啟動後，會向 zookeeper 註冊自己，同時告訴 zookeeper 自身的通信地址，
# 這個地址就是配置文件中的 advertised.listeners，
# 如果沒有配置 advertised.listeners，就會使用listeners。
# 同時從 zookeeper 中獲取兄弟節點的這個地址，以便與兄弟節點通信。
# 即 kafka 節點是從 zookeeper 獲取的其他節點的通信地址。
# 可以從 zookeeper shell 中下 get /brokers/ids/<id>
advertised.listeners=PLAINTEXT://fet-es-kfk01.pf:9092

# delete topics
delete.topic.enable=true

# Log儲存路徑
log.dirs=/data/kafka

# 建立topic的默認分區數
num.partitions=8

# 預設副本數量
default.replication.factor=3

# to protect yourself against broker failure
min.insync.replicas=2

# 多少小時後刪除數據
log.retention.hours=48

# 單一 Segment 的最大大小
# log.segment.bytes=1073741824

# 檢查週期，查看它們是否達到了刪除策略的設置 （log.retention.hours或log.retention.bytes）
log.retention.check.interval.ms=300000

# 集群中所有 zookeeper instances 
zookeeper.connect=fet-es-kfk01.pf:2181,fet-es-kfk02.pf:2181,fet-es-kfk03.pf:2181/kafka

# zookeeper 連線超時時間
zookeeper.connection.timeout.ms=6000

# 關閉自動建立topic , 這個參數會影響topic的管理與維護, 由運維人員建立
auto.create.topics.enable=false
EOF

# kafka02
# 先刪除原本的預設配置
rm /opt/kafka/config/server.properties

cat << EOF > /opt/kafka/config/server.properties
broker.id=2

# kafka 節點啟動後，會向 zookeeper 註冊自己，同時告訴 zookeeper 自身的通信地址，
# 這個地址就是配置文件中的 advertised.listeners，
# 如果沒有配置 advertised.listeners，就會使用listeners。
# 同時從 zookeeper 中獲取兄弟節點的這個地址，以便與兄弟節點通信。
# 即 kafka 節點是從 zookeeper 獲取的其他節點的通信地址。
advertised.listeners=PLAINTEXT://fet-es-kfk02.pf:9092

# delete topics
delete.topic.enable=true

# Log儲存路徑
log.dirs=/data/kafka

# 建立topic的默認分區數
num.partitions=8

# 預設副本數量
default.replication.factor=3

# to protect yourself against broker failure
min.insync.replicas=2

# 多少小時後刪除數據
log.retention.hours=48

# 單一 Segment 的最大大小
# log.segment.bytes=1073741824

# 檢查週期，查看它們是否達到了刪除策略的設置 （log.retention.hours或log.retention.bytes）
log.retention.check.interval.ms=300000

# 集群中所有 zookeeper instances 
zookeeper.connect=fet-es-kfk01.pf:2181,fet-es-kfk02.pf:2181,fet-es-kfk03.pf:2181/kafka

# zookeeper 連線超時時間
zookeeper.connection.timeout.ms=6000

# 關閉自動建立topic , 這個參數會影響topic的管理與維護, 由運維人員建立
auto.create.topics.enable=false
EOF

# kafka03
# 先刪除原本的預設配置
rm /opt/kafka/config/server.properties

cat << EOF > /opt/kafka/config/server.properties
broker.id=3

# kafka 節點啟動後，會向 zookeeper 註冊自己，同時告訴 zookeeper 自身的通信地址，
# 這個地址就是配置文件中的 advertised.listeners，
# 如果沒有配置 advertised.listeners，就會使用listeners。
# 同時從 zookeeper 中獲取兄弟節點的這個地址，以便與兄弟節點通信。
# 即 kafka 節點是從 zookeeper 獲取的其他節點的通信地址。
advertised.listeners=PLAINTEXT://fet-es-kfk03.pf:9092

# delete topics
delete.topic.enable=true

# Log儲存路徑
log.dirs=/data/kafka

# 建立topic的默認分區數
num.partitions=8

# 預設副本數量
default.replication.factor=3

# to protect yourself against broker failure
min.insync.replicas=2

# 多少小時後刪除數據
log.retention.hours=48

# 單一 Segment 的最大大小
# log.segment.bytes=1073741824

# 檢查週期，查看它們是否達到了刪除策略的設置 （log.retention.hours或log.retention.bytes）
log.retention.check.interval.ms=300000

# 集群中所有 zookeeper instances 
zookeeper.connect=fet-es-kfk01.pf:2181,fet-es-kfk02.pf:2181,fet-es-kfk03.pf:2181/kafka

# zookeeper 連線超時時間
zookeeper.connection.timeout.ms=6000

# 關閉自動建立topic , 這個參數會影響topic的管理與維護, 由運維人員建立
auto.create.topics.enable=false
EOF
```

修改Zookeeper配置 (三個節點相同)

```bash
rm /opt/kafka/config/zookeeper.properties

cat << EOF > /opt/kafka/config/zookeeper.properties
# the directory where the snapshot is stored.
dataDir=/data/zookeeper
# the port at which the clients will connect
clientPort=2181
# setting number of connections to unlimited
maxClientCnxns=0
# keeps a heartbeat of zookeeper in milliseconds
tickTime=2000
# time for initial synchronization
initLimit=10
# how many ticks can pass before timeout
syncLimit=5
# define servers ip and internal ports to zookeeper
server.1=fet-es-kfk01.pf:2888:3888
server.2=fet-es-kfk02.pf:2888:3888
server.3=fet-es-kfk03.pf:2888:3888
EOF
```

設定Zookeeper & Kafka Systemd Config ( 三個節點都操作 )

```bash
touch /etc/systemd/system/zookeeper.service

cat << EOF > /etc/systemd/system/zookeeper.service
[Unit]
Description=Apache Zookeeper server
Documentation=http://zookeeper.apache.org
Requires=network.target remote-fs.target
After=network.target remote-fs.target

[Service]
Type=simple
Environment="JAVA_HOME=/usr/java/openjdk15"
ExecStart=/opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
ExecStop=/opt/kafka/bin/zookeeper-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
EOF

cat << EOF > /etc/systemd/system/kafka.service
[Unit]
Description=Apache Kafka Server
Documentation=http://kafka.apache.org/documentation.html
Requires=zookeeper.service

[Service]
Type=simple
# 設定JMX讓Kafka-Eagle抓取相關資訊
Environment="JMX_PORT=9989"
Environment="JAVA_HOME=/usr/java/openjdk15"
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh

[Install]
WantedBy=multi-user.target
EOF
```

<br>

### 服務啟動及開機啟動

```bash
systemctl enable zookeeper --now
systemctl enable kafka --now
```

<br>

### 驗證

開兩個 SSH session 一個做生產消息 , 另一個Session觀察是否可以顯示生產者發所的訊息 

```bash
# 可以在~/.bashrc中加入以下Java路徑
export JAVA_HOME=/usr/java/openjdk15

# 建立Topic
/opt/kafka/bin/kafka-topics.sh --bootstrap-server fet-es-kfk01.pf:9092,fet-es-kfk02.pf:9092,fet-es-kfk03.pf:9092 --topic test --create

# 生產消息 
/opt/kafka/bin/kafka-console-producer.sh --bootstrap-server fet-es-kfk01.pf:9092,fet-es-kfk02.pf:9092,fet-es-kfk03.pf:9092 --topic test

# 消費消息
/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server fet-es-kfk01.pf:9092,fet-es-kfk02.pf:9092,fet-es-kfk03.pf:9092 --topic test
```

<br>

### 常用指令

```bash
# 建立Topic
# replication-factor: 副本數量
# partition: 分區數量
/opt/kafka/bin/kafka-topics.sh --create --bootstrap-server fet-es-kfk01.pf:9092,fet-es-kfk02.pf:9092,fet-es-kfk03.pf:9092 --replication-factor 1 --partitions 1 --topic mylab

# 列出當前topic有哪些
/opt/kafka/bin/kafka-topics.sh  --list --bootstrap-server fet-es-kfk01.pf:9092,fet-es-kfk02.pf:9092,fet-es-kfk03.pf:9092

# 建立生產者 , 產生message 
/opt/kafka/bin/kafka-console-producer.sh --bootstrap-server fet-es-kfk01.pf:9092,fet-es-kfk02.pf:9092,fet-es-kfk03.pf:9092 --topic mylab

# 建立消費者 , 獲取message
/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server fet-es-kfk01.pf:9092,fet-es-kfk02.pf:9092,fet-es-kfk03.pf:9092 --topic mylab --consumer-property group.id=mylabgroup

# 查看當前主題下有哪些消費組
/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server fet-es-kfk01.pf:9092,fet-es-kfk02.pf:9092,fet-es-kfk03.pf:9092 --list

# 查看消費組中的具體信息：比如當前偏移量(CURRENT-OFFSET)、最後一條消息的偏移量(LOG-END-OFFSET)、堆積的消息數量(LAG)
/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server fet-es-kfk01.pf:9092,fet-es-kfk02.pf:9092,fet-es-kfk03.pf:9092  --describe --group <group名稱>
```

<br>

## Kafka Eagle (Kafka Web介面)

Kafka eagle為Kafka監控及可視化工具 , 但測試狀況有許多問題 , 目前優先級不高 ,  待未來再研究

<br>

## Kafka Producer

此處使用 td-agent 服務作為 Kafka 的 Producer 

<br>

### 配置

```bash
####
## Source descriptions:
##

<source>
  @type tail
  @label @nginx-access
  path /var/log/nginx/access-*.log
  exclude_path ["/var/log/nginx/access-dwjt.log"]
  path_key log_path
  pos_file /var/log/td-agent/nginx-access.pos
  tag nginx-access
  <parse>
    @type regexp
    expression /^(?<remote_addr>\d.*) - (?<remote_user>[^ ]*) (?<http_host>[^ ]*) \[(?<log_time>[^\]]*)\] (?<request_time>[^ ]*) (?<request_length>[^ ]*) (?<method>\S+) (?<request>[^\"]*?) (?<protocol>[^ ]*) "(?<status>[^ ]*)" (?<body_bytes_sent>[^ ]*) "(?<http_referer>[^\"]*)" "(?<http_user_agent>[^\"]*)" "(?<http_x_forwarded_for>\S.*)" "(?<gzip_ratio>[^ ]*)" "(?<connection>[^ ]*)" "(?<connection_requests>[^ ]*)" (?<upstream_addr>\S.*) \[(?<upstream_response_time>[^\]]*)\] (?<upstream_cache_status>[^ ]*) (?<upstream_http_cache_control>[^ ]*) (?<upstream_http_expires>[^ ]*) (?<http_content_type>[^ ]*) (?<other>.*)$/
    types request_time:float, upstream_response_time:float, body_bytes_sent:integer, request_length:integer
    time_key log_time
    time_format "%d/%b/%Y:%H:%M:%S %z"
    keep_time_key true
  </parse>
</source>

####
## Filter descriptions:
##

## Filters go here.

####
## Match descriptions:
##

<label @nginx-access>
  <filter **>
    @type record_transformer
    enable_ruby true
    auto_typecast true
    <record>
      hostname "#{Socket.gethostname}"
    </record>
  </filter>

  <filter **>
  @type elasticsearch_genid
  hash_id_key _hash
  </filter>

  #<match **>
  #  @type stdout
  #</match>

  <match **>
    @type kafka2

    # list of seed brokers
    brokers 10.64.5.115:9092,10.64.5.116:9092,10.64.5.117:9092
    use_event_time false

    # buffer settings
    <buffer topic>
      @type file
      path /var/log/td-agent/buffer/td
      flush_interval 3s
    </buffer>

    # data type settings
    <format>
      @type json
    </format>

    # topic settings
    topic_key topic
    default_topic nginx-log-messages

    # producer settings
    required_acks -1
    compression_codec gzip
  </match>

</label>
```

<br>

## Kafka Consumer

此處使用 `Logstash` 服務作為 Kafka 的 Consumer

<br>

### 安裝

```bash
wget https://artifacts.elastic.co/downloads/logstash/logstash-8.4.2-amd64.deb
dpkg -i logstash-8.4.2-amd64.deb
```

<br>

### 配置

調整 JVM 記憶體用量

```bash

vim /etc/logstash/jvm.options
-Xms4g
-Xmx4g
```

建立 pipeline (在 /etc/logstash/conf.d/ 下建立 kafka.conf )

```bash
input{
    kafka{
        bootstrap_servers=>"es-kfk01.pf:9092,es-kfk02.pf:9092,es-kfk03.pf:9092"
        topics=>["nginx-access-log-messages"]
        group_id=>"nginx_access_log_group"
        type => "nginx-access"
        decorate_events => true
    }
}

filter
{
    if [type]=="nginx-access"{
        json {
            source => "message"
        }

        date {
            match => [ "log_time", "dd/MMM/yyyy:HH:mm:ss Z" ]
            target => "@timestamp"
            timezone => "Asia/Taipei"
        }
        mutate {
            remove_field => ["_hash", "@version", "original", "message"]
            add_field => {
                document_id => "%{[@metadata][kafka][topic]}-%{[@metadata][kafka][partition]}-%{[@metadata][kafka][offset]}"
            }
        }
    }
}

output{
    if [type]=="nginx-access"{
        elasticsearch{
            hosts=>["es-coordinating01.pf:9200", "es-coordinating02.pf:9200"]
            index => "nginx-mps-access-%{+YYYY.MM.dd}"
        }
				# 進行debug使用 , 預設不開起 ,否則太多日誌塞到 /var/log/syslog 內 , 塞滿硬碟空間
		    # stdout {
		    #     codec => rubydebug
		    # }
    }
}
```

<br>

### 啟動服務＆開機啟動

```bash
systemctl enable logstash.service --now
```

<br>

### 驗證

```bash
# 使用 journalctl 觀察實際的輸出 (因為我們上面配置有設定 stdout , 所以可以看到Kafka抓到的內容)
journalctl -u logstash.service -f
```

![verification](img/verification.png)

<br>

# 驗證

Elasticsearch cluster 、 Kafka cluster 及負責消費的logstash 都架設完畢後 , 最後我們可以在 kafka的機器中使用以下指令進行壓力測試

```bash
/opt/kafka/bin/kafka-producer-perf-test.sh --topic nginx-log-messages --num-records 40000000 --record-size 1024 --throughput 500000 --producer-props bootstrap.servers=fet-es-kfk01.pf:9092,fet-es-kfk02.pf:9092,fet-es-kfk03.pf:9092
```

參數用途如下

| 參數名 | 含義 |
| --- | --- |
| -h, --help | 顯示使用幫助並退出 |
| --topic | 指定生產的message要發往哪一個topic |
| --num-records | 指定生產的message總數 |
| --payload-delimeter | 如果通過--payload-file 指定了從文件中獲取消息內容，那麼這個參數的意義是指定文件的消息分隔符，默認值為\n，即文件的每一行視為一條消息；如果未指定--payload-file 則此參數不生效 |
| --throughput | 限制每秒發送的最大的消息數，設為-1 表示不限制 |
| --producer-props | 直接指定Producer 配置，格式為NAME=VALUE，例如bootstrap.server=127.0.0.1:9092，通過此種方式指定的配置優先級高於--producer.config |
| --producer-config | 指定Producer 的配置文件，格式參照官方的config/producer.properties |
| --print-metrics | 在測試結束後打印更詳盡的指標，默認為false |
| --transactional-id | 指定事務ID，測試並發事務的性能時需要，只有在--transaction-duration-ms > 0 時生效，默認值為performance-producer-default-transactional-id |
| --transactional-duration-ms | 指定事務持續的最長時間，超過這段時間後就會調用commitTransaction 來提交事務，只有指定了> 0 的值才會開啟事務，默認值為 0 |
| --record-size | 指定每條消息的大小，單位是字節，和--payload-file 兩個中必須指定一個，但不能同時指定 |
| --payload-file | 指定消息的來源文件，只支持UTF-8 編碼的文本文件，文件的消息分隔符通過--payload-delimeter 指定，和--record-size 兩個中必須指定一個，但不能同時指定 |

<br>

# Reference

---

[https://blog.csdn.net/spirit_8023/article/details/125525175](https://blog.csdn.net/spirit_8023/article/details/125525175)

[https://www.readfog.com/a/1668730672433434624](https://www.readfog.com/a/1668730672433434624)

[https://www.cnblogs.com/lkxed/p/kafka-perf-test-tools.html](https://www.cnblogs.com/lkxed/p/kafka-perf-test-tools.html)


<br>
