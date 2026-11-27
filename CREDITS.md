# CREDITS 致谢

本项目现名 **建木（angela-jianmu）**（《山海经·海内经》通天之木，众帝借以上下天地；2026-11-26 定名），自有化自 **fscarmen/sing-box**（Sing-box 全家桶一键多协议部署脚本）：

- 原作者：fscarmen
- 上游仓库：https://github.com/fscarmen/sing-box
- 收编基线：v1.3.31（上游 HEAD 7190fd2，2026-11-26 fork）
- 许可证：GNU GPL v3.0 — 本项目完整保留 [LICENSE](LICENSE) 全文与原版权声明，衍生分发继续以 GPL-3.0 发布，不更换许可证。

## 上游 README 鸣谢名单（随自有化保留）

- 千歌 sing-box 模板：https://github.com/chika0801/sing-box-examples
- zmlu 的 Cloudflare Tunnel 管理脚本（Argo API 创建隧道方法来源）：https://raw.githubusercontent.com/zmlu/sba/main/tunnel.sh

## 依赖与引用

- sing-box 内核：SagerNet/sing-box（GPL-3.0）
- cloudflared：Cloudflare 官方 Release
- jq：jqlang/jq 官方 Release
- qrencode-go 与订阅模板：fscarmen/client_template
- 规则集：SagerNet/sing-geosite、MetaCubeX/meta-rules-dat
- WARP 注册：原 fscarmen/warp（GitLab）的远程 api.sh 方法已由本地实现替代（openssl 本机生成密钥对、只向 Cloudflare 官方接口注册），不再执行任何远程脚本，见 docs/AUDIT.md 疑点 E
- s6-overlay（Docker 镜像）：just-containers/s6-overlay

## 自有化剥离说明

按自有化口径，README 中厂商赞助推广内容（原「感谢赞助商」节：SharonNetworks、vps.town）已剥离；该节属推广版位、非致谢，本文件致谢名单不受影响。安全审计与剥离全记录见 [docs/AUDIT.md](docs/AUDIT.md)。
