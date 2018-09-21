### 环境说明
- linux：centos7
- elasticsearch：5.6.4
- elasticsearch-head：5

### 目录结构
[项目源码下载](https://github.com/qijian0503/docker-elasticsearch)
```
elasticsearch/
├── docker-compose.yml
├── head
└── node
    ├── es1
    │   ├── data
    │   └── elasticsearch.yml
    └── es2
        ├── data
        └── elasticsearch.yml

```

### 主节点elasticsearch.yml配置文件
elasticsearch/node/es1/elasticsearch.yml

```
network.bind_host: 0.0.0.0
cluster.name: elasticsearch_cluster
cluster.routing.allocation.disk.threshold_enabled: false
node.name: master
node.master: true
node.data: true
http.cors.enabled: true
http.cors.allow-origin: "*"
network.host: 0.0.0.0
discovery.zen.minimum_master_nodes: 1
```

### 从节点elasticsearch.yml配置文件
elasticsearch/node/es2/elasticsearch.yml

```
network.bind_host: 0.0.0.0
cluster.name: elasticsearch_cluster
cluster.routing.allocation.disk.threshold_enabled: false
node.name: node2
node.master: false
node.data: true
http.cors.enabled: true
http.cors.allow-origin: "*"
network.host: 0.0.0.0
discovery.zen.minimum_master_nodes: 1
discovery.zen.ping.unicast.hosts: es1
```

### docker-compose.yml配置文件

```
version: '2.0'
services:
    elasticsearch-central:
        image: elasticsearch:5.6.4
        container_name: es1
        volumes:
           - /opt/modules/elasticsearch/node/es1/data:/usr/share/elasticsearch/data 
           - /opt/modules/elasticsearch/node/es1/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
        environment:
           - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
           - ES_CLUSTERNAME=elasticsearch
        command: elasticsearch
        ports:
           - "9200:9200"
           - "9300:9300"
    elasticsearch-data:
        image: elasticsearch:5.6.4
        container_name: es2
        volumes:
           - /opt/modules/elasticsearch/node/es2/data:/usr/share/elasticsearch/data
           - /opt/modules/elasticsearch/node/es2/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
        environment:
           - bootstrap.memory_lock=true
           - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
           - ES_CLUSTERNAME=elasticsearch
        command: elasticsearch
        ports:
           - "9201:9200"
           - "9301:9300"
        links:
           - elasticsearch-central:elasticsearch
    elasticsearch-head:
        image: mobz/elasticsearch-head:5
        container_name: head
        volumes:
           - /opt/modules/elasticsearch/head/Gruntfile.js:/usr/src/app/Gruntfile.js
           - /opt/modules/elasticsearch/head/_site/app.js:/usr/src/app/_site/app.js        
        ports:
           - "9100:9100"           
        links:
           - elasticsearch-central:elasticsearch
```

### 配置head
- 下载elasticsearch-head
```
cd elasticsearch
git clone git://github.com/mobz/elasticsearch-head.git
mv elasticsearch-head head
```
下载下来的代码结构如下：  
![elasticsearch-head目录](https://upload-images.jianshu.io/upload_images/8760038-05ffe6d1d6353e55.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- `elasticsearch\head\Gruntfile.js`修改以下片段  
```
connect: {
  server: {
    options: {
        /* 默认监控：127.0.0.1,修改为：0.0.0.0 */
      hostname: '0.0.0.0',
      port: 9100,
      base: '.',
      keepalive: true
    }
  }
}
```
- `elasticsearch\head\_site\app.js`修改以下代码片段

```
* 修改localhost为elasticsearch集群地址，Docker部署中，一般是elasticsearch宿主机地址 */
this.base_uri = this.config.base_uri || this.prefs.get("app-base_uri") || "http://localhost:9200";
```


### 启动
- 运行elasticsearch需要vm.max_map_count至少需要262144内存  

```
切换到root用户修改配置sysctl.conf
vi /etc/sysctl.conf
在尾行添加以下内容   
vm.max_map_count=262144
并执行命令
sysctl -p
```
> elk启动的时候可能会提示如下错误:  
> max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]  

- 启动es
```
docker-compose up 启动
docker-compose down 关闭
```

### 测试
elasticsearch-head可视化页面：http://es所在机器IP:9100 
![elasticsearch-head可视化页面](https://upload-images.jianshu.io/upload_images/8760038-3c071a872d7965de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


参考链接：  
https://www.jianshu.com/p/a26c8c7226d7  
https://blog.csdn.net/sinat_31908303/article/details/80496349     
https://blog.csdn.net/ggwxk1990/article/details/78698648