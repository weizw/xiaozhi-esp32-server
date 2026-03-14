# 小智ESP32服务端部署文档

## 一、环境要求

### 1.1 必需软件
- Docker Engine (20.10+)
- Docker Compose (2.0+)
- curl (用于下载模型)

### 1.2 端口要求
确保以下端口未被占用：
- `8000` - WebSocket服务端口
- `8002` - Web管理后台端口
- `8003` - HTTP服务端口（视觉分析接口、OTA接口）
- `3306` - MySQL端口（容器内部）
- `6379` - Redis端口（容器内部）

### 1.3 系统资源
- **磁盘空间**：至少 5GB 可用空间
  - 模型文件：893MB
  - Docker镜像：约 2-3GB
  - 数据库和日志：预留空间
- **内存**：建议 4GB 以上
- **CPU**：支持多核心处理

---

## 二、快速部署步骤

### 2.1 创建必要目录

```bash
# 进入项目根目录
cd xiaozhi-esp32-server/main/xiaozhi-server

# 创建数据目录
mkdir -p data uploadfile mysql/data
```

### 2.2 下载语音识别模型

**模型文件较大（893MB），下载时间取决于网络速度**

```bash
# 下载 SenseVoiceSmall 语音识别模型
curl -L --progress-bar https://modelscope.cn/models/iic/SenseVoiceSmall/resolve/master/model.pt -o models/SenseVoiceSmall/model.pt
```

**验证模型文件：**
```bash
ls -lh models/SenseVoiceSmall/model.pt
# 预期输出：-rw-r--r--  1 user  staff   893M ... model.pt
```

### 2.3 准备初始配置文件

```bash
# 复制示例配置文件
cp config_from_api.yaml data/.config.yaml
```

### 2.4 启动Docker服务

```bash
# 启动所有服务（MySQL、Redis、Web后台、Server）
docker compose -f docker-compose_all.yml up -d
```

**首次启动会拉取镜像，耗时较长（约5-15分钟，取决于网络）**

### 2.5 配置管理密钥

#### 步骤1：访问管理后台
浏览器打开：http://localhost:8002

#### 步骤2：注册账号
- 第一个注册的账号自动成为**超级管理员**
- 后续注册的账号为普通用户
- 普通用户只能绑定设备和配置智能体
- 超级管理员可以进行模型管理、用户管理、参数配置等

#### 步骤3：获取密钥
1. 使用管理员账号登录
2. 点击顶部菜单 **"参数字典"** → **"参数管理"**
3. 找到参数编码 `server.secret`
4. 复制参数值（UUID格式，如：`78b0b198-b36c-4faf-bcb5-27e9d19426f7`）

#### 步骤4：更新配置文件

**方法一：手动编辑（推荐）**
```bash
# 编辑配置文件
vim data/.config.yaml
# 或
nano data/.config.yaml
```

修改以下内容：
```yaml
manager-api:
  # Docker部署时使用容器名称
  url: http://xiaozhi-esp32-server-web:8002/xiaozhi
  # 替换为步骤3复制的密钥值
  secret: 你的server.secret值
```

**方法二：使用命令更新**
```bash
# 替换 <YOUR_SECRET> 为实际密钥
cat > data/.config.yaml << 'EOF'
server:
  ip: 0.0.0.0
  port: 8000
  http_port: 8003
  vision_explain: http://localhost:8003/mcp/vision/explain
manager-api:
  url: http://xiaozhi-esp32-server-web:8002/xiaozhi
  secret: <YOUR_SECRET>
prompt_template: agent-base-prompt.txt
EOF
```

### 2.6 重启服务器服务

```bash
docker restart xiaozhi-esp32-server
```

### 2.7 验证服务状态

```bash
# 检查所有容器状态
docker ps | grep xiaozhi

# 预期输出：所有容器状态为 "Up"
# xiaozhi-esp32-server-web    Up
# xiaozhi-esp32-server        Up
# xiaozhi-esp32-server-db     Up (healthy)
# xiaozhi-esp32-server-redis  Up (healthy)
```

```bash
# 查看服务器日志
docker logs --tail 30 xiaozhi-esp32-server

# 成功标志：
# - "从API读取配置"
# - "ASR模块初始化完成"
# - "Websocket地址是 ws://..."
```

---

## 三、服务访问地址

### 3.1 主要服务

| 服务名称 | 地址 | 说明 |
|---------|------|------|
| Web管理后台 | http://localhost:8002 | 设备管理、模型配置、用户管理 |
| WebSocket服务 | ws://localhost:8000/xiaozhi/v1/ | ESP32设备连接地址 |
| HTTP服务 | http://localhost:8003 | 基础HTTP服务 |
| 视觉分析接口 | http://localhost:8003/mcp/vision/explain | 图像识别分析 |
| OTA接口 | http://localhost:8002/xiaozhi/ota/ | 固件升级接口 |

### 3.2 容器内部地址

**注意**：以下地址仅供容器内部通信使用

- MySQL: `xiaozhi-esp32-server-db:3306`
- Redis: `xiaozhi-esp32-server-redis:6379`
- Web API: `xiaozhi-esp32-server-web:8002`

---

## 四、重要注意事项

### 4.1 配置文件位置 ⚠️

**配置文件路径**：`main/xiaozhi-server/data/.config.yaml`

**常见错误**：
- ❌ 直接修改 `config_from_api.yaml`（这是模板文件）
- ❌ 修改 `config.yaml`（这是参考配置）
- ✅ 正确：修改 `data/.config.yaml`（实际使用的配置）

### 4.2 管理密钥配置 ⚠️

**错误现象**：
```
Exception: 请先配置manager-api的secret
```

**解决方案**：
1. 确保 Web 管理后台已启动并可访问
2. 注册账号并登录
3. 从"参数管理"获取 `server.secret`
4. 更新 `data/.config.yaml` 中的 `manager-api.secret`
5. 重启 xiaozhi-esp32-server 容器

### 4.3 Docker网络配置 ⚠️

**Docker部署时，manager-api.url 必须使用容器名称**：
```yaml
# ✅ 正确（Docker部署）
url: http://xiaozhi-esp32-server-web:8002/xiaozhi

# ❌ 错误（Docker部署）
url: http://127.0.0.1:8002/xiaozhi
url: http://localhost:8002/xiaozhi
```

**本地部署（非Docker）时使用**：
```yaml
url: http://127.0.0.1:8002/xiaozhi
```

### 4.4 模型文件检查 ⚠️

**确保模型文件存在且完整**：
```bash
# 检查文件
ls -lh models/SenseVoiceSmall/model.pt

# 如果文件不存在或大小异常，重新下载
curl -L --progress-bar https://modelscope.cn/models/iic/SenseVoiceSmall/resolve/master/model.pt -o models/SenseVoiceSmall/model.pt
```

### 4.5 端口冲突 ⚠️

**检查端口占用**：
```bash
# macOS
lsof -i :8000
lsof -i :8002
lsof -i :8003

# Linux
netstat -tlnp | grep -E '8000|8002|8003'
```

**解决方案**：
1. 停止占用端口的服务
2. 或修改 `docker-compose_all.yml` 中的端口映射

### 4.6 数据持久化 ⚠️

**重要数据目录**：
- `data/` - 配置文件目录
- `mysql/data/` - MySQL数据目录
- `uploadfile/` - 上传文件目录

**备份建议**：
```bash
# 定期备份
tar -czf xiaozhi-backup-$(date +%Y%m%d).tar.gz data/ mysql/data/ uploadfile/
```

### 4.7 防火墙配置 ⚠️

**云服务器部署时，需要开放端口**：
- 8000 - WebSocket服务
- 8002 - Web管理后台
- 8003 - HTTP服务

**配置示例（Ubuntu UFW）**：
```bash
ufw allow 8000/tcp
ufw allow 8002/tcp
ufw allow 8003/tcp
```

### 4.8 公网部署配置 ⚠️

**修改 `vision_explain` 地址**：
```yaml
server:
  # 使用公网IP或域名
  vision_explain: http://你的公网IP:8003/mcp/vision/explain
  # 或使用域名
  vision_explain: https://your-domain.com/mcp/vision/explain
```

---

## 五、服务管理命令

### 5.1 启动服务

```bash
cd main/xiaozhi-server
docker compose -f docker-compose_all.yml up -d
```

### 5.2 停止服务

```bash
cd main/xiaozhi-server
docker compose -f docker-compose_all.yml down
```

### 5.3 重启服务

```bash
# 重启所有服务
cd main/xiaozhi-server
docker compose -f docker-compose_all.yml restart

# 仅重启服务器
docker restart xiaozhi-esp32-server
```

### 5.4 查看日志

```bash
# 查看服务器日志
docker logs xiaozhi-esp32-server

# 实时查看日志
docker logs -f xiaozhi-esp32-server

# 查看最近50行
docker logs --tail 50 xiaozhi-esp32-server

# 查看Web后台日志
docker logs xiaozhi-esp32-server-web
```

### 5.5 查看容器状态

```bash
# 查看运行状态
docker ps | grep xiaozhi

# 查看详细信息
docker inspect xiaozhi-esp32-server
```

---

## 六、常见问题排查

### 6.1 服务无法启动

**检查步骤**：
```bash
# 1. 查看容器状态
docker ps -a | grep xiaozhi

# 2. 查看错误日志
docker logs xiaozhi-esp32-server

# 3. 检查配置文件
cat data/.config.yaml

# 4. 检查模型文件
ls -lh models/SenseVoiceSmall/model.pt
```

### 6.2 无法连接WebSocket

**检查步骤**：
```bash
# 1. 确认服务运行
docker ps | grep xiaozhi-esp32-server

# 2. 确认端口监听
lsof -i :8000

# 3. 测试WebSocket连接
# 使用浏览器打开 test/test_page.html
```

### 6.3 无法访问管理后台

**检查步骤**：
```bash
# 1. 确认服务运行
docker ps | grep xiaozhi-esp32-server-web

# 2. 查看日志
docker logs xiaozhi-esp32-server-web

# 3. 确认端口监听
lsof -i :8002
```

### 6.4 数据库连接失败

**检查步骤**：
```bash
# 1. 确认MySQL运行
docker ps | grep xiaozhi-esp32-server-db

# 2. 查看MySQL日志
docker logs xiaozhi-esp32-server-db

# 3. 进入MySQL检查
docker exec -it xiaozhi-esp32-server-db mysql -uroot -p123456
```

### 6.5 容器一直重启

**原因**：通常是配置错误或服务依赖未就绪

**解决方案**：
```bash
# 1. 查看退出原因
docker logs xiaozhi-esp32-server

# 2. 常见原因：
#    - manager-api.secret 未配置
#    - 模型文件缺失
#    - MySQL/Redis 未启动完成

# 3. 检查依赖服务
docker ps | grep xiaozhi
```

---

## 七、升级指南

### 7.1 备份数据

```bash
# 备份配置和数据
cd main/xiaozhi-server
tar -czf backup-$(date +%Y%m%d).tar.gz data/ mysql/data/ uploadfile/
```

### 7.2 拉取最新镜像

```bash
cd main/xiaozhi-server
docker compose -f docker-compose_all.yml pull
```

### 7.3 重启服务

```bash
docker compose -f docker-compose_all.yml up -d
```

---

## 八、卸载指南

### 8.1 停止并删除容器

```bash
cd main/xiaozhi-server
docker compose -f docker-compose_all.yml down
```

### 8.2 删除镜像

```bash
docker rmi ghcr.nju.edu.cn/xinnan-tech/xiaozhi-esp32-server:server_latest
docker rmi ghcr.nju.edu.cn/xinnan-tech/xiaozhi-esp32-server:web_latest
```

### 8.3 删除数据（⚠️ 不可恢复）

```bash
# 谨慎操作！会删除所有数据
rm -rf data/ mysql/data/ uploadfile/
```

---

## 九、部署清单

完成部署后，请确认以下检查项：

- [ ] Docker 和 Docker Compose 已安装
- [ ] 端口 8000、8002、8003 未被占用
- [ ] 模型文件已下载（893MB）
- [ ] 配置文件 `data/.config.yaml` 已创建
- [ ] Docker 服务已启动
- [ ] Web 管理后台可访问（http://localhost:8002）
- [ ] 管理员账号已注册
- [ ] server.secret 已配置到配置文件
- [ ] xiaozhi-esp32-server 容器正常运行
- [ ] WebSocket 地址可访问
- [ ] 视觉分析接口可访问

---

## 十、技术支持

- **项目地址**：https://github.com/xinnan-tech/xiaozhi-esp32-server
- **问题反馈**：https://github.com/xinnan-tech/xiaozhi-esp32-server/issues
- **常见问题**：docs/FAQ.md
- **通信协议**：https://ccnphfhqs21z.feishu.cn/wiki/M0XiwldO9iJwHikpXD5cEx71nKh

---

**文档版本**：v1.0  
**最后更新**：2026-03-11  
**适用版本**：xiaozhi-esp32-server latest
