# GitLab + Harbor + Jenkins 容器化 CI/CD 流水线

用 Docker 在一台机器上搭建完整的 CI/CD 环境：**代码托管（GitLab）→ 流水线（Jenkins）→ 私有镜像仓库（Harbor）**，
实现「`git push` 后自动构建镜像、推送仓库、重建容器」的闭环。

## 整体架构

```
开发者 push 代码
      │
      ▼
┌─────────────┐   Webhook    ┌─────────────┐   docker build/push   ┌─────────────┐
│   GitLab    │ ───────────► │   Jenkins   │ ────────────────────► │   Harbor    │
│ 代码托管     │              │  流水线调度  │                       │ 私有镜像仓库 │
└─────────────┘              └─────────────┘                       └─────────────┘
                                     │                                     │
                                     └──────── docker run 拉取镜像 ─────────┘
                                                     │
                                                     ▼
                                             应用容器（:8888）
```

三套服务运行在自定义 bridge 网络 `cicd-net`（172.28.0.0/24）中，通过**网络别名**以域名互访，
用单机容器方案替代「两台虚拟机」的传统部署方式。

| 服务 | 地址 | 容器 IP |
| --- | --- | --- |
| GitLab | http://localhost:8083 | 172.28.0.10 |
| Harbor | http://localhost:8080 | 172.28.0.15 |
| Jenkins | http://localhost:9090 | 172.28.0.20 |

## 关键实现

### 1. Jenkins 如何调用宿主机 Docker

Jenkins 跑在容器里，但需要执行 `docker build` / `docker push`。本方案没有使用 Docker-in-Docker，
而是**挂载宿主机的 docker.sock**：

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

同时在自定义镜像中安装 docker CLI：

```dockerfile
FROM jenkins/jenkins:lts
USER root
RUN curl -sSL https://mirrors.aliyun.com/docker-ce/linux/static/stable/x86_64/docker-27.3.1.tgz \
      -o /tmp/d.tgz && tar -xzf /tmp/d.tgz -C /tmp \
      && cp /tmp/docker/docker /usr/local/bin/docker && chmod +x /usr/local/bin/docker
USER jenkins
```

这样容器内的 `docker` 命令实际操作的是宿主机引擎，镜像直接进入宿主机镜像库。

### 2. 流水线四步

`jenkins/job.xml` 中的构建脚本依次执行：

1. **拉取代码**：`git clone` 从 GitLab 取最新代码
2. **构建镜像**：`docker build -t <registry>/<project>/<image>:v${BUILD_NUMBER}`
3. **推送 Harbor**：`docker login` 后 `docker push`，镜像按构建号打 tag 便于回退
4. **部署容器**：`docker rm -f` 旧容器后 `docker run -d` 起新容器，最后 `curl` 验证服务

### 3. GitLab → Jenkins 自动触发

- Jenkins 任务勾选「触发远程构建」并设置令牌，URL 形如
  `/job/springboot-cicd/build?token=<YOUR-TOKEN>`
- GitLab 项目配置 Webhook 指向该 URL，勾选 Push events
- GitLab 默认禁止 Webhook 访问内网，需要开通：
  `ApplicationSetting.current.update(allow_local_requests_from_web_hooks_and_services: true)`

## 部署过程中踩到的问题

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| GitLab 启动后 nginx 报 `still could not bind()` | Sidekiq 健康检查默认占用 8082，与 web 端口冲突 | Web 端口改用 8083 |
| `docker login` 返回 403 / 域名解析失败 | Docker 守护进程无法解析容器别名 | 把 Harbor 的 `hostname` 改成本机可达地址，并将 HTTP 仓库加入 `insecure-registries` |
| Webhook 返回 403 `No valid crumb` | Jenkins CSRF 保护 | 关闭 CSRF（`Jenkins.instance.setCrumbIssuer(null)`），并把授权策略改为允许匿名 |
| Jenkins 内 `docker: not found` | 重建容器后 CLI 丢失 | 把 docker CLI 写进自定义镜像 |
| 镜像推送中断 `blob upload invalid` | 网络瞬时故障 | 重试推送 |

## 目录结构

```
├── gitlab/docker-compose.yml     GitLab 配置（8083 web / 2222 SSH）
├── jenkins/
│   ├── Dockerfile                Jenkins + docker CLI
│   ├── docker-compose.yml        挂载 docker.sock，固定 IP 与别名
│   └── job.xml                  流水线任务定义（四步构建脚本）
├── harbor/harbor.yml.example     Harbor 配置样例（密码已脱敏）
├── scripts/disable-csrf.groovy   Jenkins 初始化脚本
└── app/                          被部署的 Spring Boot 应用构建文件
```

## 复现方式

```bash
# 1. 创建网络
docker network create --driver bridge --subnet 172.28.0.0/24 --gateway 172.28.0.1 cicd-net

# 2. 启动三套服务（各自目录下）
docker compose up -d            # gitlab / jenkins
docker run ... goharbor 安装包   # harbor 需先解压离线包并执行 prepare

# 3. 导入 Jenkins 任务
cp jenkins/job.xml ~/jenkins_home/jobs/springboot-cicd/config.xml

# 4. 在 GitLab 配置 Webhook，push 代码验证自动部署
```

> 注意：本项目是**单机实验环境**，密码与令牌在上传时已脱敏为占位符。
