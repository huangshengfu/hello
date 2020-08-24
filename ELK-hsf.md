##################
此项目在华为云上搭建
搭建华为云内部yum源
##################
[root@proxy ~]# root mkdir -p /etc/yum.repos.d/repo_bak/
[root@proxy ~]# mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/repo_bak/
[root@proxy ~]# curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.myhuaweicloud.com/repo/CentOS-Base-7.repo
[root@proxy ~]# yum makeche
[root@proxy ~]# yum repolist

由于yum源内没有ELK的源码包自行下载
elasticsearch-2.3.4. rpm
filebeat-1.2.3-x86_ 64. rpm
kibana-4.5.2-1. x86_ 64. rpm
Iogstash-2. 3.4-1. noarch. rpm

跳板机上配置私有yum仓库
[root@proxy ~]# mkdir -p /var/ftp/localrepo/elk/   //将所需软件包上传至改目录下
[root@proxy elk]# yum -y install  createrepo
[root@proxy elk]# createrepo --update .   //更新私有仓库

#################
elasticsearch安装
#################
购买5台云服务去做集群 
ip规划
proxy         192.168.1.10
es-000-0001   192.168.1.11
es-000-0002   192.168.1.12
es-000-0003   192.168.1.13
es-000-0004   192.168.1.14
es-000-0005   192.168.1.15
logstash      192.168.1.16
es-000-0001.0005机器
利用lftp下载软件包
[root@es-000-0001 ~]# lftp 192.168.1.10
lftp 192.168.1.10:/> cd localrepo/elk
lftp 192.168.1.10:/localrepo/elk> mget *

[root@es-000-0001 ~]#  yum -y install  *.rpm
[root@es-000-0001 ~]#  yum -y install java-1.8.0-openjdk
[root@es-000-0001 ~]#  vim /etc/hosts
192.168.1.11 es-000-0001
192.168.1.12 es-000-0002
192.168.1.13 es-000-0003
192.168.1.14 es-000-0004
192.168.1.15 es-000-0005
[root@es-000-0001 ~]#  vim /etc/elasticsearch/elasticsearch.yml 
17 cluster.name: my-elk  //集群名
23 node.name: es-000-0001     //本机名
54 network.host: 0.0.0.0  //允许监听所有主机 
68 discovery.zen.ping.unicast.hosts: ["es-000-0001", "es-000-0002","es-000-0003"] //集群主机列表，不用全部都写
[root@es-000-0001 ~]#  systemctl  enable --now elasticsearch.service   //启动，并设置开机自启
5台机器全部都这么操作,一台配置好可用for循环scp穿过去，节约时间
最后结果如下，任意节点验证

[root@es-000-0001 ~]# curl http://192.168.1.11:9200/_cluster/health?pretty
{
  "cluster_name" : "my-elk",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 5,
  "number_of_data_nodes" : 5,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
返回字段解析
status: green
集群状态，绿色为正常
-黄色表示有问题但不是很严重，红色表示严重故障
number_ of_ nodes: 5
表示集群中节点的数量
number_ of_ data_ nodes: 5
用于存储数据节点数量

安装head插件
将源码包上传至跳板机利用lftp下载到本地
[root@proxy elk]# ls    //跳板机上上传
bigdesk-master.zip             elasticsearch-kopf-master.zip
elasticsearch-head-master.zip  logs.jsonl.gz

cd /usr/share/elasticsearch/bin/
./plugin install file:///root/bigdesk-master.zip
./plugin  install file:///root/elasticsearch-head-master.zip
./plugin  install file:///root/elasticsearch-kopf-master.zip
[root@es-000-0001 bin]#  ./plugin  list
Installed plugins in /usr/share/elasticsearch/plugins:
    - head
    - bigdesk
    - kopf


###########
kibana安装
##########
为了节省资源，在es-000-0001上安装
因为之前已经yum install *.rpm 包含了kibana，这里直接修改配置文件即可
[root@es-000-0001 ~]# vim /opt/kibana/config/kibana.yml 
  2 server.port: 5601  
  5 server.host: "0.0.0.0"
 15 elasticsearch.url: "http://localhost:9200"   //本机
 23 kibana.index: ".kibana"
 26 kibana.defaultAppId: "discover"
 
[root@es-000-0001 ~]# systemctl  enable  --now kibana.service
[root@es-000-0001 ~]# ss -ntulp | grep 5601
tcp    LISTEN     0      511       *:5601                  *:*                   users:(("node",pid=12635,fd=10))

然后在华为云负载均衡上添加监听器，集群监听5601端口

##############
Logstash安装
#############
另购买一台云服务器
Logstash 192.168.1.16
[root@logstash ~]# yum install -y java-1.8.0-openjdk  logstash
[root@logstash ~]# vim /etc/hosts
192.168.1.11 es-000-0001
192.168.1.12 es-000-0002
192.168.1.13 es-000-0003
192.168.1.14 es-000-0004
192.168.1.15 es-000-0005
192.168.1.16 logstash

[root@logstash ~]# touch /etc/logstash/logstash.conf  //创建配置文件
logstash没有默认的配置文件
需要自己手动编写
[root@logstash ~]# vim /etc/logstash/logstash.conf 
input {
 stdin {}
 }
filter{ }
output{
 stdout {}
 }
[root@logstash ~]# /opt/logstash/bin/logstash -f /etc/logstash/logstash.conf  //启动命令+验证
Settings: Default pipeline workers: 2
Pipeline main started
hello world!
2020-08-24T05:28:48.545Z logstash hello world!




