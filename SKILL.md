---
name: ssh-deploy-skill
version: 1.2.0
description: Universal SSH remote deployment tool - multi-server management, batch deployment, installation script templates with domestic mirror optimization. Supports remote installation of Git, Docker, MySQL, PostgreSQL, Nginx, Node.js, Redis, Python and more.
when: When you need to deploy Linux servers remotely, batch install software, or manage SSH connections (e.g., install Docker, configure databases, sync files)
examples:
  - user: 在服务器上安装 Docker
    assistant: 使用 ssh-deploy 安装 Docker
  - user: 批量部署三台服务器的环境
    assistant: 使用 ssh-deploy 批量部署
  - user: 给新服务器配置阿里云镜像源
    assistant: 使用 ssh-deploy 配置国内镜像
  - user: 上传配置文件到所有后端服务器
    assistant: 使用 ssh-deploy 上传文件
  - user: 在所有测试服务器上执行数据库迁移
    assistant: 使用 ssh-deploy 执行批量命令
metadata:
  openclaw:
    emoji: 🚀
    requires:
      python: ">=3.8"
      bins: ["python3", "ssh", "scp"]
    install:
      - id: paramiko
        kind: pip
        package: paramiko
        label: Install paramiko (SSH library)
---

# SSH Deploy Skill - 通用 SSH 远程部署工具

## 📋 概述

通过 paramiko + 自定义部署引擎管理任意 Linux 服务器。本技能已针对国内网络环境进行完整优化。

## 🚀 快速开始

### 1. 首次使用

```bash
cd /root/.openclaw/workspace/skills/ssh-deploy-skill

# 自动设置（检查依赖、创建配置）
bash scripts/setup.sh

# 手动安装依赖（如果自动失败）
pip3 install --user paramiko
```

### 2. 配置服务器（两种方式）

#### 方式 A：使用 inventory.json（原有方式）

编辑 `~/.ssh-deploy/inventory.json` 文件，添加你的服务器信息，或使用命令行添加：

```bash
python3 scripts/inventory.py add web-01 \
  --host 192.168.1.101 \
  --user root \
  --ssh-key ~/.ssh/id_rsa \
  --groups production,web \
  --tags "阿里云"
```

**配置文件位置**：所有服务器配置都保存在 `~/.ssh-deploy/inventory.json` 文件中。

#### 方式 B：直接从 ~/.ssh/config 读取（新增）

如果你已经在 `~/.ssh/config` 中配置了 Host 条目，**无需任何额外配置**，可以直接使用这些 host 名：

```bash
# ~/.ssh/config 内容示例
Host dy-c1
    HostName 101.126.92.30
    User root
    IdentityFile ~/.ssh/mypc_id_rsa
    Port 22

# 直接使用 host 名执行命令
python3 scripts/deploy.py exec dy-c1 "ls -la /opt"
```

系统会自动解析 `~/.ssh/config` 中的 `Host`、`HostName`、`Port`、`User`、`IdentityFile` 等字段。

> **注意**：从 SSH config 加载的服务器是**动态只读**的，不会保存到 `inventory.json`。如果你想添加分组或标签，请使用 `inventory.py add` 将其导入。

### 3. 执行远程命令

```bash
# 单台服务器
python3 scripts/deploy.py exec web-01 "uptime && df -h"

# 批量执行（按分组）
python3 scripts/deploy.py exec group:production "docker ps"

# 批量执行（按标签）
python3 scripts/deploy.py exec tag:"阿里云" "systemctl status nginx"

# 顺序执行（避免并发过高）
python3 scripts/deploy.py exec group:large "apt update" --sequential
```

### 4. 文件传输

```bash
# 上传文件
python3 scripts/deploy.py upload web-01 ./nginx.conf /etc/nginx/nginx.conf

# 批量上传（所有分组服务器）
python3 scripts/deploy.py upload group:web ./config.json /opt/app/config.json

# 下载文件
python3 scripts/deploy.py download web-01 /var/log/nginx/access.log ./logs/
```

### 5. 使用模板安装软件

所有模板位于 `templates/`，已预配置国内镜像源。

```bash
# 安装 Docker（自动配置国内镜像加速）
cat templates/install_docker.sh | python3 scripts/deploy.py exec tag:docker "bash -s"

# 安装 MySQL（环境变量设置密码）
MYSQL_ROOT_PASSWORD=YourPass123 cat templates/install_mysql.sh | \
  python3 scripts/deploy.py exec db-01 "bash -s"

# 基础环境配置（系统更新 + 镜像源）
cat templates/base_setup.sh | python3 scripts/deploy.py exec group:all "bash -s"
```

## 📦 安装脚本模板列表

| 模板 | 功能 | 国内镜像优化 | 环境变量 |
|------|------|-------------|----------|
| `base_setup.sh` | 基础环境配置 | ✅ | - |
| `install_git.sh` | Git | ❌ | GIT_USER_NAME, GIT_USER_EMAIL |
| `install_docker.sh` | Docker CE + 加速器 | ✅ | - |
| `install_mysql.sh` | MySQL 8.0 | ✅ | MYSQL_ROOT_PASSWORD |
| `install_postgresql.sh` | PostgreSQL 15 | ✅ | PG_VERSION |
| `install_nginx.sh` | Nginx | ❌ | - |
| `install_nodejs.sh` | Node.js | ✅（npm） | NODE_VERSION |
| `install_redis.sh` | Redis | ❌ | - |
| `install_python.sh` | Python | ✅（pip） | PYTHON_VERSION |

> **注意**：所有已添加的服务器配置保存在 `~/.ssh-deploy/inventory.json` 文件中。

## 🎯 核心功能

### 服务器清单管理 (`inventory.py`)

```bash
# 列出所有服务器
python3 scripts/inventory.py list

# 按分组列出
python3 scripts/inventory.py list --group production

# 按标签列出
python3 scripts/inventory.py list --tag "阿里云"

# 添加服务器
python3 scripts/inventory.py add SERVER_NAME \
  --host IP_OR_HOSTNAME \
  --port 22 \
  --user USERNAME \
  --ssh-key PATH_TO_KEY \
  --groups GROUP1,GROUP2 \
  --tags TAG1,TAG2 \
  --desc "描述信息"
```

**目标语法**（在 `deploy.py` 中使用）：
- `server-name` - 具体服务器
- `group:分组名` - 所有该分组的服务器
- `tag:标签名` - 所有包含该标签的服务器
- `*` - 所有服务器

### 远程命令执行 (`deploy.py`)

```bash
# 并行批量执行（默认）
python3 scripts/deploy.py exec group:web "docker pull nginx"

# 顺序执行（大 batch 或避免负载过高）
python3 scripts/deploy.py exec group:large "apt upgrade -y" --sequential

# 使用环境变量
export MYSQL_VERSION="8.0"
cat templates/install_mysql.sh | python3 scripts/deploy.py exec db-01 "bash -s"
```

### 文件传输

```bash
# 上传（单台）
python3 scripts/deploy.py upload web-01 ./local.conf /etc/app/conf.d/conf.conf

# 批量上传
python3 scripts/deploy.py upload group:web ./nginx.conf /etc/nginx/nginx.conf

# 下载
python3 scripts/deploy.py download web-01 /var/log/app.log ./logs/
```

## 🌏 国内网络优化

### 镜像源配置

本技能默认使用**阿里云镜像源**，已配置：

- **Ubuntu/Debian** → `mirrors.aliyun.com`
- **CentOS/RHEL** → `mirrors.aliyun.com`
- **npm** → `registry.npmmirror.com`（淘宝）
- **pip** → `mirrors.aliyun.com/pypi/simple`
- **Docker** → 中科大、网易、百度加速器
- **Go** → `goproxy.cn`
- **Maven** → 阿里云仓库

详细配置手册：`references/mirrors.md`

### 一键配置国内镜像

```bash
# 在所有服务器上配置系统镜像源
cat templates/base_setup.sh | python3 scripts/deploy.py exec group:all "bash -s"

# 配置 Docker 加速（在安装docker.sh中已自动执行）
# 如果需要手动配置：
cat <<'EOF' | python3 scripts/deploy.py exec group:all "bash -s"
cat > /etc/docker/daemon.json <<DOCKER
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
  ]
}
DOCKER
systemctl restart docker
EOF
```

## 🔒 安全最佳实践

1. **SSH 密钥管理**
   - 使用密钥认证，禁用密码
   - 密钥文件权限 `chmod 600 ~/.ssh/id_rsa`
   - 不同环境使用不同密钥

2. **最小权限**
   - 创建专用部署用户，不使用 root
   - 配置免密码 sudo（仅限必要命令）
   - 服务器 `sshd_config` 中：`PermitRootLogin no`, `PasswordAuthentication no`

3. **敏感信息**
   - 不在 `inventory.json` 中存储密码
   - 使用环境变量传递密码（如 `MYSQL_ROOT_PASSWORD`）
   - 配置文件权限设为 `640`，属主 `root:appgroup`

4. **审计**
   - 保留所有部署日志
   - 记录执行的命令、服务器、时间、结果

完整安全指南：`references/best-practices.md`

## 🐛 故障排查

### 快速诊断

```bash
# 1. 网络测试
ping <host>
telnet <host> 22

# 2. 手动 SSH 测试
ssh -i ~/.ssh/id_rsa root@<host> "uptime"

# 3. 查看连接详情
python3 scripts/deploy.py exec <server> "uptime" 2>&1

# 4. 验证密钥权限
ls -la ~/.ssh/id_rsa*
```

### 常见问题

| 问题 | 解决方案 |
|------|----------|
| 连接拒绝/超时 | 检查服务器状态、SSH 服务、防火墙/安全组 |
| Permission denied (publickey) | 检查公钥是否在服务器的 `~/.ssh/authorized_keys`，密钥权限 600 |
| sudo: a password is required | 配置免密码 sudo 或使用 root 用户 |
| Command not found | 使用绝对路径或确保 PATH 包含命令 |
| 国内下载慢 | 运行 `base_setup.sh` 配置镜像源，或参考 `docs/mirrors.md` |

详细排查手册：`references/troubleshooting.md`

## 📊 批量操作示例

### 场景1：全新服务器初始化

```bash
# 1. 添加服务器到清单
python3 scripts/inventory.py add web-01 --host 1.2.3.101 --groups web --tags "生产"

# 2. 基础环境配置（镜像源、工具）
cat templates/base_setup.sh | python3 scripts/deploy.py exec web-01 "bash -s"

# 3. 安装核心软件
cat templates/install_docker.sh | python3 scripts/deploy.py exec web-01 "bash -s"
cat templates/install_nginx.sh | python3 scripts/deploy.py exec web-01 "bash -s"

# 4. 上传应用配置
python3 scripts/deploy.py upload web-01 ./app-config.json /opt/app/config.json

# 5. 启动应用
python3 scripts/deploy.py exec web-01 "docker-compose up -d"
```

### 场景2：多台服务器轮询更新

```bash
# 灰度发布：先更新第一批
cat deploy-v2.sh | python3 scripts/deploy.py exec tag:"灰度-1" "bash -s"

# 检查监控指标，确认正常...

# 全量发布
cat deploy-v2.sh | python3 scripts/deploy.py exec tag:生产 "bash -s"
```

### 场景3：文件同步

```bash
# 同步配置文件到所有 Web 服务器
python3 scripts/deploy.py upload group:web ./nginx.conf /etc/nginx/nginx.conf

# 批量重启
python3 scripts/deploy.py exec group:web "nginx -t && systemctl reload nginx"
```

### 场景4：巡检与监控

```bash
# 批量收集服务器状态
python3 scripts/deploy.py exec "*" "uptime" > uptime-$(date +%F).log

python3 scripts/deploy.py exec "*" "df -h" > disk-$(date +%F).log

python3 scripts/deploy.py exec "*" "docker ps --format 'table {{.Names}}\t{{.Status}}'" > containers-$(date +%F).log
```

## 🔄 CI/CD 集成

### GitLab CI 示例

```yaml
stages:
  - deploy

deploy_production:
  stage: deploy
  script:
    - pip3 install --user paramiko
    - export TARGET="group:production"
    - cat deploy.sh | python3 skills/ssh-deploy-skill/scripts/deploy.py exec "$TARGET" "bash -s"
  only:
    - main
```

### GitHub Actions 示例

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install paramiko
        run: pip3 install paramiko
      - name: Deploy to servers
        run: |
          cat deploy.sh | python3 skills/ssh-deploy-skill/scripts/deploy.py exec "group:staging" "bash -s"
```

## 🛠️ 开发者指南

### 添加新的安装模板

1. 在 `templates/` 目录创建 `.sh` 文件
2. 使用 `#!/bin/bash` 和 `set -e`（出错退出）
3. 检测系统类型并适配（参考现有模板）
4. 使用环境变量作为参数，不要硬编码敏感数据
5. 添加国内镜像源配置（如适用）

### 自定义脚本结构

```bash
#!/bin/bash
set -e

echo "===== 安装 XXX ====="

# 1. 检测系统
if [ -f /etc/debian_version ]; then
    OS="debian"
elif [ -f /etc/redhat-release ]; then
    OS="redhat"
else
    echo "不支持的系统"
    exit 1
fi

# 2. 配置国内镜像（如果需要）
# ...

# 3. 安装软件
if [ "$OS" = "debian" ]; then
    apt-get update
    apt-get install -y xxx
elif [ "$OS" = "redhat" ]; then
    yum install -y xxx
fi

# 4. 启动服务
systemctl start xxx
systemctl enable xxx

echo "XXX 安装完成！"
```

### 在 Python 中直接使用库

```python
from inventory import Inventory, Server
from deploy import SSHDeployer

# 加载清单
inv = Inventory()
server = inv.get_server("web-01")

# 创建部署器
deployer = SSHDeployer()

# 执行命令
result = deployer.execute(server, "docker ps")
print(result.success, result.output)

# 上传文件
res = deployer.upload_file(server, "./local.conf", "/etc/conf.d/local.conf")

deployer.close()
```

## 📚 更多文档

- `README.md` - 完整使用指南（包含 API 参考）
- `references/mirrors.md` - 国内镜像源配置详解
- `references/best-practices.md` - 最佳实践和代码示例
- `references/troubleshooting.md` - 故障排查完整手册

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

## 📄 许可证

MIT License
