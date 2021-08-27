# Docker搭建ES集群
## 一、目的
>搭建一个master两个node节点的es集群以及kibana和cerebro  
同时es集群相关配置以及数据与日志挂载在linux本地
全程可以root权限
本次使用7.13.0版本，6.X版本注意yml中标注的6.x相关修改
## 二、修改Linux参数配置
```shell
#去掉锁定内存软限制
echo "* soft memlock unlimited" >> /etc/security/limits.conf
#去掉锁定内存硬限制
echo "* hard memlock unlimited" >> /etc/security/limits.conf
#最大使用物理内存 不使用swap分区
echo "vm.swappiness = 0" >> /etc/sysctl.conf
#最大文件连接句柄如果不调可能会报 max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
#使用配置
sysctl -p
```

## 三、下载es镜像，启动es并获取加密所需证书elastic-certificates.p12
```shell
#下载镜像
docker pull elasticsearch:7.13.0
#启动es容器
docker run -dit --name=es elasticsearch:7.13.0 /bin/bash
#进入容器内部
docker exec -it es /bin/bash

#生成elastic-stack-ca.p12（一直回车就好）
[root@6b52d0ce8f67 elasticsearch]# ./bin/elasticsearch-certutil ca

#生成elastic-certificates.p12，这是需要的（也是一直回车）
[root@6b52d0ce8f67 elasticsearch]# ./bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12

#需要复制出证书
#首先要退出容器ctrl+d
#docker cp到当前文件夹下
docker cp es:/usr/share/elasticsearch/elastic-certificates.p12 .
#关闭容器并删除
docker kill es
docker rm es
```


## 四、开始搭建es集群
### 1.创建文件夹并给与权限
```shell
mkdir -p es01/data
mkdir -p es01/logs
mkdir -p es02/data
mkdir -p es02/logs
mkdir -p es03/data
mkdir -p es03/logs
chmod 777 es* -R
```

### 2.创建es配置文件并配置
```shell
touch elasticsearch.yml
vim elasticsearch.yml
```
#### elasticsearch.yml
```shell
network.host: 0.0.0.0
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.keystore.type: PKCS12
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.type: PKCS12
xpack.security.audit.enabled: true
```


### 3.创建docker-compose的yml文件
```shell
touch es-7.13.0.yml
vim es-7.13.0.yml
```
#### es-7.13.0.yml
```shell
version: '2.2'
services:
  cerebro:
    image: lmenezes/cerebro:0.9.2
    container_name: cerebro
    ports:
      - "9000:9000"
    command:
      - -Dhosts.0.host=${你的}:9200
    networks:
      - elastic
 
  kibana:
    image: kibana:7.13.0
    container_name: kibana
    environment:
      - I18N_LOCALE=zh-CN
      - monitoring.enabled=false
      - XPACK_GRAPH_ENABLED=true
      - TIMELION_ENABLED=true
      - XPACK_MONITORING_COLLECTION_ENABLED="true"
    ports:
      - "5601:5601"
    networks:
      - elastic
 
  es01:
    image: elasticsearch:7.13.0
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      #7.x
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      #6.x
      #- discovery.zen.ping.unicast.hosts=es02,es03
      - bootstrap.memory_lock=true
      #内存不够就调低一些
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./es01/data:/usr/share/elasticsearch/data
      - ./es01/logs:/usr/share/elasticsearch/logs
      - ./elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./elastic-certificates.p12:/usr/share/elasticsearch/config/elastic-certificates.p12
      - ./plugins:/usr/share/elasticsearch/plugins
    ports:
      - 9200:9200
    networks:
      - elastic
  es02:
    image: elasticsearch:7.13.0
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      #7.x
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      #6.x
      #- discovery.zen.ping.unicast.hosts=es02,es03
      - bootstrap.memory_lock=true
      #内存不够就调低一些
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./es02/data:/usr/share/elasticsearch/data
      - ./es02/logs:/usr/share/elasticsearch/logs
      - ./elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./elastic-certificates.p12:/usr/share/elasticsearch/config/elastic-certificates.p12
      - ./plugins:/usr/share/elasticsearch/plugins
    ports:
      - 9201:9200
    networks:
      - elastic
  es03:
    image: elasticsearch:7.13.0
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      #7.x
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      #6.x
      #- discovery.zen.ping.unicast.hosts=es02,es03
      - bootstrap.memory_lock=true
      #内存不够就调低一些
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./es03/data:/usr/share/elasticsearch/data
      - ./es03/logs:/usr/share/elasticsearch/logs
      - ./elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./elastic-certificates.p12:/usr/share/elasticsearch/config/elastic-certificates.p12
      - ./plugins:/usr/share/elasticsearch/plugins
    ports:
      - 9202:9200
    networks:
      - elastic
 
volumes:
  data01:
    driver: local
  data02:
    driver: local
  data03:
    driver: local
 
networks:
  elastic:
    driver: bridge
```

### 4.启动es集群前，需要给两个文件权限
```shell
chmod -R 777 elastic-certificates.p12 elasticsearch.yml
```
### 此时文件夹中的所有文件
![](https://gitee.com/sjtu-lhq/blogImage/raw/master/%20images/20210827230849.png)
### 5.启动es集群
```shell
#后台启动
docker-compose -f es-7.13.0.yml up -d
```
### 启动后查看docker
![](https://gitee.com/sjtu-lhq/blogImage/raw/master/%20images/20210827233014.png)

### 6.es加密
```shell
#进入一台机器
docker exec -it es01 /bin/bash
#生成密码用auto， 自己设置用 interactive
[root@d9c1faa244c4 elasticsearch]# ./bin/elasticsearch-setup-passwords auto
```
![](https://gitee.com/sjtu-lhq/blogImage/raw/master/%20images/20210827234956.png)

### 7.修改kibana配置（可加密）
```shell
docker exec -it kibana /bin/bash
bash-4.4$ vi config/kibana.yml
```
#### kibana.yml
```shell
# ** THIS IS AN AUTO-GENERATED FILE **
#

# Default Kibana configuration for docker target
server.host: "0"
elasticsearch.hosts: [ "${你的ip}:9200" ]
monitoring.ui.container.elasticsearch.enabled: true
elasticsearch.username: "${你的用户名}"
elasticsearch.password: "${你的密码}"
xpack.security.encryptionKey: "something_at_least_32_characters"
```
```shell
重启kibana即可
```

## 五、完成图展示
周一到公司截图