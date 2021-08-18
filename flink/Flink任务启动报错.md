# Flink任务启动报错

在Flink版本为1.11.1和Hadoop版本为2.7.4？

1. ytm：taskmanager的内存
2. yjm：jobmanager的内存  ProcessMemoryUtils：The derived from fraction jvm overhead memory(102.4mb) is less than its min value 192.0mb,min value will be used instead
3. 若ytm和yjm设置过小，会报内存异常

1）flink on yarn时，AM(Application Master) container的exitCode为1

  可能原因：资源不足，在yarn-site.xml的配置文件改为4

2）通过yarn logs -applicationId application_xxxx查看日志，报错如下：

org.apache.flink.runtime.entrypoint.clusterentrypoint could not start cluster entrypoint YarnJobClusterEntrypoint

原因：flink的jar版本和环境对应版本不一样，请升级版本