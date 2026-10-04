# frameconstruction

卓爱普业务平台基础设施 / 部署文档仓库。

## 阶段 0：基础环境搭建

- 步骤1-2：ZAP-Platform 虚机（zapvm）创建、R730 → zapvm 免密 SSH。
- 步骤3：zapvm 内安装 Docker Engine + Compose 插件（官方源），配置国内 registry mirror。
- 步骤4：zapvm 内用官方 frappe_docker 部署 ERPNext（站点 `zap.local`），详见 [docs/erpnext-deploy-readme.md](docs/erpnext-deploy-readme.md)。
- 步骤5：本仓库接入 GitHub，建立版本管理起点（本次提交）。

## 范围边界

本仓库及上述部署均未涉及：ZAP-CRM-Return / crm-return-test、ZAP-Langfuse / zap-langfuse-net、R730 宿主机本身（未装 Docker）。
