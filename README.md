# Hadoop-ansible
- 利用ansible安装Hadoop相关组件
- 目前适用于 CentOS 7.x
- JDK 版本  Openjdk-1.8
- Hadoop 版本 3.0.0
- Hive 版本 2.3.2
- Hbase 版本 1.2.6

## 安装前所需操作
配置DNS服务器用于主机名解析或者更新所有集群服务器/etc/hosts

## 安装 Hadoop
1. 下载Hadoop
2. 更新vars/var_basic.yml中的 {{ download_path }}
```
download_path: "/home/pippo/Downloads" # your local path
hadoop_version: "3.0.0" # your hadoop version
hadoop_path: "/home/hadoop" # default in user "hadoop" home
hadoop_config_path: "/home/hadoop/hadoop-{{hadoop_version}}/etc/hadoop"
hadoop_tmp: "/home/hadoop/tmp"
hadoop_dfs_name: "/home/hadoop/dfs/name"
hadoop_dfs_data: "/home/hadoop/dfs/data"

```
3. 采用ansible template动态生成配置文件, 如果你需要增加配置,可以直接更新vars/var_xxx.yml中相关properties数组.Hadoop默认配置为

```
# hadoop configration
hdfs_port: 9000
core_site_properties:
  - {
      "name":"fs.defaultFS",
      "value":"hdfs://{{ master_ip }}:{{ hdfs_port }}"
  }
  - {
      "name":"hadoop.tmp.dir",
      "value":"file:{{ hadoop_tmp }}"
  }
  - {
    "name":"io.file.buffer.size",
    "value":"131072"
  }

dfs_namenode_httpport: 9001
hdfs_site_properties:
  - {
      "name":"dfs.namenode.secondary.http-address",
      "value":"{{ master_hostname }}:{{ dfs_namenode_httpport }}"
  }
  - {
      "name":"dfs.namenode.name.dir",
      "value":"file:{{ hadoop_dfs_name }}"
  }
  - {
      "name":"dfs.namenode.data.dir",
      "value":"file:{{ hadoop_dfs_data }}"
  }
  - {
      "name":"dfs.replication",
      "value":"{{ groups['workers']|length }}"
  }
  - {
    "name":"dfs.webhdfs.enabled",
    "value":"true"
  }

mapred_site_properties:
 - {
   "name": "mapreduce.framework.name",
   "value": "yarn"
 }
 - {
   "name": "mapreduce.admin.user.env",
   "value": "HADOOP_MAPRED_HOME=$HADOOP_COMMON_HOME"
 }
 - {
   "name":"yarn.app.mapreduce.am.env",
   "value":"HADOOP_MAPRED_HOME=$HADOOP_COMMON_HOME"
 }

yarn_resourcemanager_port: 8040
yarn_resourcemanager_scheduler_port: 8030
yarn_resourcemanager_webapp_port: 8088
yarn_resourcemanager_tracker_port: 8025
yarn_resourcemanager_admin_port: 8141

yarn_site_properties:
  - {
    "name":"yarn.resourcemanager.address",
    "value":"{{ master_hostname }}:{{ yarn_resourcemanager_port }}"
  }
  - {
    "name":"yarn.resourcemanager.scheduler.address",
    "value":"{{ master_hostname }}:{{ yarn_resourcemanager_scheduler_port }}"
  }
  - {
    "name":"yarn.resourcemanager.webapp.address",
    "value":"{{ master_hostname }}:{{ yarn_resourcemanager_webapp_port }}"
  }
  - {
    "name": "yarn.resourcemanager.resource-tracker.address",
    "value": "{{ master_hostname }}:{{ yarn_resourcemanager_tracker_port }}"
  }
  - {
    "name": "yarn.resourcemanager.admin.address",
    "value": "{{ master_hostname }}:{{ yarn_resourcemanager_admin_port }}"
  }
  - {
    "name": "yarn.nodemanager.aux-services",
    "value": "mapreduce_shuffle"
  }
  - {
    "name": "yarn.nodemanager.aux-services.mapreduce.shuffle.class",
    "value": "org.apache.hadoop.mapred.ShuffleHandler"
  }
```


---
**注意**
```
hdfs_site_properties:
  - {
      "name":"dfs.namenode.secondary.http-address",
      "value":"{{ master_hostname }}:{{ dfs_namenode_httpport }}"
  }
  - {
      "name":"dfs.namenode.name.dir",
      "value":"file:{{ hadoop_dfs_name }}"
  }
  - {
      "name":"dfs.namenode.data.dir",
      "value":"file:{{ hadoop_dfs_data }}"
  }
  - {
      "name":"dfs.replication",
      "value":"{{ groups['workers']|length }}"  # this is  the group "workers" you define in hosts/host
  }
  - {
    "name":"dfs.webhdfs.enabled",
    "value":"true"
  }
```

### 安装 Master
1. 查看master.yml
```
- hosts: master
  remote_user: root
  vars_files:
   - vars/user.yml
   - vars/var_basic.yml
   - vars/var_master.yml
  vars:
     add_user: true           # add user "hadoop"
     generate_key: true       # generate the ssh key
     open_firewall: true      # for CentOS 7.x is firewalld
     disable_firewall: false  # disable firewalld
     install_hadoop: true     # install hadoop,if you just want to update the configuration, set to false
     config_hadoop: true      # Update configuration
  roles:
    - user                    # add user and generate the ssh key
    - fetch_public_key        # get the key and put it in your localhost
    - authorized              # push the ssh key to the remote server
    - java                    # install jdk
    - hadoop                  # install hadoop

```
2. 执行shell命令

```
ansible-playbook -i hosts/host master.yml
```

### 安装 Workers
1. 查看 workers.yml
```
# Add Master Public Key   # get master ssh public key
- hosts: master
  remote_user: root
  vars_files:
   - vars/user.yml
   - vars/var_basic.yml
   - vars/var_workers.yml
  roles:
    - fetch_public_key

- hosts: workers
  remote_user: root
  vars_files:
   - vars/user.yml
   - vars/var_basic.yml
   - vars/var_workers.yml
  vars:
    add_user: true
    generate_key: false # workers just use master ssh public key
    open_firewall: false
    disable_firewall: true  # disable firewall on workers
    install_hadoop: true
    config_hadoop: true
  roles:
    - user
    - authorized
    - java
    - hadoop

```
执行shell:

```
master_ip:  your hadoop master ip
master_hostname: your hadoop master hostname

above two variables must be same like your real hadoop master

ansible-playbook -i hosts/host workers.yml -e "master_ip=172.16.251.70 master_hostname=hadoop-master"

```

### 安装 hive
1. **建立数据库，并且赋予正确的权限**
2. 查看 vars/var_hive.yml
```
---

# hive basic vars
download_path: "/home/pippo/Downloads"                                 # your download path
hive_version: "2.3.2"                                                  # your hive version
hive_path: "/home/hadoop"
hive_config_path: "/home/hadoop/apache-hive-{{hive_version}}-bin/conf"
hive_tmp: "/home/hadoop/hive/tmp"                                      # your hive tmp path

hive_create_path:
  - "{{ hive_tmp }}"

hive_warehouse: "/user/hive/warehouse"                                # your hdfs path
hive_scratchdir: "/user/hive/tmp"
hive_querylog_location: "/user/hive/log"

hive_hdfs_path:
  - "{{ hive_warehouse }}"
  - "{{ hive_scratchdir }}"
  - "{{ hive_querylog_location }}"

hive_logging_operation_log_location: "{{ hive_tmp }}/{{ user }}/operation_logs"

# database
db_type: "postgres"                                              # use your db_type, default is postgres
hive_connection_driver_name: "org.postgresql.Driver"
hive_connection_host: "172.16.251.33"
hive_connection_port: "5432"
hive_connection_dbname: "hive"
hive_connection_user_name: "hive_user"
hive_connection_password: "nfsetso12fdds9s"
hive_connection_url: "jdbc:postgresql://{{ hive_connection_host }}:{{ hive_connection_port }}/{{hive_connection_dbname}}?ssl=false"
# hive configration                                             # your hive site properties
hive_site_properties:
  - {
      "name":"hive.metastore.warehouse.dir",
      "value":"hdfs://{{ master_hostname }}:{{ hdfs_port }}{{ hive_warehouse }}"
  }
  - {
      "name":"hive.exec.scratchdir",
      "value":"{{ hive_scratchdir }}"
  }
  - {
      "name":"hive.querylog.location",
      "value":"{{ hive_querylog_location }}/hadoop"
  }
  - {
      "name":"javax.jdo.option.ConnectionURL",
      "value":"{{ hive_connection_url }}"
  }
  - {
    "name":"javax.jdo.option.ConnectionDriverName",
    "value":"{{ hive_connection_driver_name }}"
  }
  - {
    "name":"javax.jdo.option.ConnectionUserName",
    "value":"{{ hive_connection_user_name }}"
  }
  - {
    "name":"javax.jdo.option.ConnectionPassword",
    "value":"{{ hive_connection_password }}"
  }
  - {
    "name":"hive.server2.logging.operation.log.location",
    "value":"{{ hive_logging_operation_log_location }}"
  }

hive_server_port: 10000                                    # hive port
hive_hwi_port: 9999
hive_metastore_port: 9083

firewall_ports:
  - "{{ hive_server_port }}"
  - "{{ hive_hwi_port }}"
  - "{{ hive_metastore_port }}"
```

3. 查看 hive.yml

```
- hosts: hive                   # in hosts/host
  remote_user: root
  vars_files:
   - vars/user.yml
   - vars/var_basic.yml
   - vars/var_master.yml
   - vars/var_hive.yml            # var_hive.yml
  vars:
     open_firewall: true           
     install_hive: true           
     config_hive: true
     init_hive: true               # init hive database after install and config
  roles:
    - hive

```
4. 执行shell

```
ansible-playbook -i hosts/host hive.yml

```

## Hbase

1. 查看 vars/var_hbase.yml

```
---
# hbase basic vars
download_path: "/home/pippo/Downloads"    # your local download path
hbase_version: "1.2.6"                    # hbase version
hbase_path: "/home/hadoop"                # install dir
hbase_config_path: "/home/hadoop/hbase-{{hbase_version}}/conf"

hbase_tmp: "{{ hbase_path }}/hbase/tmp"
hbase_zookeeper_config_path: "{{ hbase_path }}/hbase/zookeeper"
hbase_log_path: "{{ hbase_path }}/hbase/logs"

hbase_create_path:
  - "{{ hbase_tmp }}"
  - "{{ hbase_zookeeper_config_path }}"
  - "{{ hbase_log_path }}"


# hbase configration
hbase_hdfs_path: "/hbase"
zk_hosts: "zookeeper"                  #zookeeper group name in hosts/host
zk_client_port: 12181                  #zookeeper client port

hbase_master_port: 60000               #hbase port
hbase_master_info_port: 60010
hbase_regionserver_port: 60020
hbase_regionserver_info_port: 60030
hbase_rest_port: 8060


hbase_site_properties:                 #property
  - {
      "name":"hbase.master",
      "value":"{{ hbase_master_port }}"
  }
  - {
      "name":"hbase.master.info.port",
      "value":"{{ hbase_master_info_port }}"
  }
  - {
      "name":"hbase.tmp.dir",
      "value":"{{ hbase_tmp }}"
  }
  - {
      "name":"hbase.rootdir",
      "value":"hdfs://{{ master_hostname }}:{{ hdfs_port }}{{ hbase_hdfs_path }}"
  }
  - {
      "name":"hbase.regionserver.port",
      "value":"{{ hbase_regionserver_port }}"
  }
  - {
      "name":"hbase.regionserver.info.port",
      "value":"{{ hbase_regionserver_info_port }}"
  }
  - {
      "name":"hbase.rest.port",
      "value":"{{ hbase_rest_port }}"
  }
  - {
      "name":"hbase.cluster.distributed",
      "value":"true"
  }
  - {
      "name":"hbase.zookeeper.quorum"
  }


firewall_ports:
  - "{{ hbase_master_port }}"
  - "{{ hbase_master_info_port }}"
  - "{{ hbase_regionserver_port }}"
  - "{{ hbase_regionserver_info_port }}"
  - "{{ hbase_rest_port }}"
 
```

2. 添加 zookeeper 至 hosts/host
```
[zookeeper]
172.16.251.70
172.16.251.71
172.16.251.72

```
3. zookeeper 集群快速安装可以使用 [zookeeper-ansible](https://gitee.com/pippozq/zookeeper-ansible)
4. 查看 hbase.yml

```
- hosts: hbase
  remote_user: root
  vars_files:
   - vars/user.yml
   - vars/var_basic.yml
   - vars/var_master.yml
   - vars/var_hbase.yml
  vars:
     open_firewall: true       # firewalld
     install_hbase: true       # install hbase
     config_hbase: true        # config hbase
  roles:
    - hbase
```
5. 执行shell

```
ansible-playbook -i hosts/host  hbase.yml

```

### License

GNU General Public License v3.0