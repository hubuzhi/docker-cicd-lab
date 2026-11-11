# 第一阶段：Maven 构建
FROM swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/maven:3.8-openjdk-17 AS builder

WORKDIR /app
COPY pom.xml .
COPY src ./src
# 在容器内直接打包，完全不需要依赖 Gitee 宿主机的 Maven 环境
RUN mvn clean package -Dmaven.test.skip=true

# 第二阶段：运行环境
FROM swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/maven:3.8-openjdk-17

WORKDIR /app
# 从第一阶段拷贝打好的 Jar 包
COPY --from=builder /app/target/demo-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
