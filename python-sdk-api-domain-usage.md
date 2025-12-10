# Python SDK - API域名与Domain域名使用分析

## 概述

E2B Python SDK中使用了两种不同的域名配置：
1. **API域名** (`api_url`) - 用于E2B控制平面API调用
2. **Domain域名** (`sandbox_domain`) - 用于连接到运行中的沙箱实例

## 一、使用API域名的功能

API域名默认为 `https://api.{domain}`（其中domain默认为`e2b.app`），用于与E2B控制平面API进行通信。

### 1.1 沙箱管理操作（通过API）

**文件位置**: `e2b/sandbox_async/sandbox_api.py` 和 `e2b/sandbox_sync/sandbox_api.py`

使用API域名的功能：
- **创建沙箱** (`post_sandboxes`)
  - 方法: `AsyncSandbox.create()` 或 `Sandbox.create()`
  - API调用: `POST /sandboxes`
  
- **获取沙箱信息** (`get_sandboxes_sandbox_id`)
  - 方法: `AsyncSandbox.get_info()` 或 `Sandbox.get_info()`
  - API调用: `GET /sandboxes/{sandbox_id}`
  
- **删除/终止沙箱** (`delete_sandboxes_sandbox_id`)
  - 方法: `AsyncSandbox.kill()` 或 `Sandbox.kill()`
  - API调用: `DELETE /sandboxes/{sandbox_id}`
  
- **列出所有沙箱** (`get_v2_sandboxes`)
  - 方法: `AsyncSandbox.list()` 或 `Sandbox.list()`
  - API调用: `GET /v2/sandboxes`
  
- **沙箱连接** (`post_sandboxes_sandbox_id_connect`)
  - 方法: `AsyncSandbox.reconnect()` 或 `Sandbox.reconnect()`
  - API调用: `POST /sandboxes/{sandbox_id}/connect`
  
- **暂停沙箱** (`post_sandboxes_sandbox_id_pause`)
  - 方法: `AsyncSandbox.pause()` 或 `Sandbox.pause()`
  - API调用: `POST /sandboxes/{sandbox_id}/pause`
  
- **设置超时时间** (`post_sandboxes_sandbox_id_timeout`)
  - 方法: `AsyncSandbox.set_timeout()` 或 `Sandbox.set_timeout()`
  - API调用: `POST /sandboxes/{sandbox_id}/timeout`
  
- **获取指标** (`get_sandboxes_sandbox_id_metrics`)
  - 方法: `AsyncSandbox.get_metrics()` 或 `Sandbox.get_metrics()`
  - API调用: `GET /sandboxes/{sandbox_id}/metrics`

### 1.2 模板管理操作（通过API）

**文件位置**: `e2b/template_async/build_api.py` 和 `e2b/template_sync/build_api.py`

使用API域名的功能：
- **创建模板构建** (`post_v2_templates` 或 `post_v3_templates`)
  - 方法: `AsyncTemplate.build()` 或 `Template.build()`
  - API调用: `POST /v2/templates` 或 `POST /v3/templates`
  
- **获取文件上传链接** (`get_templates_template_id_files_hash`)
  - 用于上传Dockerfile和其他构建文件
  - API调用: `GET /templates/{template_id}/files/{hash}`
  
- **触发构建** (`post_v_2_templates_template_id_builds_build_id`)
  - 方法: 在`AsyncTemplate.build()`流程中
  - API调用: `POST /v2/templates/{template_id}/builds/{build_id}`
  
- **获取构建状态** (`get_templates_template_id_builds_build_id_status`)
  - 方法: 在`AsyncTemplate.build()`流程中轮询构建状态
  - API调用: `GET /templates/{template_id}/builds/{build_id}/status`
  
- **列出模板** (`get_templates`)
  - API调用: `GET /templates`
  
- **获取模板详情** (`get_templates_template_id`)
  - API调用: `GET /templates/{template_id}`
  
- **更新模板** (`patch_templates_template_id`)
  - API调用: `PATCH /templates/{template_id}`

### 1.3 配置位置

**文件**: `e2b/connection_config.py`

```python
# API URL的配置逻辑（第114-118行）
self.api_url = (
    api_url
    or ConnectionConfig._api_url()  # 从E2B_API_URL环境变量
    or ("http://localhost:3000" if self.debug else f"https://api.{self.domain}")
)
```

**文件**: `e2b/api/__init__.py`

```python
# API客户端创建（第128行）
super().__init__(
    base_url=config.api_url,  # 使用api_url
    ...
)
```

## 二、使用Domain域名的功能

Domain域名默认为 `e2b.app`，用于连接到具体的沙箱实例。沙箱的实际URL格式为：`https://{port}-{sandbox_id}.{sandbox_domain}`

### 2.1 沙箱内部操作（通过envd API）

**文件位置**: `e2b/sandbox/main.py`, `e2b/sandbox_async/main.py`

所有与运行中的沙箱直接交互的功能都使用domain域名：

#### 文件系统操作
**文件位置**: `e2b/sandbox_async/filesystem/filesystem.py` 和 `e2b/sandbox_sync/filesystem/filesystem.py`

- **读取文件** (`files.read()`)
  - 连接: `https://{envd_port}-{sandbox_id}.{sandbox_domain}`
  - 端点: `GET /files`
  
- **写入文件** (`files.write()`)
  - 端点: `POST /files`
  
- **列出目录** (`files.list()`)
  - 端点: `GET /files` (带路径参数)
  
- **创建目录** (`files.make_dir()`)
  - 端点: `POST /files`
  
- **删除文件/目录** (`files.remove()`)
  - 端点: `DELETE /files`
  
- **文件监视** (`files.watch_dir()`)
  - 使用WebSocket连接到沙箱
  
- **文件上传URL** (`upload_url()`)
  - 方法位置: `e2b/sandbox/main.py` 第154-189行
  - 返回沙箱文件上传的完整URL
  
- **文件下载URL** (`download_url()`)
  - 方法位置: `e2b/sandbox/main.py` 第119-152行
  - 返回沙箱文件下载的完整URL

#### 命令执行操作
**文件位置**: `e2b/sandbox_async/commands/` 和 `e2b/sandbox_sync/commands/`

- **运行命令** (`commands.run()`)
  - 通过WebSocket连接到沙箱
  - 端点: WebSocket连接到envd API
  
- **运行后台进程** (`commands.start()`)
  - WebSocket连接到envd API
  
- **PTY会话** (`pty.start()`)
  - WebSocket连接到envd API
  - 提供交互式终端

#### 健康检查
**文件位置**: `e2b/sandbox_async/main.py` 和 `e2b/sandbox_sync/main.py`

- **健康检查** (`_wait_until_ready()`)
  - 端点: `GET /health`
  - 用于等待沙箱准备就绪

### 2.2 网络访问

**文件位置**: `e2b/sandbox/main.py`

- **获取主机地址** (`get_host()`)
  - 方法位置: 第191-202行
  - 返回格式: `{port}-{sandbox_id}.{sandbox_domain}`
  - 用于从沙箱外部访问沙箱内运行的服务
  
- **获取MCP URL** (`get_mcp_url()`)
  - 方法位置: 第204-210行
  - 返回: `https://{mcp_port}-{sandbox_id}.{sandbox_domain}/mcp`
  - 用于连接MCP服务器

### 2.3 配置位置

**文件**: `e2b/connection_config.py`

```python
# 获取沙箱URL（第137-141行）
def get_sandbox_url(self, sandbox_id: str, sandbox_domain: str) -> str:
    if self._sandbox_url:
        return self._sandbox_url
    return f"{'http' if self.debug else 'https'}://{self.get_host(sandbox_id, sandbox_domain, self.envd_port)}"

# 获取主机地址（第143-157行）
def get_host(self, sandbox_id: str, sandbox_domain: str, port: int) -> str:
    if self.debug:
        return f"localhost:{port}"
    return f"{port}-{sandbox_id}.{sandbox_domain}"
```

**文件**: `e2b/sandbox/main.py`

```python
# envd API URL初始化（第45-47行）
self.__envd_api_url = self.connection_config.get_sandbox_url(
    self.sandbox_id, self.sandbox_domain
)
```

## 三、总结对比

### API域名 (`api_url`) - 控制平面
用于管理沙箱生命周期和模板的API调用：
- ✅ 创建、删除、列出沙箱
- ✅ 获取沙箱信息和指标
- ✅ 暂停沙箱、设置超时
- ✅ 构建和管理模板
- ✅ 上传构建文件

**默认值**: `https://api.e2b.app`

### Domain域名 (`sandbox_domain`) - 数据平面
用于与运行中的沙箱实例直接交互：
- ✅ 文件系统操作（读、写、列表、删除等）
- ✅ 命令执行（同步、异步）
- ✅ PTY交互式终端
- ✅ 沙箱内服务访问
- ✅ 健康检查
- ✅ 文件上传/下载URL生成

**默认值**: `e2b.app`  
**实际连接格式**: `https://{port}-{sandbox_id}.{sandbox_domain}`

## 四、环境变量配置

```bash
# API域名配置
E2B_API_URL=https://api.e2b.app  # 可选，默认为https://api.{E2B_DOMAIN}
E2B_DOMAIN=e2b.app                # 可选，默认为e2b.app

# 沙箱域名配置
E2B_SANDBOX_URL=                  # 可选，用于覆盖默认的沙箱URL

# 调试模式
E2B_DEBUG=false                   # 设为true时，API URL为http://localhost:3000
```

## 五、架构图示

```
用户代码
    │
    ├─► API域名 (https://api.e2b.app)
    │       │
    │       ├─► POST /sandboxes (创建沙箱)
    │       ├─► GET /sandboxes/{id} (获取信息)
    │       ├─► DELETE /sandboxes/{id} (删除沙箱)
    │       ├─► POST /v2/templates (创建模板)
    │       └─► GET /templates/{id}/builds/{id}/status (构建状态)
    │
    └─► Domain域名 (https://{port}-{sandbox_id}.e2b.app)
            │
            ├─► GET /health (健康检查)
            ├─► GET /files (文件系统操作)
            ├─► POST /files (文件系统操作)
            └─► WebSocket (命令执行、PTY)
```

## 参考文件

1. `/home/runner/work/E2B/E2B/packages/python-sdk/e2b/connection_config.py` - 连接配置
2. `/home/runner/work/E2B/E2B/packages/python-sdk/e2b/api/__init__.py` - API客户端
3. `/home/runner/work/E2B/E2B/packages/python-sdk/e2b/sandbox/main.py` - 沙箱基类
4. `/home/runner/work/E2B/E2B/packages/python-sdk/e2b/sandbox_async/sandbox_api.py` - 异步沙箱API
5. `/home/runner/work/E2B/E2B/packages/python-sdk/e2b/template_async/build_api.py` - 异步模板构建API
