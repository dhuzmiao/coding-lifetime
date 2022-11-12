# 前提

本文介绍使用 Prometheus 与 JMX Exporter 来监控 Java 应用的 JVM。

# 什么是 JMX Exporter ?

JMX Exporter 利用 Java 的 JMX 机制来读取 JVM 运行时的数据，然后转换成 Prometheus 认识的 Metrics 格式，以便让 Prometheus 对其进行监控采集。
JMX 的全称是 `Java Management Extensions`，是管理 Java 的一种扩展框架，JMX Exporter 正是基于此框架来读取 JVM 的运行状态的。

# 如何使用 JMX Exporter 暴露 JVM 监控指标 ?
JMX Exporter 有两种使用方法
1. 启动独立进程。JVM 启动时指定参数，暴露 JMX 的 RMI 接口， JMX-Exporter 调用 RMI 获取 JVM 运行时状态数据，转换为 Prometheus metrics 格式，并暴露端口让 Prometheus 采集。 
2. JVM 进程内启动(in-process)。JVM 启动时指定参数，通过 javaagent 的形式运行 JMX-Exporter 的 jar 包，进程内读取 JVM 运行时状态数据，转换为 Prometheus metrics 格式，并暴露端口让 Prometheus 采集。

官方不推荐使用第一种方式，一方面配置复杂，另一方面因为它需要一个单独的进程的，而这个进程本身的监控又成了新的问题，所以本文重点围绕第二种用法讲如何在 K8S 环境下使用 JMX Exporter 暴露 JVM 监控指标。

# 环境部署
环境部署主要涉及到两部分：
1.应用以及 JMX Exporter 包的容器化
2.K8S 环境 Deployment 和 Servcie 的声明

## 应用以及 JMX Exporter 包的容器化
首先准备一个制作镜像的目录，放入 JMX Exporter 配置文件 `prometheus-jmx-config.yaml`:
```
---
ssl: false
lowercaseOutputName: false
lowercaseOutputLabelNames: false
```
然后准备好 jmx_exporter 的 jar 包文件，可以在 [jmx_exporter](https://github.com/prometheus/jmx_exporter) 的 Github 页面找到 jar 包下载地址，下载到当前目录
```
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.13.0/jmx_prometheus_javaagent-0.13.0.jar
```
主服务以 tomcat 镜像构建，采用 docker 多阶段构建方式编写 Dockerfile
```
FROM ubuntu:16.04 as jar
WORKDIR /
RUN apt-get update -y
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y wget
RUN wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.13.0/jmx_prometheus_javaagent-0.13.0.jar

FROM tomcat:jdk8-openjdk-slim
ADD prometheus-jmx-config.yaml /prometheus-jmx-config.yaml
COPY --from=jar /jmx_prometheus_javaagent-0.13.0.jar /jmx_prometheus_javaagent-0.13.0.jar
```
编译镜像
```
docker build . -t dhuzmiao.lab.com/tomcat:jdk8
```

## K8S 环境 Deployment 和 Servcie 的声明 
部署应用到 K8S，修改 JVM 启动参数以便启动时加载 JMX Exporter。 JVM 启动时会读取 `JAVA_OPTS` 环境变量，作为额外的启动参数：
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      containers:
      - name: tomcat
        image: dhumizoa.lab.com/tomcat:jdk8
        env:
        - name: JAVA_OPTS
          value: "-javaagent:/jmx_prometheus_javaagent-0.13.0.jar=8088:/prometheus-jmx-config.yaml"

---
apiVersion: v1
kind: Service
metadata:
  name: tomcat
  labels:
    app: tomcat
spec:
  type: ClusterIP
  ports:
  - port: 8080
    protocol: TCP
    name: http
  - port: 8088
    protocol: TCP
    name: jmx-metrics
  selector:
    app: tomcat

```
 - 启动参数格式： `-javaagent:<jar><port>:<config>`
 - 这里使用 8088 端口暴露 JVM 的监控指标 


# 参考资料
 - 手把手教你使用 Prometheus 监控 JVM ： https://www.cnblogs.com/tencent-cloud-native/p/13807113.html