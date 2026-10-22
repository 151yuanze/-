# 苍穹外卖（本地可复现版）

Spring Boot 3.1.2 + MyBatis + MySQL + Redis 的后端，配套 Vue 管理端和微信小程序用户端。

## 目录结构

```
cangqiongwaimai/
├─ sky-take-out-main/           # 后端（Maven 多模块）
│  └─ sky-take-out-main/
│     ├─ sky-server/            # 启动模块、Controller、配置
│     ├─ sky-pojo/              # 实体、DTO、VO
│     ├─ sky-common/            # 工具类、常量、异常
│     └─ demo/mysql.sql         # 完整数据库脚本（结构 + 初始数据）
├─ project-sky-admin-vue-ts/    # 管理端前端（Vue 2 + TypeScript）
├─ mp-weixin/                   # 用户端微信小程序
└─ nginx-1.24.0/                # 本地静态服务器 + API 反向代理
```

## 端口约定

| 服务 | 地址 |
|---|---|
| 管理端页面 | http://localhost:8088 |
| 后端 API | http://localhost:8080 |
| MySQL | localhost:3306 |
| Redis | localhost:6379 |

## 环境要求

| 依赖 | 版本 |
|---|---|
| JDK | 17 |
| Maven | 3.6+ |
| MySQL | **8.0+**（脚本含 `utf8mb4_0900_ai_ci`，5.7 不支持） |
| Redis | 3.x+（未设密码也可以） |
| Node.js | **16.x**（管理端是老项目，高版本可能编译失败） |
| 微信开发者工具 | 最新稳定版 |

## 1. 初始化数据库

在 MySQL 8 已启动的前提下执行（把 `<项目根目录>` 换成实际路径）：

```powershell
mysql -uroot -p -e "source <项目根目录>/sky-take-out-main/sky-take-out-main/demo/mysql.sql"
```

也可以直接用 DataGrip / Navicat / Workbench 打开 `demo/mysql.sql` 执行。

脚本会自动完成：

- 创建 `sky_take_out` 数据库
- 创建 11 张表
- 写入初始数据（员工、分类、菜品、套餐、用户、订单等）

注意：

- 脚本包含 `DROP TABLE IF EXISTS`，**会删掉同名表后重建**，库里已有数据会丢失
- 默认管理端账号：`admin / 123456`
- `employee.password` 存的是 MD5 值，脚本里已经处理好

导入后可以验证：

```powershell
mysql -uroot -p -e "USE sky_take_out; SELECT COUNT(*) AS dish_count FROM dish; SELECT username, status FROM employee;"
```

## 2. 配置后端敏感信息

进入：

```
sky-take-out-main/sky-take-out-main/sky-server/src/main/resources/
```

复制模板：

```powershell
copy application-env.yml.example application-env.yml
```

然后编辑 `application-env.yml`，至少填写：

- `spring.datasource.password`：MySQL root 密码
- `sky.jwt.admin-secret-key` / `sky.jwt.user-secret-key`：两串足够长的随机字符
- `sky.redis.host` / `port` / `database` / `password`
- `sky.alioss.*`：图片上传用的阿里云 OSS 四项
- `sky.wechat.appid` / `secret`：小程序 AppID 与 AppSecret

`application-env.yml` 已被 `.gitignore` 忽略，不会被提交。

## 3. 启动后端

方式一（推荐）：用 IDEA 打开后端根 `pom.xml`，Project SDK 选择 JDK 17，运行 `com.sky.SkyApplication`。

方式二（命令行）：

```powershell
cd sky-take-out-main/sky-take-out-main
mvn -pl sky-server -am spring-boot:run
```

启动成功后 8080 端口会处于监听状态。接口文档（knife4j）：http://localhost:8080/doc.html

## 4. 构建并启动管理端

### 4.1 构建

必须使用 Node 16。仓库若没有 `node16/` 目录，请先自行安装 Node 16（或使用 nvm）。

```powershell
cd project-sky-admin-vue-ts

# 建议的构建环境变量
$env:NODE_OPTIONS = '--max-old-space-size=4096'
$env:CYPRESS_INSTALL_BINARY = '0'

npm install --legacy-peer-deps --no-audit --no-fund --registry=https://registry.npmmirror.com
npm run build
```

构建产物在 `project-sky-admin-vue-ts/dist/`。

### 4.2 启动 nginx

```powershell
cd nginx-1.24.0
.\nginx.exe -t      # 检查配置
.\nginx.exe         # 启动
```

然后访问 http://localhost:8088。

说明：

- 配置里 `root` 使用相对路径 `../project-sky-admin-vue-ts/dist`，因此请保持仓库目录结构不变
- 端口用 8088 是因为本机 80 端口常被其他服务占用；如需改端口，编辑 `nginx-1.24.0/conf/nginx.conf` 的 `listen`
- 停止 nginx：`.\nginx.exe -s stop`；改配置后重载：`.\nginx.exe -s reload`

### 4.3 WebSocket 地址

`.env.production` 中：

```properties
VUE_APP_SOCKET_URL = 'ws://localhost:8080/ws/'
```

修改后需要重新执行 `npm run build`。浏览器首次访问管理端时，请强制刷新（`Ctrl+Shift+R`），避免 Service Worker 使用旧缓存。

## 5. 运行小程序

1. 用微信开发者工具导入 `mp-weixin` 目录
2. 把 `project.config.json` 里的 `appid` 换成你自己的小程序 AppID（或使用测试号）
3. 确认 `urlCheck` 为 `false`（项目已配置）
4. 后端必须先启动

小程序请求地址写在 `mp-weixin/common/vendor.js`：

```js
var baseUrl = 'http://localhost:8080';
```

如果后端不在本机 8080，需要修改这里。

关于授权弹窗：`project.private.config.json` 里基础库版本是 `3.17.4`。新版基础库下 `wx.getUserProfile` 不再弹出昵称/头像授权框（会返回匿名数据）。如果希望看到授权弹窗，把调试基础库改为 `2.24.2`。

## 6. WebSocket 语音提醒

后端提供 `/ws/{sid}` 端点：

- 支付成功：发送 `{type: 1, ...}`，管理端播放 `preview.mp3`（来单提醒）
- 用户催单：发送 `{type: 2, ...}`，管理端播放 `reminder.mp3`

注意事项：

- 后端必须处于运行状态
- 浏览器有自动播放限制，管理端页面需要先有过点击操作，否则 `play()` 可能被拦截
- 当前管理端直连 `ws://localhost:8080/ws/`，不经过 nginx
- 如果改成同源地址（如 `ws://localhost:8088/ws/`），需要在 nginx 增加 `/ws/` 反向代理，并转发 `Upgrade` / `Connection` 头
- 代码改动后需要重启后端

## 7. 图片上传（阿里云 OSS）

`/admin/common/upload` 使用阿里云 OSS。需要：

1. 在阿里云控制台创建 Bucket
2. 在 `application-env.yml` 填写 `sky.alioss.endpoint`、`access-key-id`、`access-key-secret`、`bucket-name`
3. 建议 Bucket 开启公共读，否则小程序里图片无法直接显示

## 常见问题

**管理端登录提示密码错误**

`employee.password` 必须是 `MD5('123456')` 的结果。使用本仓库的 `demo/mysql.sql` 初始化即可，不需要手动处理。

**启动报 `Could not resolve placeholder 'sky.xxx'`**

没有创建 `application-env.yml`，或少了某个键。参照 `application-env.yml.example` 补齐。

**连不上 Redis**

确认 Redis 已启动、`sky.redis.host/port/password` 正确。Redis 没设密码时 `password` 留空。

**管理端 npm install 失败**

- 确认 Node 版本是 16
- 确认加了 `--legacy-peer-deps`
- 设置 `CYPRESS_INSTALL_BINARY=0`，避免下载 Cypress 二进制

**8088 页面 404 或白屏**

- 确认 `project-sky-admin-vue-ts/dist` 已构建
- 确认 nginx 的 `root` 相对路径与实际目录一致
- 执行 `nginx -t` 检查配置

**授权弹窗不出现**

把微信开发者工具的调试基础库改成 `2.24.2`。
