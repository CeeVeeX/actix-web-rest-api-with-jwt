# Actix-web REST API（含 JWT）

基于 Actix-web + Diesel + JWT 的简单 CRUD 后端示例，内置用户认证、地址簿增删改查与分页筛选。

## 环境要求

- Rust 稳定版（推荐使用 rustup 安装）：https://rustup.rs
- PostgreSQL 数据库：https://www.postgresql.org/
- 可选：Docker 与 Docker Compose：https://www.docker.com/

### 依赖库（Diesel 后端）

本项目使用 Diesel，需安装 Postgres 与 SQLite 的客户端/开发库。

- Windows：将仓库内 `libs\windows` 目录下的所有文件复制到一个路径（如 `C:\\libs`），然后设置系统环境变量：
  - `PQ_LIB_DIR=C:\\libs`
  - `SQLITE3_LIB_DIR=C:\\libs`
  重启 PowerShell/终端后再编译项目。
- Linux：根据发行版安装 `libpq` 与 `libsqlite3`。
  - Fedora/CentOS：
    - 通过 `yum list '*sqlite*'` 查找库
    - 安装：`sudo dnf/yum install libsqlite3x.x86_64 libsqlite3x-devel.x86_64`
  - Ubuntu/Ubuntu Server：`sudo apt-get install libsqlite3-dev libpq-dev`
- macOS：使用 Homebrew 安装 `libpq`：`brew install libpq`

## 快速开始

### 使用 Docker

1. 进入项目目录
2. 本地环境：
   - 启动：`docker-compose -f docker-compose.local.yml up`
3. 生产环境：
   - 启动：`docker-compose -f docker-compose.prod.yml up`

### 手动运行（本机）

1. 准备数据库：使用 Postgres CLI 或 [pgAdmin](https://www.pgadmin.org/) 创建数据库
2. 配置密钥：将 `src/secret.key.sample` 重命名为 `src/secret.key`
   - 或在 Linux/UNIX 生成新密钥：`head -c16 /dev/urandom > secret.key`，再拷贝到 `src` 目录
3. 复制环境配置：将 `dotenv.sample` 重命名为 `.env`，并设置 `DATABASE_URL`（示例：`postgres://user:pass@localhost:5432/dbname`）
4. 构建发布版本：

```powershell
cargo build --release
```

5. 运行可执行文件：
   - Windows：`target\\release\\actix-web-rest-api-with-jwt.exe`
   - Linux/UNIX：`target/release/actix-web-rest-api-with-jwt`

服务默认监听 `localhost:8000`。

## 与 yew-address-book-client 的说明

- 前端仓库：https://github.com/SakaDream/yew-address-book-client
- 该前端目前缺少登录/注册页，如需在开发中临时禁用后端认证中间件，可参考 `src/main.rs` 中相关行并注释掉。

## API 文档

基础地址：`http://127.0.0.1:8000`

### `GET /api/ping`（健康检查）

示例：

```bash
curl -X GET -i 'http://127.0.0.1:8000/api/ping'
```

返回：

- 200 OK，文本：`pong!`

### `POST /api/auth/signup`（注册）

请求示例：

```bash
curl -X POST -i 'http://127.0.0.1:8000/api/auth/signup' \
  -H "Content-Type: application/json" \
  --data '{
    "username": "user",
    "email": "user@email.com",
    "password": "4S3cr3tPa55w0rd"
  }'
```

请求体：

```json
{
  "username": "string",
  "email": "string",
  "password": "string" // 原始密码
}
```

响应：

- 200 OK

```json
{
  "message": "signup successfully",
  "data": ""
}
```

- 400 Bad Request（用户名已存在）

```json
{
  "message": "User '{username}' is already registered",
  "data": ""
}
```

### `POST /api/auth/login`（登录）

请求示例：

```bash
curl -X POST -H 'Content-Type: application/json' -i 'http://127.0.0.1:8000/api/auth/login'  \
  --data '{"username_or_email":"user",  "password":"4S3cr3tPa55w0rd"}'
```

请求体：

```json
{
  "username_or_email": "string",
  "password": "string" // 原始密码
}
```

响应：

- 200 OK

```json
{
  "message": "login successfully",
  "data": {
    "token": "string" // Bearer Token
  }
}
```

- 400 Bad Request（用户名或密码错误）

```json
{
  "message": "wrong username or password, please try again",
  "data": ""
}
```

### `POST /api/auth/logout`（登出）

示例：

```bash
curl -X POST -H 'Content-Type: application/json' \
  -H 'Authorization: bearer <token>' \
  -i 'http://127.0.0.1:8000/api/auth/logout'
```

### `GET /api/address-book`（获取所有联系人）

示例：

```bash
curl -X GET -H 'Content-Type: application/json' \
  -H 'Authorization: bearer <token>' \
  -i 'http://127.0.0.1:8000/api/address-book'
```

响应（200 OK）：

```json
{
  "message": "ok",
  "data": [
    {
      "id": "int32",
      "name": "string",
      "gender": true,     // 男为 true，女为 false
      "age": "int32",
      "address": "string",
      "phone": "string",
      "email": "string"
    }
  ]
}
```

### `GET /api/address-book/{id}`（按 ID 获取联系人）

示例：

```bash
curl -X GET -H 'Content-Type: application/json' \
  -H 'Authorization: bearer <token>' \
  -i 'http://127.0.0.1:8000/api/address-book/2'
```

路径参数：`id: int32`

响应：

- 200 OK

```json
{
  "message": "ok",
  "data": {
    "id": "int32",
    "name": "string",
    "gender": true,     // 男为 true，女为 false
    "age": "int32",
    "address": "string",
    "phone": "string",
    "email": "string"
  }
}
```

- 404 Not Found

```json
{
  "message": "person with id {id} not found",
  "data": ""
}
```

### `GET /api/address-book/filter`（筛选联系人）

示例：

```bash
curl -X GET -H 'Content-Type: application/json' \
  -H 'Authorization: bearer <token>' \
  -i 'http://127.0.0.1:8000/api/address-book/filter?name=foo&sort_by=name&sort_direction=asc&page_num=0&page_size=10'
```

查询参数：

- id: int32
- name: string
- gender: boolean
- age: int32
- address: string
- phone: string
- email: string
- sort_by: string
- sort_direction: string（asc 或 desc）
- page_num: int32
- page_size: int32

响应（200 OK）：

```json
{
  "message": "ok",
  "data": [
    {
      "id": "int32",
      "name": "string",
      "gender": true,
      "age": "int32",
      "address": "string",
      "phone": "string",
      "email": "string"
    }
  ],
  "page_num": "int32",
  "page_size": "int32",
  "total_elements": "int32"
}
```

### `POST /api/address-book`（新增联系人）

示例：

```bash
curl -X POST -H 'Content-Type: application/json' \
  -H 'Authorization: bearer <token>' \
  -i 'http://127.0.0.1:8000/api/address-book' \
  --data '{
    "name": "c",
    "gender": true,
    "age": 32,
    "address": "addr",
    "phone": "133",
    "email": "e@q.com"
  }'
```

请求体：

```json
{
  "name": "string",
  "gender": true,
  "age": 32,
  "address": "string",
  "phone": "string",
  "email": "string"
}
```

响应：

- 201 Created

```json
{
  "message": "ok",
  "data": ""
}
```

- 500 Internal Server Error（插入失败）

```json
{
  "message": "can not insert data",
  "data": ""
}
```

### `PUT /api/address-book/{id}`（更新联系人）

示例：

```bash
curl -X PUT -H 'Content-Type: application/json' \
  -H 'Authorization: bearer <token>' \
  -i 'http://127.0.0.1:8000/api/address-book/2' \
  --data '{
    "name": "b",
    "gender": true,
    "age": 32,
    "address": "addr",
    "phone": "133",
    "email": "b@q.com"
  }'
```

路径参数：`id: int32`

请求体同“新增联系人”。

响应：

- 200 OK

```json
{
  "message": "ok",
  "data": ""
}
```

- 500 Internal Server Error（更新失败）

```json
{
  "message": "can not update data",
  "data": ""
}
```

### `DELETE /api/address-book/{id}`（删除联系人）

示例：

```bash
curl -X DELETE -H 'Content-Type: application/json' \
  -H 'Authorization: bearer <token>' \
  -i 'http://127.0.0.1:8000/api/address-book/2'
```

路径参数：`id: int32`

响应：

- 200 OK

```json
{
  "message": "ok",
  "data": ""
}
```

- 500 Internal Server Error（删除失败）

```json
{
  "message": "can not delete data",
  "data": ""
}
```

## 目录结构与重要文件

- `src/secret.key.sample`：JWT 秘钥示例文件，重命名为 `secret.key` 后使用
- `dotenv.sample`：环境变量示例，重命名为 `.env` 并设置 `DATABASE_URL`
- `migrations/`：Diesel 数据库迁移脚本
- `docker-compose.*.yml`：不同环境的启动配置

## 许可证

本项目使用仓库中所附的 `LICENSE` 文件所述许可协议。
