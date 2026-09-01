# AGT Cloud MaaS

Kubernetes 部署说明。

```
ghcr.io/nacitech/agtcloud-maas-image
```

---

## 前置要求

| 组件 | 要求 | 说明 |
|---|---|---|
| Kubernetes | 1.21+ | |
| **PostgreSQL** | 13+ | **必须外部提供。不支持 MySQL，也不能用 SQLite** |
| Redis | 5.0+ | **必须配** |
| Ingress | ingress-nginx | 或自备 LoadBalancer / NodePort |

数据库**只支持 PostgreSQL**。程序靠 `SQL_DSN` 的 `postgresql://` 前缀识别类型，
填成 MySQL 格式会启动失败。多副本下 SQLite 会数据分裂（单文件数据库，各 Pod 各写各的）。

不配 Redis 则各副本额度不同步，并发请求会击穿额度限制，**余额被扣成负数**。
这两条都不是可选项。

建库即可，表结构由程序自动迁移，不需要导入 SQL：

```sql
CREATE DATABASE new_api ENCODING 'UTF8';
CREATE USER newapi WITH PASSWORD '你的密码';
GRANT ALL PRIVILEGES ON DATABASE new_api TO newapi;

-- PostgreSQL 15+ 还需要这两条，否则建表时报 permission denied for schema public
\c new_api
GRANT ALL ON SCHEMA public TO newapi;
```

---

## 部署

**1. 改配置**

编辑 `deploy.yaml`，替换所有 `CHANGE_ME` 和标了 ⚠️ 的地方：

```bash
openssl rand -hex 32    # SESSION_SECRET
openssl rand -hex 32    # CRYPTO_SECRET（必须与上面不同）
```

| 配置项 | 说明 |
|---|---|
| `SQL_DSN` | PostgreSQL 连接串，**必须以 `postgresql://` 开头** |
| `REDIS_CONN_STRING` | Redis 连接串 |
| `SESSION_SECRET` | 会话密钥，**不能填 `random_string`，程序会拒绝启动** |
| `CRYPTO_SECRET` | 渠道密钥加密密钥 |
| `BATCH_SITE` | 站点标识，**每个站点必须唯一** |
| `FRONTEND_BASE_URL` | 本站点前端地址 |
| `CHANNEL_MODEL_WEIGHT_REDIS_CONN_STRING` | 📮 需要开启动态调度的，请联系管理员获取 Redis 连接信息 |
| Ingress `host` / `tls.hosts` | 真实域名 |

> ⚠️ `SESSION_SECRET` 和 `CRYPTO_SECRET` 定下来就不能再改。
> 改前者所有用户被强制登出；改后者会导致库里已加密的渠道 API Key **全部无法解密**，
> 等同渠道配置丢失，只能手工重填。第一次部署就定好并备份。

**2. 部署**

```bash
kubectl apply -f deploy.yaml
```

**3. 确认**

```bash
kubectl -n new-api get pods -w
```

首次启动 master 要跑数据库迁移，慢一些正常（探针给了 5 分钟窗口）。
slave 会等 master 就绪后自动启动，不用手工控制顺序。

```bash
kubectl -n new-api logs -f deployment/new-api-master
kubectl -n new-api port-forward svc/new-api 3000:3000
curl http://localhost:3000/api/status
```

访问站点，**第一个注册的账号自动成为超级管理员**，请立刻注册。

---

## 架构：为什么是两个 Deployment

`new-api-master`（1 副本）和 `new-api-slave`（N 副本）共用一个 Service，
**都正常承接 API 流量**。master 额外负责：数据库表结构迁移、渠道健康检测与自动启停、
渠道状态快照、动态调度权重同步、异步任务轮询。

两条硬约束：

- **master 必须固定 1 副本**，不要扩容、不要加 HPA。跑多个会重复执行上述任务，
  表现为渠道被反复启停、任务状态错乱。要加机器就加 slave。
- **slave 必须等 master 迁移完成才能启动。** 已由 `wait-for-master` initContainer
  固化，部署和升级都不用手工干预。

---

## 运维

**升级**（⚠️ 必须先 master 后 slave，迁移只在 master 执行）

把下面的 `v1.1.0` 换成我们通知你的新版本号：

```bash
kubectl -n new-api set image deployment/new-api-master new-api=ghcr.io/nacitech/agtcloud-maas-image:v1.1.0
kubectl -n new-api rollout status deployment/new-api-master

kubectl -n new-api set image deployment/new-api-slave \
  wait-for-master=ghcr.io/nacitech/agtcloud-maas-image:v1.1.0 \
  new-api=ghcr.io/nacitech/agtcloud-maas-image:v1.1.0
kubectl -n new-api rollout status deployment/new-api-slave
```

> slave 有两个容器（`wait-for-master` 初始化容器和 `new-api` 主容器）都用这个镜像，
> 两个要一起改，否则 initContainer 还停在旧版本。

也可以改 `deploy.yaml` 里的三处 `image:` 再 `kubectl apply -f deploy.yaml`，
效果一样，还能让文件和线上保持一致，推荐这种。

> ⚠️ **镜像标签固定为具体版本，不要改成 `:latest`。**
> 文件里 `imagePullPolicy: IfNotPresent`，节点缓存过 `:latest` 后就不会再拉新的，
> 你以为升级了其实没有，而且没有任何报错。用确定版本号才能保证升级真的生效，
> 回滚也才有意义。

> 升级前先备份数据库。迁移自动执行但不可逆。
> master 是 `Recreate` 策略，升级期间定时任务暂停几十秒，API 流量由 slave 承接，不中断。

**回滚**

```bash
kubectl -n new-api rollout undo deployment/new-api-master
kubectl -n new-api rollout undo deployment/new-api-slave
```

> ⚠️ 只回滚代码，**回滚不了数据库 schema**。跨大版本回滚前先确认迁移是否引入不兼容改动。

**扩容**

```bash
kubectl -n new-api scale deployment/new-api-slave --replicas=5
```

> ⚠️ 扩容前先算：`SQL_MAX_OPEN_CONNS × (副本数 + 1)` 必须小于数据库 `max_connections`。
> 按默认 200 跑 5 个 slave 就是 `200 × 6 = 1200`。

**看日志**（只输出 stdout，容器内不落盘）

```bash
kubectl -n new-api logs -f deployment/new-api-master
kubectl -n new-api logs -f deployment/new-api-slave --all-containers
```

---

## 排障

**Pod CrashLoopBackOff，日志有 `Please set SESSION_SECRET to a random string`**
`SESSION_SECRET` 填成了字面量 `random_string`，换成 `openssl rand -hex 32` 生成的随机串。

**slave 一直卡在 Init**
initContainer 在等 master。看 `kubectl -n new-api logs deployment/new-api-master`，
多半是数据库或 Redis 连不上，或迁移还没跑完。

**回答只出来一半就断 / 没有打字机效果**
入口的缓冲和超时没配对。Ingress 用户检查 `deploy.yaml` 里那几条注解是否被改动；
用 LoadBalancer 的要在云厂商控制台调空闲超时（默认 60s 或 300s 会截断流式响应）；
用 NodePort 的要在前置 Nginx 上配 `proxy_buffering off` 和 `proxy_read_timeout 3600s`。

**日志刷 `too many connections`**
`SQL_MAX_OPEN_CONNS × 副本数` 超过了数据库 `max_connections`。调小前者或调大后者。

**用户余额被扣成负数**
Redis 没配或连不上，各副本额度不同步。检查 `REDIS_CONN_STRING`。

**动态调度不生效，但也没报错**
`BATCH_SITE` 填错了。程序读 `availability:weights:<BATCH_SITE>`，读到空结果会
**静默返回，不报错也不打日志**。确认日志里出现过
`channel model weight sync completed`，这是唯一能确认它真在跑的信号。

**镜像拉取失败 `denied`**
镜像包未设为 public，联系我们处理。
