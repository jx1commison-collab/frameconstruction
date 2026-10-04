# 卓爱普业务平台 · ZAP-Platform / ERPNext 部署说明

## 位置
- 宿主：zapvm（R730 内的 KVM/libvirt 虚机，NAT IP: `192.168.122.239`，经 R730 转发）
- 代码：`~/zap-platform/frappe_docker`（官方 https://github.com/frappe/frappe_docker 的浅克隆，未做任何本地修改）
- 项目名（docker compose -p）：`zap`

## Docker 镜像源（daemon.json）
zapvm 访问 Docker Hub registry (`registry-1.docker.io`) 在当前网络下连接超时，已在 `/etc/docker/daemon.json` 配置国内镜像源，dockerd 会按顺序尝试，失败再退回下一个：

```json
{
  "registry-mirrors": ["https://docker.m.daocloud.io", "https://docker.1panel.live"]
}
```

修改后需要 `sudo systemctl restart docker` 生效。若两个镜像源都失效，可追加其他可用源（如 `docker.1ms.run`、`hub.rat.dev` 等，需现测连通性）到该数组后重启 docker。

## ERPNext 版本
- `frappe/erpnext:v16.37.0`（Docker Hub 上 `v16`/`version-16` 系列当前最新稳定版，2026-10-03 发布）
- 通过 `.env` 里的 `ERPNEXT_VERSION` 控制，升级时改这个值并重新 `docker compose config` 生成 compose 文件即可。

## Compose 组成
最终运行的 `compose.custom.yaml` 由官方 base + 3 个 override 合并生成（**不含反向代理**，直接发布 8080 端口）：

```bash
cd ~/zap-platform/frappe_docker
docker compose --env-file .env \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.noproxy.yaml \
  config > compose.custom.yaml

sudo docker compose -p zap -f compose.custom.yaml up -d
```

容器清单：`configurator`(一次性)、`backend`、`frontend`(发布 `8080:8080`)、`websocket`、`queue-short`、`queue-long`、`scheduler`、`db`(MariaDB 11.8)、`redis-cache`、`redis-queue`。

## .env 模板（真实文件在 zapvm 上，权限 600，不提交、不外传；此处只列字段，密码留空）

```env
ERPNEXT_VERSION=v16.37.0
DB_PASSWORD=                    # MariaDB root 密码，随机生成，仅本机 .env 内，600 权限
FRAPPE_SITE_NAME_HEADER=zap.local   # 通过 IP/隧道(非域名)访问时，告诉 nginx 用哪个站点
# 其余变量保持 example.env 默认（外部数据库/Redis、反代、GUNICORN 等均未启用）
```

> 管理员（Administrator）登录密码**不**在 `.env` 或任何 compose/脚本中出现，由运维人员在创建站点时通过交互输入，详见下方“创建站点”。

## 创建站点 zap.local（需手动执行，见下）
containers 起来后，`db` healthy、`configurator` 跑完退出即可创建站点。命令见对话里给出的、读取 `.env` 里 DB_PASSWORD 并用 `read -s` 交互输入 Administrator 密码的版本——密码不会落地到任何文件或 shell 历史。

**坑：`bench new-site` 必须显式传 `--db-root-username root`。**
`--db-root-password` 单独给了不够——`new-site` 的 root 用户名参数（`db_root_username`）不传时，frappe 会在 `get_root_connection()` 里卡在交互提示 `Enter mysql super user [root]:`（如果 exec 分配了 TTY，会真的阻塞等输入；Ctrl-C 后报 `Aborted!`）。每次这样中断，`make_site_dirs()`/`make_conf()` 已经执行过，会在 `sites/<site>/` 留下一个指向从未真正建出来的 MariaDB 用户的半成品 `site_config.json`（残留目录要 `rm -rf sites/<site>` 清掉，不会自动清理；对应的 MariaDB 用户/库那一步从未跑到，通常不需要额外清）。
正确姿势：
- 用 `docker compose exec -T`（非交互，不分配 TTY）执行 bench 命令，从源头避免触发该 prompt；
- 同时显式传 `--db-root-username root --db-root-password "$DBPW"`，不要只给密码；
- 首次建站带上 `--force`，避免因为前序失败残留的空目录又触发“Site already exists”。

## 访问方式（无反向代理，仅局域网/隧道）
1. X1 上：`ssh -L 8080:192.168.122.239:8080 wangyusong@192.168.1.10`
2. 浏览器打开 `http://127.0.0.1:8080`
3. 因为是按 IP/隧道访问（非 `zap.local` 域名解析），nginx 靠 `FRAPPE_SITE_NAME_HEADER=zap.local` 固定路由到该站点。

## 后续升级/迁移站点
参考官方文档 `docs/02-setup/06-setup-examples.md` 的 “Updating Images” 一节：改 `.env` 里 `ERPNEXT_VERSION` → 重新 `docker compose config` 生成 yaml → `pull` → `down` → `up -d` → `bench --site zap.local migrate`。

## 范围边界
本部署仅涉及 zapvm 内的 `zap` compose 项目。未触碰、未修改：
- ZAP-CRM-Return / crm-return-test
- ZAP-Langfuse / zap-langfuse-net
- R730 宿主机（未装 Docker）
