# 安全审计报告 — fscarmen/sing-box 自有化（静态审计）

- 审计对象：dukangalex/sing-box-1（fork 自 fscarmen/sing-box），基线 HEAD 7190fd2（上游 v1.3.31）
- 审计日期：2026-11-26
- 审计方式：静态只读通读 + 全文文本检索（URL / curl / wget 出站逐项捞取核对）
- **未做安装实测**：未在任何机器上运行安装（会改防火墙 / 装服务）；如需实测，只能用一次性容器或闲置机器并先报批
- 许可证：GPL-3.0，LICENSE 全文保留，衍生分发继续 GPL-3.0

## 结论（先说要紧的）

脚本主体功能与宣称一致：多协议入站生成、多客户端订阅输出、Argo 隧道、WARP 解锁分流，未发现把节点凭据批量回传作者服务器的后门式代码。

原有 **两处与部署功能无关或削弱隐私的外发**（疑点 A、B）。已按主聊天 2026-11-26 定夺（依全项目基础章程隐私底线）**处置完毕**：

1. **运行次数统计（遥测）** — 已整段移除，提交 7595fce
2. **Reality 私钥外发兜底** — 已删除外发路径，提交 dcd8f2e

另有三项已知风险（下载无校验和、经第三方 GitHub 代理下载、菜单内远程脚本直跑），属上游设计取舍，逐项列明，自有化迭代时按优先级收。

## 疑点清单

### A.【已处置，提交 7595fce】运行统计回传作者统计服务
- sing-box.sh:643-652 `statistics_of_run_times()`；主入口 sing-box.sh:7751 每次运行脚本都执行 `update`（注释明示该行必须保留）；docker_init.sh:1400 容器版同理。
- 行为：向 `https://stat.cloudflare.now.cc/updateStats?script=<脚本名>` 发 GET，取回当天 / 累计运行次数并展示。域名为作者自有服务（*.cloudflare.now.cc）。
- 外发内容：脚本名、访问时间、服务器公网 IP（HTTP 请求天然携带）。不带 UUID、节点配置等凭据。
- 定性：与部署功能无关的遥测，撞自有化隐私底线（遥测组件不收编）。
- 处置（2026-11-26）：函数本体、主入口 update 调用、节点清单 get 调用、docker_init.sh 容器版统计块已全部删除；grep 复查 stat.cloudflare.now.cc / statistics_of_run_times / updateStats 三词在两个脚本中零残留（语言表里的「运行次数」字样为无调用的 inert 文案，不构成外发）；bash -n 复验通过。

### B.【已处置，提交 dcd8f2e】自定义 Reality 私钥在兜底路径下外发第三方
- sing-box.sh:5127：当用户指定了 Reality 私钥、且本机**没有 xxd 命令**时，脚本把私钥以 GET 参数发往 `https://realitykey.cloudflare.now.cc/?privateKey=$REALITY_PRIVATE`，让作者的在线服务算出公钥。
- 正常路径（有 xxd）是本地用 openssl 派生公钥，不外发；docker_init.sh 亦全程本地派生。
- 定性：私钥离开本机且走 URL 查询串（易被日志留存），属凭据级外发，不可接受。
- 处置（2026-11-26）：兜底分支已删除。无 xxd 时改为调用脚本既有 error() 报错退出（中英双语），明示原因（派生公钥需要 xxd、私钥不会发往任何在线服务）与 remedy（安装 xxd 后重试，或不指定 REALITY_PRIVATE 让脚本本地生成新密钥对）；不静默降级、不换用其他在线服务。自动生成密钥对的本地流程（generate_reality_keypair）未动。grep 复查 realitykey 零残留，bash -n 复验通过。

### C.【已处置，见加固台账】下载的二进制无完整性校验，且大量使用 --no-check-certificate
- sing-box、jq、qrencode、cloudflared 均直接下载解压 / 执行，全仓无 sha256 / 签名校验（grep 实证）；`--no-check-certificate` 在 sing-box.sh 出现 37 处、docker_init.sh 10 处。
- 风险：下载通道被劫持时无从发现。自有化迭代建议：对官方 Release 补 sha256 对照（官方 releases 附带校验文件时）。

### D.【已处置，见加固台账】GitHub 下载经第三方反代加速节点
- sing-box.sh:7 `GITHUB_PROXY=('https://ghproxy.lvedong.eu.org/' 'https://accel.cloudflare.nyc.mn/')`，sing-box.sh:26 另有优选 IP 探测 `ip.cloudflare.now.cc / ip.cloudflare.nyc.mn`（作者自有服务）。
- 脚本会探测并选用可用代理前缀下载二进制与订阅模板（sing-box.sh:588-603）。经第三方代理下载 + 无校验和（疑点 C）叠加，风险更高。建议：自有化版默认直连官方源，代理改为用户显式可选配置。

### E.菜单 / 工具箱内的远程脚本直接执行（用户手动触发，非静默）
- `sb -b` 与工具箱菜单：`bash <(wget... ylx2016/Linux-NetSpeed/tcpx.sh)`（内核 / BBR）、`bash <(wget... fscarmen/argox/argox.sh)`、`bash <(wget... fscarmen/sba/sba.sh)`、`bash <(wget https://tcp.hy2.sh/)`。
- WARP 账号注册：`bash <(wget https://gitlab.com/fscarmen/warp/-/raw/main/api.sh) --register`（sing-box.sh:2051、2535），属功能性（向 Cloudflare 注册 WARP 账户），但执行的是远程代码。
- 定性：均为用户在菜单里主动选才触发，不是后台静默执行；但自有化口径下应逐项标注来源、能本地化的本地化。

### F.【已处置，提交 4956e9a】订阅二维码在线链接（仅印在节点清单里，脚本不自动访问）
- 节点清单文本里印有 `https://api.qrserver.com/v1/create-qr-code/?...&data=<订阅URL>`（sing-box.sh:6868-6871 等、docker_init.sh:1300）。
- 脚本本身不请求该地址；但用户点开链接时，含 UUID 的订阅地址会发给 qrserver.com。清单里同时给出了本地 qrencode 生成的二维码，建议自有化版只留本地二维码、删在线链接行。
- 处置（2026-11-26，用户亲令本项目数据安全零妥协后加固）：sing-box.sh 与 docker_init.sh 中 6 处 qrserver 链接已全部删除，只保留本地 qrencode 生成的二维码；grep 复查 qrserver 零残留，bash -n 通过。

### G.Docker 版写死一把共享 WARP 私钥
- docker_init.sh 生成的 02_endpoints.json 里 WARP endpoint 使用固定 private_key（上游公开的共享兜底账户，README 更新日志 2026.08 v1.3.21 提及共享密钥兜底）。
- 不是用户数据外泄，但所有用默认值的部署共用同一 WARP 身份；VPS 版有预注册独立账户逻辑，Docker 版没有。

## 全部外部出站清单（按用途归类）

### 下载源（二进制 / 模板 / 规则集）
| 来源 | 内容 | 备注 |
|---|---|---|
| github.com/SagerNet/sing-box/releases（+ releases.atom、api.github.com 同仓） | sing-box 内核二进制、版本检测 | 官方源；可经疑点 D 的代理前缀 |
| github.com/cloudflare/cloudflared/releases | cloudflared 二进制、版本检测 | 官方源；同上 |
| github.com/jqlang/jq/releases（jq-1.7.1） | jq 二进制 | 官方源 |
| github.com/fscarmen/client_template（raw） | qrencode-go 二进制、全部订阅模板（SUBSCRIBE_TEMPLATE） | 作者仓库，功能必需 |
| raw.githubusercontent.com/fscarmen/sing-box/main/force_version | 上游强制指定内核版本 | 作者仓库 |
| raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/*.srs、api.github.com/SagerNet/sing-geosite | geosite 规则集（含 geosite-openai.srs） | 官方源，运行时由 sing-box 拉取 |
| api.github.com/MetaCubeX/meta-rules-dat、raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/* | 自定义路由规则集 | 用户启用自定义规则时 |
| github.com/just-containers/s6-overlay/releases（Dockerfile） | s6-overlay | 仅 Docker 构建阶段 |
| gitlab.com/fscarmen/warp/-/raw/main/api.sh | WARP 注册脚本 | 见疑点 E |

### 功能性外联（用户配置触发或协议必需）
| 目标 | 用途 | 携带数据 |
|---|---|---|
| api.cloudflare.com/client/v4/* | 用户提供 Cloudflare API 令牌时建隧道、配 DNS | 用户自己的令牌（发往 Cloudflare 官方） |
| dash.cloudflare.com | 仅提示文案里的链接 | 无 |
| fscarmen.cloudflare.now.cc | Argo Json 获取的作者自有站（提示文案引导用户去该站生成 Json） | 用户在该站页面上的操作；脚本本身不调用。自有化时改中立表述 |
| api.openai.com/compliance/cookie_requirements、ios.chat.openai.com、platform.openai.com | ChatGPT 解锁探测，决定 OpenAI 流量直连还是走 WARP | 浏览器伪装头 + Bearer null，不带节点凭据 |
| realm.hy2.io + STUN（turn.cloudflare.com、stun.nextcloud.com、stun.sip.us、global.stun.twilio.com） | 仅启用 Hysteria2 Realm 时的打洞协调 | realm_id（由节点 UUID 派生） |
| engage.cloudflareclient.com（WARP endpoint） | WARP 出站落地 | 见疑点 G |
| time.apple.com（NTP）、151.101.1.91 / 2a04:4e42:200::347（ping） | 时间同步、IPv4/IPv6 可用性探测 | 无业务数据 |
| localhost / 127.0.0.1 各端口 | cloudflared quicktunnel 指标、订阅与 API 本地回环 | 本机回环 |

### 与功能无关的外发
| 目标 | 用途 | 处置 |
|---|---|---|
| stat.cloudflare.now.cc/updateStats | 运行次数统计 | 疑点 A，已移除（7595fce） |
| realitykey.cloudflare.now.cc（兜底） | 在线算 Reality 公钥 | 疑点 B，已移除兜底（dcd8f2e） |
| api.qrserver.com（清单链接） | 在线二维码图 | 疑点 F，建议删链接行 |

## 数据安全加固台账（2026-11-26，用户亲令「必须 100% 保证用户数据安全」后立）

口径：与凭据、订阅、节点配置沾边的外发零容忍——已清的：疑点 A 遥测、疑点 B 私钥外发、疑点 F 订阅地址在线二维码、脚本提示文案引导用户去作者自有站生成 Argo 凭据（已改官方控制台 / 本地 cloudflared 表述，提交 b21295f）。未清的四项完整性风险及处置方向：

| 项 | 风险 | 处置方向 | 状态 |
|---|---|---|---|
| C | 下载的 sing-box / cloudflared / jq / qrencode 二进制无完整性校验，--no-check-certificate 共 47 处 | 已落地：新增 gh_release_download / gh_file_download 两道校验门——Release 资产用 GitHub 官方 API 的 sha256 digest 校验，仓库文件（模板、qrencode、force_version）用 git trees 的 blob sha 按 git blob 规则校验，不过即删档中止并给 remedy（稍后重试或设 GITHUB_TOKEN 提 API 额度）；47 处 --no-check-certificate 与 curl -k 已全清（两脚本 grep 零残留）。jq 固定版由 1.7.1 升 1.8.1（1.7.1 的资产无 digest、1.8.1 有）。校验链已实测：解析对活体 API JSON 命中、jq 1.8.1 真实下载校验通过且可运行、API 额度不足时失败关闭。注：远程 rule-set（.srs）由 sing-box 内核按配置直拉官方仓，不经脚本下载，不在本项范围 | 已处置 |
| D | 下载可经第三方 GitHub 反代（ghproxy.lvedong.eu.org、accel.cloudflare.nyc.mn）与作者优选 IP 服务 | 已落地：check_cdn 的自动探测 / 自动启用反代逻辑整段删除，默认直连官方源；仅当用户显式设置环境变量 GH_PROXY 时才经其指定地址下载，且校验仍走官方 API、完整性判定不受代理影响。grep 复查 ghproxy / GITHUB_PROXY 默认表零残留 | 已处置 |
| E | 菜单远程脚本直跑（tcpx / argox / sba / tcp.hy2.sh）与 WARP 注册远程 api.sh | 逐项评估：能本地化的本地化；不能的在菜单文案明示来源与风险，或整项移除 | 待评估 |
| G | Docker 版写死共享 WARP 私钥 | 移植 VPS 版的 WARP 预注册逻辑，为每个部署生成独立账户；或明确提示用户自备 | 待排期 |

说明：以上四项均为「知悉」级完整性风险，不涉及用户数据已被外发；涉及用户数据的外发点（A / B / F 与作者站凭据引导）已全部清零。

## 基线检查（2026-11-26）

- `bash -n sing-box.sh`、`bash -n docker_init.sh`：均通过。
- ShellCheck 0.11.0：docker_init.sh 无 error 级问题；sing-box.sh 在 7627 行附近报一组解析 error（SC1046/SC1073/SC1036 等），经人工核对为**误报**——菜单代码用 `&&` / `||` 条件式定义 `ACTION[n]()` 函数，ShellCheck 解析不了这种写法，bash -n 通过且为上游有意写法，不记真问题。
- warning 级以 SC2155（声明与赋值同行）58 处为主，属上游普遍写法风格，不记真问题。
- **未做安装实测**（见文首）。

## 私货剥离记录（本轮已执行）

- README 第 14 节「感谢赞助商」整节删除：SharonNetworks（sharon.io 广告整段）、vps.town（赞助段）。目录与后续章节序号已顺延。
- README 第 5 节 Argo Json 获取：删去作者自有站的推广式引导，改为中立的功能表述。
- 保留：第 13 节鸣谢（千歌 sing-box 模板、zmlu 隧道脚本）属致谢、不是推广；全部功能说明、教程、参数表保留。
- 脚本代码处置：统计遥测与 Reality 私钥外发兜底已按定夺删除（见疑点 A/B 的处置记录）；脚本内提示文案中的作者自有站链接（疑点 D 相关）留待命名拍板后的标识替换轮一并处理。
