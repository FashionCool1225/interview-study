# Docker & Kubernetes - 面试知识卡片

> 面向3年经验Java开发者 | 中型互联网公司面试准备

---

## 一、Docker

### Q1：Docker镜像的分层结构是怎样的？

**答：** Docker镜像采用**联合文件系统（UnionFS）**的分层结构：

**分层原理：** 镜像由多个只读层（Layer）叠加而成，每一层对应Dockerfile中的一条指令（如RUN、COPY、ADD等）。层与层之间通过UnionFS叠加，最终呈现为一个统一的文件系统视图。

**具体结构：**
- **最底层：** BootFS（引导文件系统），包含bootloader和kernel，由宿主机提供，镜像不存储这一层。
- **第二层：** RootFS（根文件系统），即基础镜像（如ubuntu、alpine）。
- **上层：** 每一层Dockerfile指令生成的只读层。
- **最顶层（运行时）：** 容器启动时在最上层添加一个可写层（Container Layer），容器的所有写操作都在这一层。

**分层的优势：**
- **镜像共享：** 多个镜像可以共享相同的基础层，节省磁盘空间。
- **构建高效：** 构建时利用缓存，未变化的层直接使用缓存，加速构建过程。
- **拉取快速：** 拉取镜像时只下载缺失的层，已有的层复用。

**查看镜像层级：** 使用`docker history <image>`可以查看镜像的每一层及其大小。

---

### Q2：Dockerfile常用指令有哪些？

**答：**

| 指令 | 作用 | 说明 |
|------|------|------|
| `FROM` | 指定基础镜像 | 每个Dockerfile必须以FROM开头，如`FROM openjdk:17-jdk-alpine` |
| `MAINTAINER`/`LABEL` | 指定维护者信息 | LABEL更推荐，如`LABEL maintainer="xxx@xxx.com"` |
| `RUN` | 执行命令并生成新层 | 构建时执行，如`RUN apt-get update && apt-get install -y curl` |
| `COPY` | 复制文件到镜像 | 从构建上下文复制，如`COPY target/app.jar /app/` |
| `ADD` | 复制文件到镜像 | 类似COPY，但支持自动解压tar包和下载URL |
| `WORKDIR` | 设置工作目录 | 后续指令的工作目录，如`WORKDIR /app` |
| `ENV` | 设置环境变量 | 如`ENV JAVA_OPTS="-Xmx512m"` |
| `EXPOSE` | 声明端口 | 只是文档说明，不实际映射端口，如`EXPOSE 8080` |
| `VOLUME` | 声明匿名卷 | 如`VOLUME /data`，运行时可挂载 |
| `CMD` | 容器启动命令 | 可被`docker run`参数覆盖，如`CMD ["java","-jar","app.jar"]` |
| `ENTRYPOINT` | 容器启动入口 | 不会被覆盖，而是将`docker run`参数追加到后面 |
| `ARG` | 构建参数 | 仅在构建时可用，如`ARG VERSION=1.0` |

**CMD vs ENTRYPOINT的区别：** CMD的默认命令可以被`docker run`后面的参数覆盖；ENTRYPOINT不会被覆盖，`docker run`的参数会作为ENTRYPOINT命令的附加参数。通常用ENTRYPOINT指定主程序，用CMD指定默认参数。

---

### Q3：Docker Compose的作用是什么？

**答：** Docker Compose是一个用于**定义和运行多容器Docker应用**的工具，通过一个YAML文件（`docker-compose.yml`）来描述所有服务及其依赖关系。

**核心作用：**
- **一键编排多容器：** 一个命令`docker-compose up`就能同时启动应用、数据库、Redis、MQ等多个服务，不用逐个手动启动。
- **环境一致性：** 开发、测试、生产使用相同的Compose文件，保证环境一致，避免"在我机器上能跑"的问题。
- **服务依赖管理：** 通过`depends_on`配置服务启动顺序，如应用服务依赖数据库先启动。
- **网络自动创建：** Compose自动创建一个专用网络，所有服务都在该网络内，可以通过服务名互相访问。

**常用命令：**
- `docker-compose up -d`：后台启动所有服务
- `docker-compose down`：停止并删除所有容器和网络
- `docker-compose logs -f`：查看日志
- `docker-compose ps`：查看服务状态
- `docker-compose restart <service>`：重启指定服务

**典型场景：** 本地开发环境搭建（Spring Boot + MySQL + Redis + RocketMQ），一键启动完整依赖。

---

### Q4：Docker的网络模式有哪些？

**答：** Docker提供四种主要的网络模式：

**bridge模式（默认）：**
- 容器连接到Docker创建的虚拟网桥（docker0）上
- 容器拥有独立的网络命名空间和IP地址
- 容器之间可以通过IP互相通信，与宿主机通过端口映射（`-p`）通信
- 最常用的模式，适合大多数场景

**host模式（`--network host`）：**
- 容器与宿主机共享网络命名空间
- 容器没有独立的IP，直接使用宿主机的网络栈
- 容器内的端口直接暴露在宿主机上，不需要端口映射
- 网络性能最好，但端口可能冲突，Linux支持，Mac/Windows不支持

**none模式（`--network none`）：**
- 容器没有网络连接，只有lo回环接口
- 需要手动配置网络，适合对安全隔离要求极高的场景

**container模式（`--network container:<name>`）：**
- 新容器与已存在的容器共享网络命名空间
- 两个容器使用相同的IP和端口空间，通过localhost通信
- 适合sidecar模式，如主容器+日志采集容器共享网络

---

### Q5：如何限制Docker容器的资源使用？

**答：** Docker提供了对CPU、内存、磁盘IO等资源的限制能力：

**内存限制：**
- `--memory` 或 `-m`：限制容器最大内存使用量，如`-m 512m`
- `--memory-swap`：限制内存+swap总量，设为与`-m`相同值则禁用swap
- 超出限制时，容器进程可能被OOM Killer杀掉

**CPU限制：**
- `--cpus`：限制容器可用的CPU数量，如`--cpus 1.5`表示最多用1.5个CPU核心
- `--cpu-shares`：设置CPU权重（默认1024），用于多容器竞争时按比例分配CPU时间
- `--cpuset-cpus`：绑定容器到指定的CPU核心，如`--cpuset-cpus="0,1"`

**磁盘IO限制：**
- `--device-read-bps`/`--device-write-bps`：限制读写速率
- `--device-read-iops`/`--device-write-iops`：限制读写IOPS

**在docker-compose.yml中配置：**
```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G
        reservations:
          cpus: '1'
          memory: 512M
```

**实际意义：** 生产环境中一定要设置资源限制，防止某个容器资源泄漏导致整个宿主机被拖垮（"吵闹的邻居"问题）。

---

### Q6：Docker镜像优化有哪些实践？

**答：**

**1. 选择精简基础镜像：** 使用`alpine`或`distroless`镜像代替完整OS镜像。如`openjdk:17-jdk-alpine`约100MB，而完整版可能超过600MB。

**2. 多阶段构建（Multi-stage Build）：**
```dockerfile
# 构建阶段
FROM maven:3.8-openjdk-17 AS builder
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests

# 运行阶段
FROM openjdk:17-jdk-alpine
COPY --from=builder /app/target/app.jar /app/
CMD ["java", "-jar", "/app/app.jar"]
```
最终镜像只包含运行产物，不包含Maven等构建工具，大幅减小镜像体积。

**3. 合并RUN指令：** 每条RUN都会生成一个新层，合并可以减少层数。
```dockerfile
# 不好（3层）
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# 好（1层）
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
```

**4. 善用构建缓存：** 将不常变化的指令放在前面（如安装依赖），频繁变化的指令放在后面（如COPY源码）。

**5. 使用.dockerignore：** 排除不必要的文件（.git、node_modules、日志等），减小构建上下文。

---

## 二、Kubernetes

### Q7：K8s的核心组件有哪些？

**答：** K8s集群由**Master节点**（控制平面）和**Node节点**（工作平面）组成。

**Master节点组件：**

- **API Server：** K8s的"前台"，所有操作的统一入口。提供RESTful API，负责资源的增删改查和认证授权。所有组件之间的通信都通过API Server。
- **etcd：** 分布式键值存储，保存整个集群的状态数据（如Pod信息、ConfigMap、Secret等）。是K8s的"数据库"，需要定期备份。
- **Controller Manager：** 运行各种控制器（如Deployment Controller、Node Controller），负责维护集群的期望状态。例如Deployment Controller确保Pod副本数始终符合预期。
- **Scheduler：** 负责Pod调度，根据资源需求、亲和性规则、污点容忍等策略，将Pod分配到合适的Node上。

**Node节点组件：**

- **Kubelet：** Node上的"管家"，负责接收Master下发的指令，管理本节点上的Pod生命周期（创建、启动、停止容器）。
- **Kube-proxy：** 负责网络代理和负载均衡，维护Node上的网络规则（iptables/IPVS），实现Service的访问。
- **Container Runtime：** 容器运行时，负责实际运行容器。早期是Docker，现在推荐containerd或CRI-O。

---

### Q8：什么是Pod？它的生命周期是怎样的？

**答：** Pod是K8s中**最小的调度和管理单元**，一个Pod包含一个或多个紧密关联的容器。

**为什么需要Pod而不是直接管理容器？**
- 同一个Pod中的容器共享网络（相同IP、可以localhost通信）和存储（共享Volume）
- Pod是调度的原子单位，同一个Pod的容器总是在同一个Node上
- 典型场景：主容器 + Sidecar容器（如日志采集、服务网格代理）

**Pod生命周期：**

1. **Pending：** Pod已被API Server接受，但容器尚未全部就绪（如镜像拉取中、调度等待中）
2. **Running：** Pod已被调度到Node上，至少有一个容器正在运行
3. **Succeeded：** 所有容器正常退出且不会重启（常见于Job）
4. **Failed：** 至少有一个容器异常退出
5. **Unknown：** 无法获取Pod状态（通常是与Node通信失败）

**Pod内的容器状态：**
- **Waiting：** 等待中（拉取镜像、等待依赖）
- **Running：** 运行中
- **Terminated：** 已终止（正常退出或异常退出）

**重要概念 —— 重启策略（restartPolicy）：**
- `Always`：容器退出总是重启（默认，适用于Deployment）
- `OnFailure`：异常退出时才重启（适用于Job）
- `Never`：从不重启

---

### Q9：Deployment和StatefulSet有什么区别？

**答：**

**Deployment（无状态应用）：**
- 管理无状态应用，如Web服务、API服务
- Pod之间是对等的，可以互相替代
- Pod名称是随机的（如`app-5d8f7b6c4-x2k9p`）
- 扩缩容时任意创建/删除Pod
- 更新时支持滚动更新和回滚
- 通常配合Service使用ClusterIP或LoadBalancer

**StatefulSet（有状态应用）：**
- 管理有状态应用，如数据库（MySQL、Redis Cluster）、消息队列（Kafka）
- Pod有固定的、有序的名称（如`mysql-0`、`mysql-1`、`mysql-2`）
- Pod的启动和删除是有序的（0->1->2启动，2->1->0删除）
- 每个Pod绑定独立的持久化存储（PVC），Pod重建后仍然绑定原来的PVC
- 提供稳定的网络标识（Headless Service + Pod DNS：`pod-name.service-name.namespace.svc.cluster.local`）

**选型原则：**
- Web服务、微服务 -> Deployment
- 数据库、中间件集群 -> StatefulSet
- 一次性任务 -> Job/CronJob
- 常驻后台任务 -> DaemonSet（如日志采集、监控Agent）

---

### Q10：K8s Service有哪些类型？

**答：** Service是一组Pod的抽象和访问入口，通过Label Selector关联Pod。

**ClusterIP（默认）：**
- 分配一个集群内部虚拟IP
- 只能在集群内部访问
- 适合集群内部服务之间的调用（如订单服务调用库存服务）

**NodePort：**
- 在ClusterIP的基础上，在每个Node上开放一个固定端口（范围30000-32767）
- 外部可以通过`NodeIP:NodePort`访问服务
- 适合开发测试环境或不需要负载均衡器的场景

**LoadBalancer：**
- 在NodePort的基础上，请求云服务商创建一个外部负载均衡器
- 提供一个统一的外部IP和端口
- 适合生产环境暴露服务给外部访问
- 在裸金属环境需要配合MetalLB等工具使用

**ExternalName：**
- 将Service映射到一个外部域名（CNAME记录）
- 如将集群内的Service指向集群外的数据库地址
- 方便服务在集群内外迁移时保持访问地址不变

**补充 —— Headless Service：** 设置`clusterIP: None`，不分配虚拟IP，DNS直接返回Pod的IP列表。常用于StatefulSet，客户端自己做负载均衡。

---

### Q11：ConfigMap和Secret有什么区别？怎么用？

**答：** 两者都用于将配置与镜像解耦，区别在于存储的内容类型：

**ConfigMap：** 存储非敏感配置信息，以明文Key-Value对的形式存储。
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_URL: "jdbc:mysql://mysql:3306/mydb"
  LOG_LEVEL: "INFO"
  application.yml: |
    server:
      port: 8080
```

**Secret：** 存储敏感信息，内容经过Base64编码（注意：Base64不是加密，只是编码，安全性依赖etcd的访问控制和加密配置）。
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: cm9vdA==    # base64编码
  password: cGFzc3dvcmQ=
```

**使用方式（两者相同）：**
1. **环境变量注入：** 通过`envFrom`或`env.valueFrom`注入到容器中。
2. **文件挂载：** 通过Volume挂载为文件，Key变成文件名，Value变成文件内容。适合配置文件（如application.yml、nginx.conf）。

**注意事项：**
- Secret的Base64编码不是安全措施，生产环境应启用etcd加密（EncryptionConfiguration）或使用外部密钥管理（如Vault、AWS Secrets Manager）。
- ConfigMap/Secret更新后，使用环境变量方式的Pod需要重启才能生效；使用Volume挂载方式可以自动更新（有一定延迟）。

---

### Q12：什么是HPA？如何实现自动伸缩？

**答：** HPA（Horizontal Pod Autoscaler，水平Pod自动伸缩器）根据监控指标自动调整Pod副本数量。

**工作原理：**
1. HPA Controller定期（默认15秒）从Metrics Server获取Pod的指标数据
2. 将当前指标值与目标值对比，计算需要的副本数
3. 通过修改Deployment的`replicas`字段来扩缩容

**基本配置示例：**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```
上述配置表示：当CPU平均使用率超过70%或内存使用率超过80%时扩容，最多扩到10个副本，最少保持2个。

**支持的指标类型：**
- **Resource：** CPU、内存使用率（需要Metrics Server）
- **Pods：** 自定义的Pod级别指标（如QPS）
- **Object：** 关联对象的指标（如Ingress的请求数）
- **External：** 外部指标（如MQ消息积压量）

**注意事项：**
- 使用CPU/内存指标需要先为容器设置`resources.requests`，HPA基于requests计算使用率
- 缩容有冷却时间（默认5分钟），防止频繁缩容
- 配合PDB（PodDisruptionBudget）保证缩容时最少可用副本数

---

### Q13：K8s的服务发现和负载均衡是怎么实现的？

**答：**

**服务发现：** K8s通过**Service + DNS**实现服务发现。
- 创建Service后，K8s自动分配一个ClusterIP（虚拟IP）
- CoreDNS会自动为Service注册DNS记录，格式为`<service-name>.<namespace>.svc.cluster.local`
- Pod内可以通过服务名直接访问其他服务，如`http://order-service:8080/api`
- Service通过Label Selector关联后端Pod，当Pod变化时，Endpoints Controller自动更新Service关联的Pod列表

**负载均衡：**
- **集群内部：** Kube-proxy在每个Node上维护iptables或IPVS规则。当请求发往ClusterIP时，iptables/IPVS规则将流量分发到后端的Pod。
  - **iptables模式：** 随机分发，规则数随Service增多而线性增长，性能在大规模集群下下降
  - **IPVS模式：** 支持多种负载均衡算法（轮询、最少连接、源地址哈希等），性能更好，适合大规模集群
- **外部流量：** 通过NodePort或LoadBalancer类型的Service暴露，外部负载均衡器将流量分发到各Node的NodePort上

**Session亲和性：** 可以通过`service.spec.sessionAffinity: ClientIP`实现基于客户端IP的会话保持。

---

### Q14：K8s的探针（Probe）有哪些类型？

**答：** K8s提供三种探针来监控容器健康状态：

**Liveness Probe（存活探针）：**
- 判断容器是否"活着"
- 探测失败时，K8s会**重启容器**
- 适用于检测死锁、进程假死等场景
- 示例：`GET /healthz` 返回200表示存活

**Readiness Probe（就绪探针）：**
- 判断容器是否"准备好接收流量"
- 探测失败时，K8s会将Pod从Service的Endpoints中**移除**，不重启容器
- 适用于应用启动初始化、临时过载等场景
- 示例：`GET /ready` 返回200表示可以接收请求

**Startup Probe（启动探针）：**
- 判断容器是否已经启动完成
- 在Startup Probe成功之前，Liveness和Readiness Probe**不会生效**
- 适用于启动慢的应用（如Java应用加载大量类、初始化缓存），避免启动期间被Liveness Probe误杀
- 可以设置较长的`failureThreshold * periodSeconds`来容忍慢启动

**探针配置示例：**
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30    # 启动后等30秒再开始探测
  periodSeconds: 10           # 每10秒探测一次
  timeoutSeconds: 3           # 超时时间3秒
  failureThreshold: 3         # 连续3次失败才认为不健康
```

**探测方式：** httpGet（HTTP请求）、tcpSocket（TCP连接）、exec（执行命令，如`cat /tmp/healthy`）。

**最佳实践：** Spring Boot Actuator提供了`/actuator/health/liveness`和`/actuator/health/readiness`两个端点，分别对应Liveness和Readiness探针。

---

### Q15：Ingress的作用是什么？

**答：** Ingress是K8s中**管理外部HTTP/HTTPS流量进入集群**的资源对象，相当于集群的"七层网关"。

**为什么需要Ingress？**
- Service的LoadBalancer类型虽然能暴露服务，但每个Service都需要一个独立的负载均衡器，成本高
- Ingress可以将多个Service统一在一个域名下，通过URL路径路由到不同的Service，只需要一个入口

**功能：**
- **域名路由：** `api.example.com` -> API服务，`web.example.com` -> 前端服务
- **路径路由：** `/api/v1/users` -> 用户服务，`/api/v1/orders` -> 订单服务
- **TLS终止：** 统一管理HTTPS证书
- **负载均衡、限流、认证**等高级功能

**Ingress示例：**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx    # 需要Ingress Controller
  tls:
  - hosts:
    - api.example.com
    secretName: tls-secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 8080
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 8080
```

**Ingress Controller：** Ingress只是规则定义，需要Ingress Controller来执行。常见的有Nginx Ingress Controller、Traefik、Kong等。Ingress Controller本身也是以Pod形式运行在K8s集群中。

---

## 三、DevOps实践

### Q16：如何设计CI/CD流水线？

**答：** 一个典型的Java微服务CI/CD流水线包括以下阶段：

**CI（持续集成）阶段：**
1. **代码提交：** 开发者推送代码到Git仓库（GitLab/GitHub），触发Webhook
2. **代码检查：** SonarQube静态代码分析，检查代码规范、潜在Bug、安全漏洞
3. **单元测试：** 执行JUnit/Mockito测试，生成覆盖率报告（要求覆盖率达标）
4. **构建打包：** Maven/Gradle编译打包，生成JAR包
5. **构建镜像：** 通过Dockerfile构建Docker镜像，推送到镜像仓库（Harbor/Docker Hub）

**CD（持续部署）阶段：**
6. **部署到测试环境：** 更新K8s Deployment的镜像版本，自动部署到测试环境
7. **集成测试/自动化测试：** 执行接口自动化测试
8. **部署到预发布环境：** 模拟生产环境验证
9. **部署到生产环境：** 人工审批后，滚动更新或金丝雀发布到生产

**工具链：**
- 代码管理：GitLab / GitHub
- CI工具：Jenkins / GitLab CI / GitHub Actions
- 镜像仓库：Harbor（私有）
- 部署工具：Helm / ArgoCD（GitOps模式）
- 监控告警：Prometheus + Grafana

**GitOps模式（推荐）：** 使用ArgoCD监听Git仓库中K8s部署清单的变化，自动同步到集群。部署配置和代码一样版本化管理，变更可追溯、可回滚。

---

### Q17：蓝绿部署和金丝雀发布有什么区别？

**答：**

**蓝绿部署（Blue-Green Deployment）：**

原理：同时维护两套完全相同的生产环境（蓝环境和绿环境），任何时刻只有一套在处理真实流量。

流程：
1. 当前蓝环境在运行v1版本，处理所有流量
2. 将v2版本部署到绿环境
3. 在绿环境上做验证测试
4. 验证通过后，通过负载均衡器将流量从蓝环境**一次性切换**到绿环境
5. 蓝环境保留一段时间作为回滚备份，确认无问题后回收

优点：切换快速、回滚简单（切回蓝环境即可）
缺点：需要双倍资源，两套环境同时运行

**金丝雀发布（Canary Release / 灰度发布）：**

原理：先让**一小部分用户**使用新版本，逐步扩大范围，最终全量上线。

流程：
1. 当前v1版本处理100%流量
2. 部署v2版本，先导入5%流量到v2
3. 观察v2的监控指标（错误率、延迟、业务指标）
4. 无异常则逐步增加到20%、50%、100%
5. 发现问题立即回滚（将流量切回v1）

K8s中的实现方式：
- 使用两个Deployment（v1和v2），通过Service的权重或Ingress的注解控制流量比例
- 使用Istio等Service Mesh实现更精细的流量控制（按Header、Cookie、用户ID等路由）

优点：影响范围小、风险可控、可以A/B测试
缺点：发布周期长、需要完善的监控系统

**对比总结：**

| 维度 | 蓝绿部署 | 金丝雀发布 |
|------|---------|-----------|
| 资源开销 | 高（双份） | 低（增量少量Pod） |
| 切换速度 | 快（一次性） | 慢（渐进式） |
| 风险 | 中（全量切换） | 低（小范围验证） |
| 回滚速度 | 快（切回即可） | 快（切回流量） |
| 适用场景 | 变更风险高、需要快速回滚 | 变更风险可控、逐步验证 |

---

### Q18：K8s中如何实现滚动更新和回滚？

**答：**

**滚动更新（Rolling Update）：** Deployment默认的更新策略。逐步用新版本Pod替换旧版本Pod，保证服务不中断。

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 更新时最多多出1个Pod（可以是百分比，如25%）
      maxUnavailable: 0     # 更新时不允许有不可用的Pod
```

- `maxSurge`：超出期望副本数的最大Pod数量。如设为1，Replicas=3时，更新期间最多4个Pod同时存在。
- `maxUnavailable`：允许不可用的最大Pod数量。设为0表示保证所有Pod都可用（先启新再停旧）。

**回滚：**
- `kubectl rollout undo deployment/my-app`：回滚到上一个版本
- `kubectl rollout history deployment/my-app`：查看历史版本
- `kubectl rollout undo deployment/my-app --to-revision=3`：回滚到指定版本

**Deployment保留历史版本：** 通过`revisionHistoryLimit`控制保留的历史ReplicaSet数量（默认10），设为0则无法回滚。

---

### Q19：Docker和虚拟机有什么区别？

**答：**

| 维度 | Docker容器 | 虚拟机 |
|------|-----------|--------|
| **虚拟化级别** | 操作系统级虚拟化，共享宿主机内核 | 硬件级虚拟化，每个VM有独立内核 |
| **启动速度** | 秒级（启动进程） | 分钟级（启动完整OS） |
| **资源占用** | MB级别（只包含应用和依赖） | GB级别（包含完整OS） |
| **性能** | 接近原生性能 | 有一定性能损耗（Hypervisor层） |
| **隔离性** | 进程级隔离（cgroup+namespace），较弱 | 完全隔离，安全性更高 |
| **密度** | 单机可运行上百个容器 | 单机通常几十个VM |
| **适用场景** | 微服务、CI/CD、快速部署 | 强隔离需求、多租户、运行不同OS |

**关键理解：** 容器不是轻量级虚拟机。容器本质上是宿主机上的一个受限进程，通过Linux的Namespace实现隔离（PID、网络、文件系统等），通过Cgroup实现资源限制。它共享宿主机内核，所以不能运行不同操作系统的容器。

---

### Q20：K8s中Pod调度策略有哪些？

**答：** Scheduler负责将Pod调度到合适的Node上，可以通过以下策略影响调度决策：

**nodeSelector：** 最简单的调度方式，通过Label匹配Node。
```yaml
nodeSelector:
  disk: ssd
  region: cn-beijing
```

**亲和性（Affinity）：**

- **Node亲和性：** 更灵活的nodeSelector，支持`In`、`NotIn`、`Exists`等操作符，支持软性偏好（preferred）和硬性要求（required）。
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:  # 硬性要求
      nodeSelectorTerms:
      - matchExpressions:
        - key: zone
          operator: In
          values: ["cn-beijing-a", "cn-beijing-b"]
```

- **Pod亲和性（PodAffinity）：** 将Pod调度到与特定Pod相同的Node或同一可用区（如Web服务和缓存服务部署在一起减少延迟）。

- **Pod反亲和性（PodAntiAffinity）：** 将Pod调度到不同的Node上（如Redis集群的不同实例分散部署，提高可用性）。

**污点和容忍（Taints & Tolerations）：**
- Node设置污点（Taint）：`kubectl taint nodes node1 key=value:NoSchedule`
- Pod设置容忍（Toleration）：只有容忍该污点的Pod才能调度到该Node
- 常用于Master节点不允许调度普通Pod、专用节点（GPU节点）等场景

**优先级调度（PriorityClass）：** 为关键Pod设置高优先级，资源不足时驱逐低优先级Pod来腾出资源。

---

## 附录：面试高频追问

### Q：你实际工作中是怎么使用Docker和K8s的？

**答：** （参考回答框架）
- 本地开发使用Docker Compose搭建依赖环境（MySQL、Redis、RocketMQ），通过Dockerfile构建应用镜像
- CI/CD使用GitLab CI，流水线包括：代码检查 -> 单元测试 -> Maven构建 -> Docker镜像构建推送 -> Helm更新部署清单 -> ArgoCD自动同步到K8s集群
- 生产环境使用Deployment管理无状态服务，StatefulSet管理Redis Cluster，HPA根据CPU使用率自动扩缩容
- 使用ConfigMap管理应用配置，Secret管理数据库密码
- 通过Ingress暴露服务，Nginx Ingress Controller做七层负载均衡
- 配置了Liveness和Readiness探针，保证服务健康检查和优雅上下线

### Q：K8s中Pod一直Pending怎么排查？

**答：** 排查步骤：
1. `kubectl describe pod <pod-name>`查看Events，通常能看到调度失败原因
2. 常见原因：资源不足（CPU/内存不满足requests）、没有匹配的Node（nodeSelector/affinity不匹配）、PVC绑定失败、污点不容忍
3. `kubectl get events`查看集群事件
4. `kubectl get nodes -o wide`检查Node状态是否正常
5. 检查是否有资源配额（ResourceQuota）限制
