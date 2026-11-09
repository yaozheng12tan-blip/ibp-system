# RuoYi-Cloud Docker Compose 部署方案

> 文档编号：01
> 适用项目：ibp-system（基于 RuoYi-Cloud 3.6.8，master 分支 / Spring Boot 4.x）
> 适用场景：Windows 本地开发 → Linux 单机服务器迁移，前后端同库，单人/小团队维护
> 编写日期：2026-11-09
> 参考来源：RuoYi 官方文档（环境部署、应用容器部署）、社区 ECS 全家桶部署实践

---

## 1. 总体架构

采用单机 Docker Compose 全容器化部署。一台机器、一份 `docker-compose.yml`，本地 Windows 与 Linux 服务器运行完全相同的编排，环境差异由容器屏蔽。

```
宿主机 (Windows Docker Desktop / Linux Docker Engine)
└── 容器网络 ruoyi_default（容器间通过服务名互通）
    ├── ruoyi-nginx        :80    前端入口，唯一对宿主机暴露的服务
    ├── ruoyi-gateway      :8080  网关
    ├── ruoyi-auth         :9200  认证中心
    ├── ruoyi-system       :9201  系统模块
    ├── ruoyi-gen          :9202  代码生成（可选）
    ├── ruoyi-job          :9203  定时任务（可选）
    ├── ruoyi-file         :9300  文件服务（可选）
    ├── ruoyi-monitor      :9100  监控中心（可选）
    ├── ruoyi-nacos        :8848  注册中心 / 配置中心
    ├── ruoyi-redis        :6379  缓存
    └── ruoyi-mysql        :3306  ry-cloud 业务库 + ry-config 配置库
         ↑ 数据全部落在宿主机数据卷，容器可销毁重建
```

### 1.1 服务分级

- **必选服务**：mysql、redis、nacos、gateway、auth、system、nginx
- **可选服务**：gen、job、file、monitor，按需启动
- **不安装**：Sentinel 控制台（规则可持久化到 Nacos，不影响运行）、Seata（业务模块未引用依赖）

### 1.2 端口清单

| 端口 | 服务 | 用途 |
| --- | --- | --- |
| 80 | nginx | 前端访问入口 |
| 8080 | gateway | 网关 |
| 9200 | auth | 认证中心 |
| 9201 | system | 系统模块 |
| 9202 | gen | 代码生成 |
| 9203 | job | 定时任务 |
| 9300 | file | 文件服务 |
| 9100 | monitor | 监控中心 |
| 8848 / 9848 / 9849 | nacos | HTTP 控制台 / gRPC 通信端口 |
| 6379 | redis | 缓存 |
| 3306 | mysql | 数据库 |

---

## 2. 版本矩阵

所有镜像锁定具体版本，禁止使用 `latest`，避免镜像漂移导致环境不一致。

| 组件 | 仓库现状 | 锁定版本 | 说明 |
| --- | --- | --- | --- |
| JDK（本机构建） | Corretto 17.0.21 | JDK 17 | 位于 `C:\Users\orford\.jdks\corretto-17.0.21`，需配置 JAVA_HOME |
| 应用基础镜像 | `openjdk:17` | `eclipse-temurin:17-jre` | 官方 openjdk 镜像已停止维护，统一替换为 Temurin，需改 7 个 Dockerfile |
| Nacos | `nacos/nacos-server:v3.0.2` | `nacos/nacos-server:v3.0.2` | 与 Spring Boot 4.x 分支匹配 |
| MySQL | `mysql:5.7` | `mysql:8.0.46` | 8.0 LTS 主力版本；5.7 已于 2023-10 EOL，全新部署采用 8.0，详见 2.1 |
| Redis | `redis`（latest） | `redis:7-alpine` | 锁定大版本，体积小 |
| Nginx | `nginx`（latest） | `nginx:1.27-alpine` | 锁定稳定版 |
| Node（构建机） | v22.23.2 | v22.x | 仅本地构建使用，不安装到服务器 |
| Maven（构建机） | 无命令行 | 3.8+ | 独立安装或使用 IDEA 内置 Maven |

### 2.1 MySQL 8.0 适配说明

官方要求 MySQL >= 5.7，本项目锁定 8.0.46。兼容性已确认：Nacos 自 2.2 版本起支持 MySQL 8.0（3.0.2 无问题），Spring Boot 4.1.0 自带 `mysql-connector-j` 新驱动原生支持。

相对于 5.7 的关键差异与处理方式：

| 差异点 | 说明与处理 |
| --- | --- |
| `lower_case_table_names` 初始化时机 | 8.0 要求该参数必须在数据目录**首次初始化时**设置，已有数据后修改会被拒绝启动。compose 已在 command 中保留 `--lower-case-table-names=1`，空数据卷首次启动即生效，**禁止中途修改该参数** |
| 默认认证插件 | 8.0 默认 `caching_sha2_password`（替代旧的 `mysql_native_password`），新驱动默认支持，无需回退，安全性更高 |
| 默认排序规则 | 8.0 默认 `utf8mb4_0900_ai_ci`（5.7 为 `utf8mb4_unicode_ci`），compose 已在 command 中显式指定，保证环境行为一致 |
| 新增保留字 | 8.0 新增 `RANK`、`GROUPS`、`SYSTEM` 等保留字，二次开发建表/字段撞名时需加反引号 |
| 整数显示宽度 | `bigint(20)` 等显示宽度被弃用，仅产生 warning，不影响功能 |
| 8.0 新特性 | 窗口函数、CTE 公共表达式、JSON 增强、原子 DDL，可直接用于报表与分析 SQL |

---

## 3. 目录规划

```
F:\RuoYi-Cloud-master\
├── doc\                         项目文档（本目录，后续方案均放这里）
├── ruoyi-ui\                    前端工程（Vue3，需单独下载放入，删除自带 .git）
├── ruoyi-gateway\               网关模块源码
├── ruoyi-auth\                  认证模块源码
├── ruoyi-api\                   接口模块源码
├── ruoyi-common\                通用模块源码
├── ruoyi-modules\               业务模块源码（system/gen/job/file）
├── ruoyi-visual\                图形化模块源码（monitor）
├── docker\                      Docker 编排与镜像构建目录
│   ├── mysql\                   MySQL 镜像（db 下存放初始化 SQL）
│   ├── redis\                   Redis 镜像
│   ├── nacos\                   Nacos 镜像
│   ├── nginx\                   Nginx 镜像（html/dist 存放前端产物）
│   ├── ruoyi\                   业务模块镜像（各模块 jar 目录）
│   ├── copy.sh                  产物拷贝脚本
│   ├── deploy.sh                部署控制脚本
│   └── docker-compose.yml       容器编排文件
├── sql\                         数据库脚本
│   ├── ry_20260417.sql          业务库 ry-cloud 表结构（必选）
│   ├── quartz.sql               Quartz 定时任务表（可选）
│   ├── ry_config_20260918.sql   配置库 ry-config（必选，含建库语句与全部 Nacos 配置）
│   └── ry_seata_20210128.sql    Seata 脚本（当前不使用）
├── bin\                         本地打包/启动脚本
├── pom.xml                      Maven 公共配置
└── README.md
```

### 3.1 数据库说明

| 库名 | 用途 | 初始化脚本 |
| --- | --- | --- |
| `ry-cloud` | 业务库，存放系统与业务数据 | `ry_20260417.sql` + `quartz.sql` |
| `ry-config` | Nacos 外置配置库，存放 8 份配置（`application-dev.yml` + 7 份服务配置） | `ry_config_20260918.sql`（自带建库语句） |

Nacos 配置中的默认连接参数（部署时按实际环境修改）：

- MySQL：`localhost:3306/ry-cloud`，用户 `root`，密码 `123456`
- Redis：`localhost:6379`，无密码
- MinIO：`minioadmin/minioadmin`，bucket `test`

---

## 4. 分阶段实施

### 阶段 0：修复仓库问题（不修复必踩坑）

1. **`copy.sh` 文件名过期**：脚本中引用的 `ry_20260402.sql`、`ry_config_20260311.sql` 与仓库实际文件不符，改为：
   - `ry_20260417.sql`
   - `ry_config_20260918.sql`
2. **密码统一**：`docker-compose.yml` 中 MySQL root 密码为 `password`，而 Nacos 业务配置中为 `123456`，统一为一套强密码，同步修改：
   - `docker/nacos/conf/application.properties`
   - Nacos 配置内容（`ry_config` 脚本中的 `application-dev.yml`）
3. **补 `.gitignore`**，追加：

   ```gitignore
   # 前端依赖
   node_modules/
   # 前端本地环境变量与日志
   npm-debug.log*
   yarn-error.log*
   .env.local
   .env.*.local
   ```

4. **放入前端工程**：下载 RuoYi-Cloud-Vue3 解压到 `F:\RuoYi-Cloud-master\ruoyi-ui`，删除目录自带的 `.git`，避免嵌套 Git 仓库导致代码无法提交。

### 阶段 1：本地启动基础三件套

安装 Docker Desktop（WSL2 后端），在 `docker/` 目录执行：

```powershell
docker compose up -d ruoyi-mysql ruoyi-redis ruoyi-nacos
```

MySQL 镜像通过 `ADD ./db/*.sql /docker-entrypoint-initdb.d/` 在首次启动时自动初始化：

- `ry-cloud` 由 compose 中 `MYSQL_DATABASE` 自动创建
- `ry-config` 由 `ry_config` 脚本自带语句创建

注意事项：

- `depends_on` 只保证容器启动顺序，不保证 MySQL 就绪，Nacos 首次可能连库失败后自动重启，观察日志等待自愈即可
- Nacos 3.x 需同时放行 8848、9848、9849 三个端口
- MySQL 8.0 首次启动初始化时，`lower_case_table_names=1` 与排序规则 `utf8mb4_0900_ai_ci` 由 command 参数在初始化阶段写入；若已产生过数据卷，需先清空 `docker/mysql/data` 再启动，否则参数不生效会导致容器启动失败

### 阶段 2：构建产物

后端打包（项目根目录）：

```powershell
mvn clean package -Dmaven.test.skip=true
```

前端打包（`ruoyi-ui` 目录）：

```powershell
npm install --registry=https://registry.npmmirror.com
npm run build:prod
```

执行 `docker/copy.sh`（或等价手动拷贝）：

- 各模块 jar 拷贝到 `docker/ruoyi/*/jar/`
- 前端 dist 拷贝到 `docker/nginx/html/dist/`

服务器只接收构建产物，不安装 JDK、Maven、Node。

### 阶段 3：本地全量启动并验证

```powershell
docker compose up -d --build
docker compose ps
docker compose logs -f ruoyi-gateway
```

验证清单：

- 所有容器处于 running / healthy 状态
- 浏览器访问 `http://localhost`，使用 `admin/admin123` 登录
- 验证菜单加载、定时任务、文件上传等核心功能

### 阶段 4：迁移 Linux 服务器

1. 服务器安装 Docker Engine 与 compose 插件，配置阿里云镜像加速；
2. 将整个 `docker/` 目录（含 jar、dist、SQL）打包上传，同一份 compose 文件直接使用；
3. 启动：

   ```bash
   docker compose up -d
   ```

4. 代码侧通过 Git 同步：

   ```bash
   git pull
   ```

   本地重新构建产物后上传，重建对应容器，迁移完成。

---

## 5. 生产环境加固清单

| 项目 | 加固措施 |
| --- | --- |
| 端口收敛 | 仅 Nginx 80 映射到宿主机；3306、6379、8848 不映射公网，仅容器网络内通信（可用单独的 `docker-compose.prod.yml` 覆盖） |
| 重启策略 | 所有服务配置 `restart: always`（或 `unless-stopped`），宿主机重启后自动拉起 |
| 密码 | 替换全部框架默认值（MySQL root、MinIO、Nacos token）；Redis 内网可不开密码，公网暴露必须开启 |
| 防火墙 | 只放行 80/443，不照搬 `deploy.sh port` 的全量开放策略 |
| JVM 参数 | 每个 Java 服务配置 `JAVA_OPTS=-Xms256m -Xmx512m`，防止单机多容器内存打满 |
| 时区 | compose 各服务增加 `TZ=Asia/Shanghai` |
| 数据备份 | 定期 `mysqldump` 备份两个库，备份 `ruoyi/uploadPath` 上传目录；Redis 缓存可清库重建 |
| actuator | Nginx 已配置 403 拦截，保持现状 |

---

## 6. 日常运维

### 6.1 更新代码

- 改 Java 代码：

  ```powershell
  mvn clean package -Dmaven.test.skip=true
  # 上传新 jar 后只重建对应服务
  docker compose up -d --build ruoyi-modules-system
  ```

- 改前端代码：

  ```powershell
  npm run build:prod
  # 覆盖 dist 后重启 Nginx
  docker compose restart ruoyi-nginx
  ```

- 改配置：直接在 Nacos 控制台修改，服务自动刷新，无需重建镜像

### 6.2 常用排障命令

```bash
# 查看服务状态
docker compose ps

# 查看服务日志
docker compose logs -f <服务名>

# 进入 MySQL 容器
docker compose exec ruoyi-mysql mysql -uroot -p

# 进入容器终端
docker compose exec <服务名> bash

# 停止 / 启动全部服务
docker compose stop
docker compose start
```

### 6.3 Git 提交规范

```powershell
git add -A
git commit -m "feat: 描述本次改动"
git push
```

---

## 7. 明确不做的事项

以下内容在单人单机场景下属于过度设计，暂不实施；待业务量与团队扩张后再演进，compose 编排可平滑替换为 K8s 清单，无需推倒重来：

- Kubernetes / 集群编排
- Nacos 集群
- MySQL 主从
- Sentinel 控制台
- Seata 分布式事务
- CI/CD 流水线

---

## 8. 待办跟踪

| 序号 | 事项 | 状态 |
| --- | --- | --- |
| 1 | MySQL 升级至 8.0.46（compose 镜像、Dockerfile、排序规则与初始化参数适配） | 已完成 |
| 2 | 修复 `copy.sh` 中过期的 SQL 文件名 | 待执行 |
| 3 | 统一 MySQL 密码（compose 与 Nacos 配置） | 待执行 |
| 4 | 补 `.gitignore`（node_modules 等） | 待执行 |
| 5 | 下载 RuoYi-Cloud-Vue3 放入 `ruoyi-ui` 并删除 `.git` | 待执行 |
| 6 | 替换 7 个应用 Dockerfile 基础镜像为 `eclipse-temurin:17-jre` | 待执行 |
| 7 | 本地 Docker Desktop 部署验证 | 待执行 |
| 8 | Linux 服务器迁移验证 | 待执行 |
