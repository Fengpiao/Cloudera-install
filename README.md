## 安装Cloudera相关服务  
Cloudera产品的安装需要依赖数据库，这里选择外部MySql数据库  
主节点上安装mysql，执行：  

    $ sudo apt-get install mysql-server  

备份原有的数据库文件：  

    $ sudo mkdir /var/lib/mysql_backup  
    $ sudo mv /var/lib/mysql/ib_logfile0 /var/lib/mysql_backup  
    $ sudo mv /var/lib/mysql/ib_logfile1 /var/lib/mysql_backup  
    $ sudo cp /etc/mysql/my.cnf /var/lib/mysql_backup  

启动Mysql服务：  

    $ sudo service mysql start  

初始化数据库及用户  

    $ mysql -uroot -p < init.sql  

## 安装Cloudera
Cloudera Manager下载地址: http://archive.cloudera.com/cm5/cm/5/  
CDH安装包地址：http://archive.cloudera.com/cdh5/parcels/5.9.0/  

主节点上解压cm压缩包并放在/opt目录下  

    $ sudo tar -zxvf cloudera-manager-trusty-cm5.9.0_amd64.tar.gz -C /opt  

去MySql的官网下载JDBC驱动，不论哪一个版本都可以主要是要解压后的mysql-connector-java-5.1.41-bin.jar 文件，放到主节点上的 /opt/cm-5.9.0/share/cmf/lib/下。  
初始化cloudera manager数据库配置，主节点上执行：  

    $ sudo /opt/cm-5.9.0/share/cmf/schema/scm_prepare_database.sh mysql cm scm cdh -uroot -proot106A –-scm-host cdh1 scm scm scm  

## 其他配置  
1.在所有节点创建cloudera-scm用户  

    $ sudo useradd --system --home=/opt/cm-5.9.0/run/cloudera-scm-server/ --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm  

2.修改cloudera-scm-agent/config.ini中的server_host为主节点的主机名  

    $ sudo vim /opt/cm-5.9.0/etc/cloudera-scm-agent/config.ini  
    server_host=cdh1  

3.同步Agent到其他节点（因为直接放在opt目录下权限拒绝，所以先间接放在home目录下再移到opt目录下）  

    $ scp -r /opt/cm-5.9.0 cst@166.111.7.247:/home/cst  
    $ sudo mv /home/cst/cm-5.9.0 /opt/cm-5.9.0   

4.准备Parcels，用以安装CDH5  
将CHD5相关的Parcel包放到主节点的/opt/cloudera/parcel-repo/目录中（parcel-repo需要手动创建）。   
相关的文件如下：  

    CDH-5.9.0-1.cdh5.9.0.p0.23-trusty.parcel   
    CDH-5.9.0-1.cdh5.9.0.p0.23-trusty.parcel.sha1   
    manifest.json  
    
最后将CDH-5.9.0-1.cdh5.9.0.p0.23-wheezy.parcel.sha1重命名为CDH-5.9.0-1.cdh5.9.0.p0.23-wheezy.parcel.sha  

5.相关启动脚本  

    $ sudo /opt/cm-5.9.0/etc/init.d/cloudera-scm-server start   
    $ sudo /opt/cm-5.9.0/etc/init.d/cloudera-scm-agent start   
    
<主节点都启动、其他节点只启动agent>  
打开浏览器 输入http://cdh1:7180 ,按照步骤引导开始配置。  

完成Cloudera服务安装后，需要进行如下初始化工作：  
（1）在部署了HDFS Namenode的节点上以root用户运行fs-dir.sh 脚本;  
（2）修改HIve配置  
CM管理界面（Hive -> 配置 -> 服务范围 -> 高级 -> hive-site.xml 的 Hive 服务高级配置代码段 -> 点击右侧 View as XML）添加（更新）如下信息：  

    hive.exec.dynamic.partition.mode  
    nostrict  

    hive.exec.dynamic.partition  
    true  

    hive.exec.max.dynamic.partitions  
    81920  

    hive.exec.max.dynamic.partitions.pernode  
    81920  

    parquet.column.index.access  
    true  

    hive.merge.size.per.task    
    1024000000    

    hive.input.dir.recursive    
    true  

    hive.mapred.supports.subdirectories  
    true  

    hive.supports.subdirectories  
    true  

    mapred.input.dir.recursive  
    true  

(3) 配置主机监控触发器, CM管理界面->所有主机->配置->监控->主机触发器  

      [{"validityWindowInMs": 300000, "triggerName": "cpu_iowait_rate_bad", "suppressed": false, "triggerExpression": "IF (select cpu_iowait_rate where entityName=$HOSTID AND last(cpu_iowait_rate / getHostFact(numCores, 1)) > 0.15) DO health:bad", "enabled": true, "expressionEditorConfig": null, "streamThreshold": 3}, {"validityWindowInMs": 300000, "triggerName": "cpu_load_5_bad", "suppressed": false, "triggerExpression": "IF (select load_5 where entityName=$HOSTID AND last(load_5) > 10) DO health:bad", "enabled": true, "expressionEditorConfig": null, "streamThreshold": 3}, {"validityWindowInMs": 300000, "triggerName": "cpu_soft_irq_rate_bad", "suppressed": false, "triggerExpression": "IF (select cpu_iowait_rate where entityName=$HOSTID AND last(cpu_soft_irq_rate / getHostFact(numCores, 1)) > 0.02) DO health:bad", "enabled": true, "expressionEditorConfig": null, "streamThreshold": 3}, {"validityWindowInMs": 300000, "triggerName": "memory_available_bad", "suppressed": false, "triggerExpression": "IF (select physical_memory_used, physical_memory_total where entityName=$HOSTID AND last(physical_memory_used/physical_memory_total) > 0.95) DO health:bad", "enabled": true, "expressionEditorConfig": null, "streamThreshold": 3}, {"validityWindowInMs": 300000, "triggerName": "net_receive_rate_bad", "suppressed": false, "triggerExpression": "IF (select total_bytes_receive_rate_across_network_interfaces where entityName=$HOSTID AND last( total_bytes_receive_rate_across_network_interfaces)>50Mb) DO health:bad", "enabled": true, "expressionEditorConfig": null, "streamThreshold": 3}, {"validityWindowInMs": 300000, "triggerName": "disk_free_percent_bad", "suppressed": false, "triggerExpression": "IF (select capacity_free, capacity where entityName=$HOSTID AND last(capacity_free/capacity)<0.05) DO health:bad", "enabled": true, "expressionEditorConfig": null, "streamThreshold": 3}, {"validityWindowInMs": 300000, "triggerName": "cpu_iowait_rate_concerning", "suppressed": false, "triggerExpression": "IF (select cpu_iowait_rate where entityName=$HOSTID AND last(cpu_iowait_rate / getHostFact(numCores, 1)) > 0.15) DO health:concerning", "enabled": true, "expressionEditorConfig": null, "streamThreshold": 3}, {"validityWindowInMs": 300000, "triggerName": "cpu_load_5_concerning", "suppressed": false, "triggerExpression": "IF (select load_5 where entityName=$HOSTID AND last(load_5) > 8) DO health:bad", "enabled": true, "expressionEditorConfig": null, "streamThreshold": 3}, {"validityWindowInMs": 300000, "triggerName": "cpu_soft_irq_rate_concerning", "suppressed": false, "triggerExpression": "IF (select cpu_iowait_rate where entityName=$HOSTID AND last(cpu_soft_irq_rate / getHostFact(numCores, 1)) > 0.015) DO health:concerning", "enabled": true, "expressionEditorConfig": null, "streamThreshold": 3}, {"validityWindowInMs": 300000, "triggerName": "memory_available_concerning", "suppressed": false, "triggerExpression": "IF (select physical_memory_used, physical_memory_total where entityName=$HOSTID AND last(physical_memory_used/physical_memory_total) > 0.9) DO health:concerning", "enabled": true, "expressionEditorConfig": null, "streamThreshold": 3}, {"validityWindowInMs": 300000, "triggerName": "net_receive_rate_concerning", "suppressed": false, "triggerExpression": "IF (select total_bytes_receive_rate_across_network_interfaces where entityName=$HOSTID AND last( total_bytes_receive_rate_across_network_interfaces)>40Mb) DO health:concerning", "enabled": true, "expressionEditorConfig": null, "streamThreshold": 3}, {"validityWindowInMs": 300000, "triggerName": "disk_free_percent_concerning", "suppressed": false, "triggerExpression": "IF (select capacity_free, capacity where entityName=$HOSTID AND last(capacity_free/capacity)<0.1) DO health:concerning", "enabled": true, "expressionEditorConfig": null, "streamThreshold": 3}]  
