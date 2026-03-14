将 Spark C++ 任务从 Hadoop 迁移到 Kubernetes (K8s) 是一个系统工程。核心思路是将 C++ 逻辑封装在 Docker 镜像中，通过 Spark 的 **Kubernetes 调度器** 自动创建计算节点（Pod）。

以下是手把手的配置指南：

---

## 第一步：环境准备（Linux & K8s Cluster）

在开始 Spark 之前，你的 Linux 集群需要具备以下基础：

1. **容器运行时**：每台 Linux 节点必须安装 `containerd` 或 `Docker`。
2. **K8s 集群**：建议使用 `v1.24+` 版本。确保你有 `kubectl` 访问权限。
3. **镜像仓库**：你需要一个私有仓库（如 Harbor 或阿里云 ACR），因为 K8s 节点需要拉取你打包好的 C++ 环境镜像。

---

## 第二步：构建包含 C++ 环境的 Docker 镜像

这是最关键的一步。你需要把 Spark 的运行环境和你的 C++ 二进制文件打在一起。

### 1. 编写 Dockerfile

```dockerfile
# 使用 Spark 官方基础镜像（通常带 Java 环境）
FROM apache/spark:3.5.0-scala2.12-java11-python3-ubuntu

USER root
# 安装 C++ 运行所需的库（如 glibc, libstdc++, 或你的特定依赖）
RUN apt-get update && apt-get install -y \
    libstdc++6 \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# 将你的 C++ 编译产物放入镜像
COPY ./my_cpp_worker /opt/spark/work-dir/my_cpp_worker
RUN chmod +x /opt/spark/work-dir/my_cpp_worker

# 如果使用 JNI，把 .so 文件放到库路径
COPY ./libnative_logic.so /usr/lib/

USER spark

```

### 2. 构建并推送

```bash
docker build -t my-registry.com/spark-cpp-app:v1 .
docker push my-registry.com/spark-cpp-app:v1

```

---

## 第三步：Spark on K8s 的核心参数配置

当你运行 `spark-submit` 时，以下参数决定了系统如何运作：

### 1. 资源边界参数（决定稳定性）

* **`spark.executor.instances`**: 启动多少个 Pod（Worker）。
* **`spark.kubernetes.container.image`**: 指定你刚推送到仓库的镜像。
* **`spark.executor.memoryOverhead`**: **(重点)** 必须预留足够内存给 C++。如果 C++ 任务分配了 4GB 内存，JVM 分配了 2GB，那么这个值至少设为 `4096`。

### 2. 调度与安全参数

* **`spark.kubernetes.authenticate.driver.serviceAccountName`**: K8s 需要一个有权限创建 Pod 的账号（通常叫 `spark-sa`）。
* **`spark.kubernetes.namespace`**: 任务运行的隔离空间。

---

## 第四步：提交任务的命令示例

你可以直接在 Linux 终端执行：

```bash
spark-submit \
    --master k8s://https://<K8s-API-Server-IP>:6443 \
    --deploy-mode cluster \
    --name spark-cpp-job \
    --conf spark.kubernetes.container.image=my-registry.com/spark-cpp-app:v1 \
    --conf spark.kubernetes.namespace=spark-jobs \
    --conf spark.executor.instances=5 \
    --conf spark.executor.cores=2 \
    --conf spark.executor.memory=2g \
    --conf spark.executor.memoryOverhead=4096 \
    --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark-sa \
    local:///opt/spark/work-dir/your-main-app.jar

```

---

## 第五步：你需要做的关键决策 (Decision List)

在配置过程中，你需要根据业务压力决定以下几点：

### 1. 网络模式：Overlay 还是 HostNetwork？

* **默认 (Overlay)**: 安全，但 C++ 频繁通信会有 5%-10% 延迟。
* **HostNetwork**: 如果你的 C++ 任务对网络延迟极其敏感，可以开启 `spark.kubernetes.driver.pod.featureSteps` 相关配置使用宿主机网络，性能最高。

### 2. 存储选择：如何读数？

* **直接读 HDFS**: 在镜像里配置 `core-site.xml`。
* **读 S3/OSS**: 推荐做法。配置 `spark.hadoop.fs.s3a.access.key` 等参数。

### 3. C++ 崩溃处理

* **`spark.task.maxFailures`**: 默认 4 次。如果 C++ 代码不够稳，建议调大这个值，防止一个 Task 崩溃导致整个 Job 失败。

---

### 下一步建议

目前 K8s 社区更推荐使用 **Spark Operator**（通过 YAML 文件管理任务，而不是命令行）。**你想尝试编写一个 SparkApplication 的 YAML 配置文件吗？这会让你的 CI/CD 流程更加自动化。**
