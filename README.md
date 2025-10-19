# Bifrost 架构设计方案

## 一、架构设计目标

该平台旨在构建一个 **跨语言、可扩展、解耦的远程命令执行平台**，用于支持多类型设备（云主机、容器、网络设备等）的自动化命令执行与结果回传。

目标包括：
- 支持多种连接与执行方式（SSH、Ansible、Netmiko、API、后续可扩展）
- 架构解耦、易于维护与扩展
- 可混合语言实现（Go + Python）
- 可插拔的执行层设计
- 统一的上下文协议与审计能力
- 可灵活集成后续的日志、存储、监控、认证等中间件

---

## 二、总体架构概览

### 架构层级

```
+---------------------------------------------------------+
|                      Web / API Layer                    |
|----------------------------------------------------------|
|   REST / gRPC 接口（任务提交、状态查询、结果回调）         |
+----------------------------------------------------------+
|                      Gateway Service (Go)                |
|----------------------------------------------------------|
|  - 请求解析与认证                                         |
|  - 执行调度与上下文构建                                   |
|  - gRPC 调用 Executor                                     |
|  - 状态管理与日志回传接口                                 |
+----------------------------------------------------------+
|                      Executor Layer                      |
|----------------------------------------------------------|
|  - Ansible Executor (Python)                             |
|  - SSH Executor (Python / Go)                             |
|  - Netmiko Executor (Python)                             |
|  - 未来扩展：API Executor / Script Executor / Plugin ... |
+----------------------------------------------------------+
|                     Target Environment                   |
|----------------------------------------------------------|
|  - Linux / Docker / Network Devices / Cloud VM           |
+----------------------------------------------------------+
```

### 服务间通信
- **Gateway 与 Executor** 之间通过 **gRPC + Protobuf** 通信；
- **Executor** 独立运行，可分布式部署；
- **Protobuf** 定义上下文结构（执行请求与执行结果）；
- **日志 / 状态 / 审计** 模块可后续以接口方式接入。

---

## 三、模块职责划分

| 模块 | 主要职责 | 技术实现建议 |
|------|-----------|---------------|
| **Gateway Service (Go)** | 提供外部接口、接收执行请求、构建上下文、分发任务、聚合结果、对接日志系统 | Go + gRPC + Protobuf |
| **Executor Registry** | 管理可用执行器（本地或远程），执行器发现、健康检查、负载分配 | Go + 内存注册表（后续可扩展为 etcd/Consul） |
| **Executor (Python/Go)** | 负责具体命令执行逻辑，与底层工具（Ansible、Netmiko、SSH）交互 | Python / Go，独立 gRPC Server |
| **Protobuf Schema** | 定义 Gateway 与 Executor 通信协议（上下文、输入、输出、元信息） | Protobuf |
| **Logging / Auditing (接口层)** | 定义日志与审计接口，MVP 阶段可使用本地文件，后续对接 ELK / Prometheus | Go interface + optional adapter |
| **Plugin Framework** | 允许第三方执行器注册（通过 gRPC 协议） | Go 插件接口机制 or Registry 模式 |

---

## 四、仓库结构设计

```
infra-executor-platform/
├── api/                        # Protobuf 定义与生成文件
│   └── executor.proto
│
├── cmd/                        # 各个服务的启动入口
│   ├── gateway/                # Gateway 主服务入口
│   └── executors/              # 各执行器启动脚本
│
├── pkg/                        # Go 公共包
│   ├── gateway/                # Gateway 核心逻辑（调度/状态/上下文管理）
│   ├── registry/               # 执行器注册与发现
│   ├── executor_stub/          # gRPC 客户端生成代码
│   ├── common/                 # 通用工具（配置、日志、错误封装）
│   └── interfaces/             # 日志、审计、存储接口定义
│
├── executors/                  # 各语言实现的执行器模块
│   ├── ansible_executor/       # Python: 调用 ansible-runner
│   ├── ssh_executor/           # Python: 使用 paramiko/asyncssh
│   ├── netmiko_executor/       # Python: 网络设备控制
│   ├── go_executor/            # Go: 原生 SSH / 命令执行实现
│   └── mock_executor/          # 测试与CI用
│
├── scripts/                    # 构建与开发辅助脚本
│   ├── build_protos.sh
│   ├── run_gateway.sh
│   └── run_executor.sh
│
├── docs/                       # 项目设计文档、接口说明
│   └── architecture.md
│
└── Makefile / go.mod / requirements.txt
```

---

## 五、服务间通信协议（Protobuf）

```protobuf
syntax = "proto3";

package executor;

service ExecutorService {
  rpc Execute (ExecutionRequest) returns (ExecutionResult);
}

message ExecutionRequest {
  string task_id = 1;
  string executor_type = 2;       // "ansible", "ssh", "netmiko"
  map<string, string> metadata = 3; // 认证、变量、上下文信息
  string payload = 4;             // 命令或 playbook 内容
}

message ExecutionResult {
  string task_id = 1;
  int32 exit_code = 2;
  string stdout = 3;
  string stderr = 4;
  string metadata = 5;
}
```

---

## 六、插件化设计思路

- 每个 Executor 独立实现 `ExecutorService` 接口；
- Gateway 通过注册表发现可用执行器；
- 执行器可动态注册（通过启动参数或服务发现）；
- 执行器实现只需遵循 protobuf 协议即可，无需修改 Gateway；
- 插件可使用不同语言实现，只需兼容 gRPC。

---

## 七、MVP 阶段实施建议

**第一阶段（MVP）：**
- 构建 `gateway` 服务（Go）
- 定义并实现基础 `protobuf` 协议
- 实现 `SSH Executor`（Python）
- Gateway 通过 gRPC 调用执行器执行命令
- 结果回传与打印日志（本地文件方式）
- 编写基础测试与 CLI 工具验证执行流

**第二阶段：**
- 扩展 Ansible 与 Netmiko 执行器
- 实现执行器注册表机制
- 引入日志与状态存储接口
- 实现异步任务回调（Webhook / 队列）

**第三阶段：**
- 插件注册机制（动态加载执行器）
- 跳板机多级代理支持
- 日志接入 ELK / Prometheus
- 审计与重放能力

---

## 八、总结

该架构通过 **Go + gRPC + Protobuf** 构建核心调度与通信层，以 **Python 执行器** 兼容现有成熟生态，实现了：
- 扩展性与解耦性；
- 多语言支持；
- 插件化执行层；
- 可持续演进的企业级架构。

未来可平滑演进为统一的 **分布式命令执行与任务编排平台**，支持多租户、多执行方式及审计追踪能力。

