随着xxxx

构建镜象
``` shell
./bin/docker-image-tool.sh  -t base build
```

Dockerfile
``` dockerfile
FROM spark:base
COPY target/xxx.jar /opt/xxx.jar
```
构建镜象
``` shell
./bin/docker-image-tool.sh  -t base build
docker build -t spark-k8s-demo .
docker image tag spark-k8s-demo  remote/spark-k8s-demo:v1.0
docker push remote/spark-k8s-demo:v1.0
```
提交任务
``` shell
./bin/spark-submit    
	--master k8s-http-url
	--name spark-pi  
	--num-executors 1  
	--deploy-mode cluster
	--conf spark.kubernetes.driver.limit.cores=1
	--conf spark.kubernetes.driver.request.cores=0.1 
	--conf spark.kubernetes.executor.limit.cores=1
	--conf spark.kubernetes.executor.request.cores=0.1
	--conf spark.kubernetes.container.image.pullSecrets=harbor-key
	--conf spark.kubernetes.container.image.pullPolicy=Always
	--conf spark.kubernetes.container.image=remote/spark-k8s-demo:v1.0
	--conf spark.kubernetes.namespace=spark-app
	--class SparkPi  local:///opt/spark-k8s-demo-1.0-SNAPSHOT-jar-with-dependencies.jar args
```

spark-on-k8s 模式：
1. cluster模式
2. driver模式

启动spark-history-server  查看历史任务，历史任务存储在s3目录
spark-defaults.conf 配置项
``` config
spark.history.fs.logDirectory 	 	s3a://xxx/spark-events
spark.hadoop.fs.s3a.impl          	org.apache.hadoop.fs.s3a.S3AFileSystem
spark.hadoop.fs.s3a.aws.credentials.provider  org.apache.hadoop.fs.s3a.SimpleAWSCredentialsProvider
spark.hadoop.fs.s3a.access.key 		[s3-key]
spark.hadoop.fs.s3a.secret.key		[s3-secret]
spark.hadoop.fs.s3a.endpoint		[s3-region-endpoing]
```
启动history-server
``` shell
./sbin/start-history-server.sh
```


flink 提交方式
``` Dockerfile
Dockerfile

FROM flink:1.15.2
COPY target/xxx.jar /opt/xxx.jar
```
``` shell
flink run-application 
	--target kubernetes-application 
	-Dkubernetes.cluster-id=flink-k8s-demo 
	-Dkubernetes.namespace=flink-app 
	-Dkubernetes.container.image.pull-secrets=harbor-key 
	-Dkubernetes.container.image.pull-policy=Always 
	-Dkubernetes.container.image=us-harbor.huoli101.com/yelichen/flink-k8s-demo:v1.0 
	-Dkubernetes.jobmanager.cpu=0.1 
	-Dkubernetes.taskmanager.cpu=0.1 
	-Djobmanager.memory.process.size=1024M 
	-Dtaskmanager.memory.process.size=1024M 
	-Dcontainerized.master.env.ENABLE_BUILT_IN_PLUGINS=flink-s3-fs-hadoop-1.15.2.jar 
	-Dcontainerized.taskmanager.env.ENABLE_BUILT_IN_PLUGINS=flink-s3-fs-hadoop-1.15.2.jar 
	-c WordCount local:///opt/flink-k8s-demo-1.0-SNAPSHOT-jar-with-dependencies.jar 
	--input s3://cf-supply/algo-center/user/cyl/WordCountData.txt
```

flink-on-k8s 模式：
1. application模式
2. session模式


![[k8s-on.png]]



| application  | mode | node | act |
|------|----|----|----|
| spark | client | driver | xxx |
| spark | client | executor | xxx |
| spark | cluster | driver | xxx |


