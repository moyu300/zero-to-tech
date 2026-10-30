# Zero-to-Tech 部署文档

## 项目结构

```
/home/dev/zero-to-tech/
├── backend/          # FastAPI 后端
│   ├── .venv/        # Python 3.13 虚拟环境
│   ├── .env          # 环境变量（systemd EnvironmentFile）
│   ├── main.py       # 应用入口
│   ├── history.db    # SQLite 数据库（权限 664，dev 组可写）
│   └── requirements.txt
├── frontend/         # Next.js 前端源码
│   └── out/          # 静态导出产物（Nginx root 指向此目录）
└── README.md
```

## 环境信息

|前提：服务器上已装好 Python 3.10+、Node.js 18+ 和 Nginx。

| 项目 | 值 |
| :--- | :--- |
| 公网 IP | `121.196.150.145` |
| 系统 | Ubuntu 22.04, Python 3.10 (系统) / 3.13 (venv) |
| 前端端口 | `9527` (Nginx) |
| 后端端口 | `8000` (Uvicorn, 仅监听 127.0.0.1) |
| 运行用户 | `fastapi` (后端 systemd), `dev` (开发) |

---

## Systemd 服务配置

### 服务文件

路径：`/etc/systemd/system/zero-to-tech-backend.service`

```ini
[Unit]
Description=Zero-to-Tech FastAPI Backend
After=network.target

[Service]
Type=simple
User=fastapi
Group=fastapi
WorkingDirectory=/home/dev/zero-to-tech/backend
ExecStart=/home/dev/zero-to-tech/backend/.venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000
Restart=always
RestartSec=5
EnvironmentFile=/home/dev/zero-to-tech/backend/.env

[Install]
WantedBy=multi-user.target
```

### 配置说明

| 字段 | 值 | 说明 |
| :--- | :--- | :--- |
| User/Group | `fastapi` | 隔离运行，不使用 root |
| WorkingDirectory | `/home/dev/zero-to-tech/backend` | 工作目录，确保相对路径正确 |
| ExecStart | `.venv/bin/uvicorn` | 直接使用 venv 中的 uvicorn，无需 activate |
| --host | `127.0.0.1` | 仅本地监听，外网通过 Nginx 反代访问 |
| Restart | `always` | 异常退出自动重启 |
| RestartSec | `5` | 重启间隔 5 秒 |
| EnvironmentFile | `.env` | 加载环境变量（如数据库路径、密钥等） |

### 常用命令

```bash
# 启动 / 停止 / 重启
sudo systemctl start zero-to-tech-backend
sudo systemctl stop zero-to-tech-backend
sudo systemctl restart zero-to-tech-backend

# 查看状态 / 日志
sudo systemctl status zero-to-tech-backend
sudo journalctl -u zero-to-tech-backend -f

# 修改服务文件后重载
sudo systemctl daemon-reload
sudo systemctl restart zero-to-tech-backend

# 开机自启
sudo systemctl enable zero-to-tech-backend
```

### ⚠️ 权限注意事项

- `history.db` 必须对 `dev` 组可写：`chmod g+w history.db`
- `fastapi` 用户已加入 `dev` 组，通过组权限访问项目文件
- 新建/重建数据库文件后需重新设置组写权限

---

## Nginx 配置

### 配置文件

路径：`/etc/nginx/sites-available/zero-to-tech`（软链至 `sites-enabled`）

```nginx
server {
    listen 9527;
    listen [::]:9527;
    server_name _;

    # Next.js 静态导出目录
    root /home/dev/zero-to-tech/frontend/out;
    index index.html index.htm;

    # API 反向代理 → 后端 Uvicorn
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # 前端静态文件
    location / {
        try_files $uri $uri.html $uri/ =404;
    }
}
```

### 配置说明

| 指令 | 值 | 说明 |
| :--- | :--- | :--- |
| listen | `9527` | 前端对外端口 |
| root | `frontend/out` | Next.js `next export` 输出目录 |
| location /api/ | `proxy_pass http://127.0.0.1:8000` | 反代到后端，末尾无 `/` 保留 `/api/` 前缀 |
| try_files | `$uri $uri.html $uri/ =404` | Next.js 静态导出使用 `.html` 后缀路由 |

### 常用命令

```bash
# 语法检查 + 重载
sudo nginx -t && sudo systemctl reload nginx

# 重新构建前端后重载
cd /home/dev/zero-to-tech/frontend && npm run build
sudo systemctl reload nginx
```

---

## 后端开发模式（Screen）

> 生产环境使用 systemd，开发调试时使用 screen 后台运行。

```bash
# 后台启动
screen -dmS fastapi bash -c "source /home/dev/zero-to-tech/backend/.venv/bin/activate && fastapi run > /tmp/fastapi.log 2>&1"

# 接入 / 分离 / 停止
screen -r fastapi        # 接入
Ctrl+A, D                # 分离
screen -S fastapi -X quit  # 停止
tail -f /tmp/fastapi.log   # 查看日志
```

---

## 依赖安装

```bash
cd /home/dev/zero-to-tech/backend
source .venv/bin/activate
pip install -r requirements.txt -i http://mirrors.cloud.aliyuncs.com/pypi/simple/ --trusted-host mirrors.cloud.aliyuncs.com
```

> **注意**：必须使用 `.venv` (Python 3.13)，系统 Python 3.10 不兼容部分依赖。

## 访问地址

| 服务 | 地址 |
| :--- | :--- |
| 前端 | `http://121.196.150.145:9527` |
| 后端 API（经 Nginx 反代） | `http://121.196.150.145:9527/api/` |
| 后端 API 文档（直连） | `http://121.196.150.145:8000/docs` |

## 安全组端口

| 端口 | 用途 | 状态 |
| :--- | :--- | :--- |
| 22 | SSH | ✅ 已放行 |
| 9527 | 前端 + API 反代 (Nginx) | ✅ 已放行 |
| 8000 | 后端直连（仅调试用） | ⚠️ 按需放行 |

## 常见问题

### `fastapi: command not found`
未激活虚拟环境，执行 `source .venv/bin/activate` 后再运行。

### pip 安装极慢
确认使用 venv 中的 pip：`which pip` 应输出 `.venv/bin/pip`。

### POST /api/analyze 返回 500
检查数据库写权限：`chmod g+w /home/dev/zero-to-tech/backend/history.db`

### 前端页面空白或 404
确认 Nginx root 指向 `frontend/out`，修改后执行 `nginx -t && systemctl reload nginx`。

### systemd 启动失败
查看日志：`journalctl -u zero-to-tech-backend -f`，常见原因：
- `.env` 文件缺失或权限不对
- `history.db` 对 fastapi 用户不可写
- venv 路径错误或 Python 版本不匹配

## 路径说明

- **绝对路径**：从 `/` 开始，如 `/home/dev/zero-to-tech/backend/.venv/bin/activate`
- **相对路径**：相对于当前 `pwd`，如 `.venv/bin/activate`（需在 backend 目录下执行）
