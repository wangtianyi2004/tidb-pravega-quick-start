# tidb-pravega-quick-start




## Bootstrap

```
## Clone project
git clone https://github.com/wangtianyi2004/tidb-pravega-quick-start.git
cd tidb-pravega-quick-start

## Reset env
docker-compose down
rm -rf logs

## Start up
export HOST_IP=192.168.232.66    ## change this ip for your env
docker-compose up -d
```

Now you can use flink dashboard in http://$HOST_IP:8081



## Check docker compose status

```
[root@test-quick-deploy tidb-pravega-quick-start]# docker-compose ps
                 Name                                Command                  State                                                                    Ports
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
tidb-pravega-quick-start_bookie1_1        /bin/bash /opt/bookkeeper/ ...   Up (healthy)   0.0.0.0:3181->3181/tcp,:::3181->3181/tcp
tidb-pravega-quick-start_bookie2_1        /bin/bash /opt/bookkeeper/ ...   Up (healthy)   3181/tcp, 0.0.0.0:3182->3182/tcp,:::3182->3182/tcp
tidb-pravega-quick-start_bookie3_1        /bin/bash /opt/bookkeeper/ ...   Up (healthy)   3181/tcp, 0.0.0.0:3183->3183/tcp,:::3183->3183/tcp
tidb-pravega-quick-start_controller_1     /opt/pravega/scripts/entry ...   Up             10000/tcp, 0.0.0.0:10080->10080/tcp,:::10080->10080/tcp, 12345/tcp, 0.0.0.0:9090->9090/tcp,:::9090->9090/tcp, 9091/tcp
tidb-pravega-quick-start_hdfs_1           /entrypoint.sh -d                Up             22/tcp, 0.0.0.0:2222->2222/tcp,:::2222->2222/tcp, 0.0.0.0:50010->50010/tcp,:::50010->50010/tcp,
                                                                                          0.0.0.0:50020->50020/tcp,:::50020->50020/tcp, 0.0.0.0:50070->50070/tcp,:::50070->50070/tcp,
                                                                                          0.0.0.0:50075->50075/tcp,:::50075->50075/tcp, 0.0.0.0:50090->50090/tcp,:::50090->50090/tcp,
                                                                                          0.0.0.0:8020->8020/tcp,:::8020->8020/tcp
tidb-pravega-quick-start_jobmanager_1     /docker-entrypoint.sh jobm ...   Up             6123/tcp, 0.0.0.0:8081->8081/tcp,:::8081->8081/tcp
tidb-pravega-quick-start_pd_1             /pd-server --name=pd --cli ...   Up             0.0.0.0:49162->2379/tcp,:::49162->2379/tcp, 2380/tcp
tidb-pravega-quick-start_segmentstore_1   /opt/pravega/scripts/entry ...   Up             10000/tcp, 0.0.0.0:12345->12345/tcp,:::12345->12345/tcp, 9090/tcp, 9091/tcp
tidb-pravega-quick-start_taskmanager_1    /docker-entrypoint.sh task ...   Up             6121/tcp, 6122/tcp, 6123/tcp, 8081/tcp
tidb-pravega-quick-start_tidb_1           /tidb-server --store=tikv  ...   Up             0.0.0.0:12080->12080/tcp,:::12080->12080/tcp, 0.0.0.0:4000->4000/tcp,:::4000->4000/tcp
tidb-pravega-quick-start_tikv_1           /tikv-server --addr=0.0.0. ...   Up             20160/tcp
tidb-pravega-quick-start_zookeeper_1      /docker-entrypoint.sh zkSe ...   Up             0.0.0.0:2181->2181/tcp,:::2181->2181/tcp, 2888/tcp, 3888/tcp, 8080/tcp

```





## Create scope & stream in Pravega

```
## Use pravega CLI tool in controller container
docker-compose exec controller ./bin/pravega-cli

Pravega User CLI Tool.
        Usage instructions: https://github.com/pravega/pravega/wiki/Pravega-User-CLI

Initial configuration:
        controller-uri=localhost:9090
        default-segment-count=4
        timeout-millis=60000
        max-list-items=1000
        pretty-print=true
        auth-enabled=false
        auth-username=
        auth-password=
        tls-enabled=false
        truststore-location=

Type "help" for list of commands, or "exit" to exit.

> scope create test
Scope 'test' created successfully.

> stream create test/test
Stream 'test/test' created successfully.


```



## Create test table in TiDB 

```
mysql -uroot -P4000 -h$HOST_IP
mysql> create table test.t3(id int);
Query OK, 0 rows affected (0.11 sec)

```





## Create test table in Flink for Pravega & TiDB

```
[root@test-quick-deploy tidb-pravega-quick-start]# docker-compose exec jobmanager ./bin/sql-client.sh embedded -l ./connector-lib
No default environment specified.
Searching for '/opt/flink/conf/sql-client-defaults.yaml'...found.
Reading default environment from: file:/opt/flink/conf/sql-client-defaults.yaml
No session environment specified.

Flink SQL> create table test (
>    id int
> ) with (
>    'connector' = 'pravega'
> ,  'controller-uri' = 'tcp://192.168.232.66:9090'
> ,  'scope' = 'test'
> ,  'scan.streams' = 'test'
> ,  'sink.stream' = 'test'
> ,  'format' = 'json'
> );
[INFO] Table has been created.

Flink SQL> create table t3 (
>    id int
> ) with (
>    'connector' = 'jdbc'
> ,  'url' = 'jdbc:mysql://192.168.232.66:4000/test'
> ,  'table-name' = 't3'
> ,  'username' = 'root'
> ,  'password' = ''
> );
>
[INFO] Table has been created.

```



## Insert data into Pravega

```
Flink SQL> insert into test values(3);
[INFO] Submitting SQL update statement to the cluster...
[INFO] Table update statement has been successfully submitted to the cluster:
Job ID: 168a3edb858836bcbfbe013c8808194c

```





## Create a data channel from Pravega to TiDB

```
Flink SQL> insert into t3 select * from test;
[INFO] Submitting SQL update statement to the cluster...
[INFO] Table update statement has been successfully submitted to the cluster:
Job ID: 97dada3ce3b483fd22d3782fc568c477
```



## Check the result in TiDB

```
mysql> select * from test.t3;
+------+
| id   |
+------+
|    3 |
+------+
1 row in set (0.00 sec)

```











