# 网络安全与服务器安全测试完全手册
## —— 攻击者视角倒推法 · 服务器/网络/WiFi 全栈测试作业书

> **版本**：2026-09 完全版 v2
> **方法论主线**：**倒推法**——先还原黑客的完整攻击路径，再对每一条路径逐项设计测试用例
> **适用**：自建机房 / 公有云 / 混合云 / 生产业务系统 / 办公网 / 无线局域网
> **用途**：直接下发给测试员执行，每条用例可执行、可判定、可复测
> **覆盖**：经典方法（Legacy）+ 2025/2026 最新攻防 + 学术论文级攻击技术

---

## 卷首：给测试员的作业规范（执行前必读）

### 0.1 三条铁律

| # | 铁律 | 说明 |
|---|---|---|
| 1 | **没有书面授权不动手** | 授权书必须写明：目标清单（IP/域名/**物理地址**）、时间窗、允许与禁止动作、双方紧急联系人、中止机制。缺一项都不开工。 |
| 2 | **生产环境不做破坏性测试** | 提权 exploit、域控操作、数据导出、deauth、DoS、注入写操作 —— 一律先在**镜像生产配置的实验室环境**验证，再决定是否在生产做只读验证。 |
| 3 | **测完必须清理并复核** | 测试账号、webshell、改过的配置、植入的样本、抓取的凭证，逐项核销，出《清理确认单》。凭证类数据测试后即销毁。 |

### 0.2 高风险动作清单（需单独签字）

| 动作 | 风险 | 谁签字 |
|---|---|---|
| 漏洞利用（尤其 RCE） | 可能导致服务崩溃 | 系统负责人 + 安全负责人 |
| 域控/AD 操作 | 影响全公司认证 | CISO / IT 总监 |
| 无线 deauth / Evil Twin 引流真实用户 | 业务中断 | 网络负责人 |
| 社工钓鱼演练 | 员工情绪、HR 风险 | HR + 法务 + 管理层 |
| 物理渗透 | 法律风险、人身安全 | 高管 |
| 压力测试 / 故障注入 | 生产抖动 | 运维负责人 + 业务负责人 |
| 数据外带验证 | 数据泄露 | 数据负责人（且必须脱敏） |

### 0.3 测试交付物清单

1. 《测试方案与范围确认书》（双方签字）
2. 《测试用例执行表》（逐条：通过/不通过/不适用 + 证据截图）
3. 《安全测试报告》（管理版 + 技术版）
4. 《攻击链叙事报告》—— 最有价值的一份
5. 《修复建议与优先级清单》
6. 《复测报告》
7. 《清理确认单》

### 0.4 手册使用方式

- **测试员**：从**第六卷测试用例库**开始，按编号执行，逐条打勾留证。遇到不懂的原理回查第二~五卷。
- **安全负责人**：重点看**第一卷（倒推法）**和**第七卷（度量与报告）**，用来判断测试是否覆盖到位。
- **决策层**：只看每卷开头的"风险摘要"和《攻击链叙事报告》。

---

# 第一卷 · 倒推法：黑客怎么打，我们就怎么测

> **核心思想**：不要从"我们有什么资产"出发，要从"敌人要走哪条路"出发。
> 每一步都问三个问题：① 这一步攻击者会做什么？② 我们怎么验证这条路走得通？③ 走通了我们会不会知道？

## 1.1 真实攻击链十五阶段（2026 版）

> 基于 2025–2026 勒索软件与 APT 实战情报还原。每一阶段给出：攻击动作 → 攻击者工具 → **我们的测试项** → **检测指标** → **阻断控制**。

| # | 阶段 | 攻击者实际做什么 | 常用工具/手法 | **我们测什么（倒推）** | 应产生的告警 | 阻断控制 |
|---|---|---|---|---|---|---|
| 1 | 暴露面发现 | 扫描你的公网资产、子域、泄漏凭证、边缘设备 | Shodan/Censys、infostealer 日志市场、IAB 初始访问经纪人 | 公网资产测绘；子域爆破；GitHub/网盘凭证泄漏扫描；域名仿冒监控 | 新暴露资产出现 | 资产台账 + 收敛 + 仿冒域名监控 |
| 2 | 初始访问 | 打边缘设备漏洞 / 偷会话 / ClickFix | VPN、防火墙、MFT（MOVEit 类）、RDWeb、ClickFix（**占微软 2025 通报的 47%**）、AiTM 钓鱼 | 边缘设备补丁核查；互联网暴露面收敛；钓鱼演练；ClickFix 场景演练 | 边缘设备异常登录 | 抗钓鱼 MFA（FIDO2）、及时打边缘补丁 |
| 3 | 绕过 MFA | **AiTM 反向代理实时中继**，MFA 全程真实通过，只偷最后下发的会话 Cookie；或从终端偷已存在的 Cookie | Tycoon 2FA（占 AiTM 检测 **59%**）、Sneaky2FA、Evilginx、FlowerStorm | 用 AiTM 模拟验证：短信/TOTP/推送 MFA 是否可绕过；FIDO2 是否可阻断 | 不可能旅行、同账号异地并发、Token 使用上下文异常 | **FIDO2/通行密钥（域名绑定）**、条件访问、会话绑定设备 |
| 4 | 落地与 C2 | 投放 loader → 回连 C2；同时装 RMM 留第二条路 | Cobalt Strike、Sliver、Brute Ratel；AnyDesk/Atera/Splashtop/ScreenConnect | 出网管控测试（能否连任意外网）；RMM 白名单/阻断策略验证 | 未知进程外连、RMM 进程启动 | 出网白名单、应用控制、RMM 管控 |
| 5 | 规避防御 | **BYOVD 直接内置在勒索载荷里**杀 EDR；直接系统调用绕过；无文件执行 | 脆弱驱动（NSecKrnl 等）、Direct Syscall、LOLBAS | BYOVD 驱动加载测试；EDR 抗卸载/抗杀能力验证 | 驱动加载异常、EDR 服务停止 | 驱动阻止列表（HVCI/WDAC）、EDR 自保护、凭据窃取防护 |
| 6 | 发现 | 四件套 11 秒完成：`whoami` / `net localgroup administrators` / `nltest /domain_trusts` / `wmic shadowcopy list brief` | 原生命令、AdFind、SharpHound、BloodHound | 模拟该命令序列，看 EDR/SIEM 是否告警 | 父进程为 Outlook 的 cmd 批量侦察 | 命令行审计、行为检测（非仅签名） |
| 7 | 提权 | 内存凭据、Kerberoasting、**AD CS 滥用**、NTDS.dit、文件共享里的明文口令 | Mimikatz、Rubeus、Certipy、secretsdump | 本地提权路径排查；AD CS 模板审计；共享目录口令扫描 | LSASS 访问、异常 TGT 请求 | LSASS 保护、Credential Guard、AD CS 加固 |
| 8 | 横向移动 | PsExec/WMI/WinRM/计划任务/GPO —— 全是系统自带 | Impacket、CrackMapExec、PsExec | 从低权限主机到DC 的最短路径（BloodHound）；PsExec 能否打通 | ADMIN$ 写入 PSEXESVC、异常服务创建 | 分层管理模型、网络分段、主机防火墙 |
| 9 | AD 域攻陷 | ACL 滥用、DCSync、委派、Golden Ticket | BloodHound、Rubeus、Mimikatz | BloodHound 路径分析；敏感 ACL 审计；域控加固核查 | DCSync（目录复制权限滥用） | 敏感组隔离、ACL 治理、Tier 0 加固 |
| 10 | 数据窃取 | **先偷后加密**；Rclone 传 Mega/对象存储 | Rclone、FileZilla、rclone+Cloudflare Worker 隧道 | 出网流量审计（能否识别 Rclone 特征）；大流量外传检测；DLP 有效性 | 异常出站流量、新域名大数据量上传 | 出网白名单、DLP、对象存储外传管控 |
| 11 | 破坏恢复能力 | **先打备份和 NAS**，删 VSS 卷影副本 | `vssadmin delete shadows`、`wmic shadowcopy delete` | 备份隔离测试（备份是否可被生产凭据删掉）；恢复演练 | 卷影删除命令 | **不可变备份**、备份独立凭据域、离线/离线副本 |
| 12 | 加密/施压 | 混合加密（AES-256 逐文件 + RSA/ECC 包裹；2026 已出现用 **ML-KEM/Kyber1024** 包裹的样本） | LockBit 4.0、Qilin、Akira、INC、MedusaLocker、Interlock | 加密行为检测（能否在加密前几分钟拦住）；EDR 回滚能力 | 高熵文件写入风暴 | EDR 行为阻断、蜜罐文件、恢复预案 |
| 13 | 无加密勒索（新趋势） | 2026 起"去掉 ware"，纯数据泄露勒索；2025 赎金支付率降到 **28%** | ShinyHunters 类数据泄露站 | 数据分类分级；外传防护；泄露监控 | —— | 数据防泄露优先于备份 |
| 14 | 持久化 | 计划任务、服务、WMI 事件订阅、Golden Ticket、**UEFI 植入** | `Set-WmiInstance`、Golden Ticket、LoJax/BlackLotus 类 | 持久化点全面排查；**UEFI 完整性校验**；黄金票据检测 | WMI 事件订阅创建 | 固件完整性校验、定期轮换 krbtgt |
| 15 | 痕迹清理 | 清日志、删样本、伪装时间戳 | 清事件日志、Timestomp | 日志防篡改测试；日志完整性校验 | 日志服务停止/日志清空 | 日志外发、WORM 存储、SIEM 侧留存 |

> **给测试员的一句话**：第 3 步（MFA 绕过）和第 5 步（EDR 击杀）是 2026 年最值得单独立项测的两项——大多数公司的"我们上了 MFA 和 EDR"在这两步面前是失效的。

## 1.2 MITRE ATT&CK 十四战术倒推测试矩阵

> 每个战术都要问：**"这条技术在我们环境里能不能走通？走通了有没有告警？"**

| 战术 | 代表技术 | 倒推测试用例 | 检测验证要点 |
|---|---|---|---|
| **侦察 Reconnaissance** | 主动扫描、收集受害者身份信息、钓鱼服务搭建 | TC-RECON-001~005：公网测绘、子域枚举、员工画像、仿冒域名监控、凭证泄漏监控 | 是否有外部扫描告警、是否有仿冒域名发现机制 |
| **资源开发 Resource Development** | 获取基础设施、编译能力、购买访问 | TC-RES-001~003：域名仿冒注册监控、凭据市场监控、第三方供应商暴露面 | 品牌保护、威胁情报接入 |
| **初始访问 Initial Access** | 外部远程服务、边缘设备漏洞、钓鱼附件、供应链、ClickFix、有效账号 | TC-IA-001~012：暴露面收敛、VPN/防火墙/MFT 补丁、弱口令、钓鱼演练、ClickFix 演练、AiTM 验证 | 边缘设备登录告警、异常地理位置登录 |
| **执行 Execution** | 命令脚本解释器、PowerShell、WMI、计划任务、用户执行（ClickFix）、服务执行 | TC-EX-001~008：PowerShell 受限语言、脚本块日志、应用白名单、宏策略、LNK/ISO/HTA 处理 | 进程创建链、父进程异常 |
| **持久化 Persistence** | 注册表 Run、计划任务、服务、WMI 订阅、启动文件夹、**固件植入**、Golden Ticket、OAuth 授权滥用 | TC-PER-001~012：全持久化点排查、UEFI 完整性、krbtgt 轮换、OAuth 同意审计 | 持久化点变更告警、启动项基线 |
| **权限提升 Privilege Escalation** | 提权漏洞、SUID/sudo 滥用、令牌操纵、绕过 UAC、AD CS 滥用、容器逃逸 | TC-PRIV-001~010：本地提权面、sudo 审计、UAC 级别、AD CS 模板、容器逃逸 | 提权行为告警、sudo 审计 |
| **防御规避 Defense Evasion** | 禁用安全工具、**BYOVD**、进程注入、伪装、清除日志、混淆、直接系统调用、LOLBAS | TC-DEF-001~014：驱动阻止列表、EDR 抗杀、注入检测、日志防篡改、LOLBAS 检测覆盖 | EDR 自保护告警、驱动加载审计 |
| **凭据访问 Credential Access** | LSASS 内存、SAM/NTDS、浏览器凭据、Kerberoasting、AS-REP、窃取会话 Cookie、**暴力破解** | TC-CRED-001~012：LSASS 保护、Kerberoasting 检测、AD CS、浏览器凭据访问检测、会话令牌防盗 | LSASS 访问告警、异常 Kerberos 请求 |
| **发现 Discovery** | 系统/网络/账户/域发现、云资源发现、容器发现 | TC-DISC-001~008：命令序列检测（`whoami`/`net`/`nltest`）、网络扫描检测、云 API 枚举检测 | 批量侦察行为检测（比单条命令重要） |
| **横向移动 Lateral Movement** | PsExec/WMI/WinRM/SMB/RDP、Pass-the-Hash、SSH 密钥复用、内部钓鱼、云账号切换 | TC-LAT-001~010：分区互通性、PsExec 检测、PTH 检测、SSH 密钥治理、跨账号 assume role 检测 | ADMIN$ 写入、服务创建、异常 NTLM |
| **收集 Collection** | 本地/网络共享/邮件/云存储收集、屏幕/剪贴板、**浏览器会话** | TC-COL-001~006：敏感数据定位、共享权限、邮箱规则、浏览器数据库访问检测 | 敏感文件批量访问、非浏览器进程读 Cookie 库 |
| **命令与控制 C2** | 应用层协议、DNS 隧道、**云函数/CDN 做中继**、反向代理（Chisel+Cloudflare Worker） | TC-C2-001~008：出网白名单、DNS 隧道检测、TLS 指纹/JA3、域名生成算法检测 | 外连新域名、DNS 异常查询、长连接心跳 |
| **数据渗出 Exfiltration** | 外传至云存储/网盘、C2 通道、物理介质、定时外传 | TC-EX-001~007：出网管控、Rclone/FileZilla 特征检测、大流量检测、USB/DLP | 出站流量突增、未授权网盘域名 |
| **影响 Impact** | 数据加密、数据销毁、服务停止、资源劫持、数据泄露站 | TC-IMP-001~006：备份不可变性、恢复演练、加密行为检测、勒索止损 | 卷影删除、高熵写入、服务停止 |

**执行建议**：不要一次性全测，先做**"高风险路径优先"**——把 1.1 表里第 2、3、5、7、8、10、11 步对应的用例先跑一遍，这几个是真实入侵中命中率最高、破坏最大的环节。

## 1.3 学术论文级攻击技术（高端对手/Aligned 场景测这些）

> 普通渗透测试不覆盖，**但如果你的服务器承载高价值数据、或属于关键基础设施，这些必须有人懂、必须测**。

### 1.3.1 微架构与侧信道（CPU 层）

| 攻击 | 论文/出处 | 原理 | **测试/防御验证项** |
|---|---|---|---|
| **Spectre 系列** | Kocher et al., *Spectre Attacks: Exploiting Speculative Execution*, IEEE S&P 2019 | 利用推测执行，通过缓存侧信道泄露跨安全域数据 | 微码/补丁版本核查；云环境是否隔离不可信租户；浏览器站点隔离；`spectre-meltdown-checker` 扫描 |
| **Meltdown** | Lipp et al., USENIX Security 2018 | 乱序执行绕过用户/内核隔离 | KPTI 是否启用；内核版本核查 |
| **Flush+Reload / Prime+Probe** | 经典缓存攻击 | 共享缓存争用时序差异泄露密钥 | 云上是否允许跨租户同机；密码库是否用**恒定时间实现**；是否启用缓存隔离 |
| **ZombieLoad / RIDL / Fallout** | 2019 瞬态执行族 | 利用填充缓冲/存储缓冲泄露 | 微码更新；同时多线程（SMT）是否在高敏环境关闭 |
| **Hertzbleed** | 2022 | 频率调整导致功耗/时序侧信道，可远程提取密钥 | 高敏服务器是否禁用动态调频；密码实现是否有防护 |
| **BranchScope / BTB 攻击** | 分支预测结构 | 推断跨边界秘密数据 | 微码与内核缓解核查 |
| **时序攻击（Timing）** | Kocher 1996 | 执行时间差异恢复密钥 | 密码/令牌比较是否恒定时间；API 响应时延是否泄露存在性（用户枚举） |

**测试动作**：`spectre-meltdown-checker`、`iucode_tool` 微码版本核对、OpenSSL/LibreSSL 恒定时间核查、云主机是否与其他租户共享物理核（可用缓存时序探测验证）。

### 1.3.2 内存与故障注入（DRAM 层）

| 攻击 | 出处 | 要点 | **测试/防御验证项** |
|---|---|---|---|
| **Rowhammer 原论文** | Kim et al., ISCA 2014 | 反复访问 DRAM 行导致相邻行 bit flip | 是否使用 **ECC 内存**；是否启用 TRR（Target Row Refresh）；是否开启内存加固 |
| **TRRespass** | 2020 | **双面/多面 hammering 绕过 DDR4 的 TRR 保护** | DDR4 机型是否仍有缓解缺口 |
| **Half-Double** | Google, 2021 | 利用 DRAM 物理特性恶化 | 同上 |
| **ZenHammer** | ETH Zürich, 2024-03 | 首个针对 **AMD Zen** 的 rowhammer，并首次攻击 **DDR5** | AMD 平台服务器专项核查 |
| **RISC-H** | ETH Zürich, 2024-06 | 首个 RISC-V 平台 rowhammer | 异构平台（如国产化/RISC-V 服务器）专项 |
| **Phoenix** | ETH Zürich, **2025-09** | **绕过 DDR5 全部 TRR 缓解**，用更长更复杂的图案实测成功 | **2025–2026 新部署服务器必测**：确认厂商缓解与 ECC 告警监控 |
| **RAMBleed** | 2019 | 利用 rowhammer 读取（而非篡改）内存 | 同上 |
| **冷启动攻击（Cold Boot）** | Halderman et al., 2009 | 断电后 DRAM 残留数据恢复密钥 | 是否启用内存加密（AMD SEV / Intel TME/MKTME）；是否禁止休眠到磁盘；是否有开机口令/TPM PIN |
| **故障注入（Glitching）** | 电压/时钟毛刺、激光、电磁 | 使芯片产生错误行为跳过校验 | 物理安全：机箱入侵检测、调试口（JTAG/SWD/UART）封堵 |

**给测试员的判定标准**：服务器是否启用 ECC + TRR/内存加固 + 安全启动 + 内存加密（SEV/TME/MKTME），缺少则在高价值场景判定为**高风险**。

### 1.3.3 固件、DMA 与供应链（硬件层）

| 攻击面 | 代表技术 | **测试项** |
|---|---|---|
| **UEFI/BIOS 植入** | LoJax（首个野外 UEFI rootkit）、BlackLotus（绕过 Secure Boot） | SPI flash 完整性校验；Secure Boot 是否启用且为**标准模式**；是否有 UEFI 口令；`chipsec` 扫描；DBX 吊销列表是否更新 |
| **BMC/IPMI/iLO/iDRAC** | 带外管理被攻破 = 完全控制主机 | 管理口是否独立网络；默认口令；固件版本；是否可从业务网直达 |
| **DMA 攻击** | PCIe/Thunderbolt 设备直读内存（Inception、Thunderclap） | **IOMMU/VT-d/AMD-Vi 是否启用**；是否禁用 Thunderbolt 热插拔；是否禁止未授权 PCIe 设备 |
| **硬件木马/供应链** | 芯片级植入 | 采购渠道可追溯；关键设备开箱检测；固件签名验证 |
| **TPM / 可信根** | TPM 2.0、Measured Boot、远程证明 | TPM 是否启用；是否做远程证明；BitLocker/LUKS 是否绑定 TPM + PIN |
| **SED 自加密盘** | 硬件加密实现缺陷 | **结论：软件加密（BitLocker/LUKS）比多数 SED 硬件加密更可信**；测 SED 是否可绕过 |
| **NIST 标准** | SP 800-147（BIOS 保护）、SP 800-155（BIOS 完整性度量） | 对照核查 |

### 1.3.4 协议与网络层学术攻击

| 攻击 | 要点 | 测试项 |
|---|---|---|
| **BGP 劫持 / 前缀劫持** | 路由泄露导致流量被劫持 | 是否做 RPKI/ROA；路由监控 |
| **DNS 缓存投毒（SADDNS 等）** | 伪造 DNS 响应 | 是否启用 DNSSEC 校验；源端口随机化；是否限制递归 |
| **TCP 侧信道 / 盲注入** | 利用 IP ID、窗口大小推断 | 边界设备是否泄露内部状态 |
| **分片攻击 / IP 分片重叠** | 绕过 IDS/防火墙 | IDS 是否做分片重组；是否丢弃异常分片 |
| **SSL/TLS 攻击史** | BEAST、CRIME、BREACH、POODLE、FREAK、Logjam、DROWN、ROBOT、Raccoon | `testssl.sh` 全量跑，禁 CBC 套件、禁 TLS1.0/1.1、禁重协商、禁压缩 |
| **证书透明日志滥用** | 从 CT log 发现内部域名 | 内部域名是否上了公网证书（信息泄露） |
| **降级攻击（通用）** | 强制协商到弱协议 | 是否禁用 fallback（TLS_FALLBACK_SCSV）、是否禁用弱算法 |

### 1.3.5 无线学术攻击（详见第四卷）

**KRACK**（Key Reinstallation Attacks, 2017，Vanhoef）—— 重放握手消息 3 导致 nonce 重用；
**Dragonblood**（2019，Vanhoef & Ronen）—— WPA3 SAE 侧信道与降级；
**FragAttacks**（2021，Vanhoef）—— 分片与聚合混淆，影响所有 Wi-Fi 设备；
**A-MSDU 注入**、**分片缓存污染**、**Beacon 洪水**。

### 1.3.6 AI/ML 系统攻击（若系统含 AI 能力）

| 攻击 | 说明 | 测试项 |
|---|---|---|
| 提示注入（直接/间接） | 间接注入更危险：藏在网页/PDF/简历里的白字指令 | 输入/输出过滤、最小权限、人工审批闸门 |
| 系统提示泄露 | 系统提示含业务逻辑与密钥 | 输出过滤检测泄露 |
| 越狱与对齐绕过 | 多轮诱导、编码绕过 | 红队提示库测试 |
| 训练/数据投毒 | 污染 RAG 知识库或微调数据 | 数据源可信度、向量库写权限 |
| 成员推断 / 模型抽取 / 模型反演 | 推断训练数据成员、复制模型、重建样本 | 查询限流、输出扰动 |
| 对抗样本 / 逃逸 | 扰动输入使模型误判 | 鲁棒性测试 |
| 向量与嵌入弱点 | 跨租户泄漏、嵌入反演 | 多租户隔离、访问控制 |
| 过度代理 / 无界消耗 | Agent 权限过大、token 洪水导致成本爆炸 | 工具调用白名单、预算上限、速率限制 |

---

---

# 第二卷 · 服务器安全测试（全栈十层）

> 从**硅片到应用**逐层测。任何一层被突破，上层的所有防护都失去意义。

## 2.0 分层模型与风险速查

| 层 | 对象 | 被突破的后果 | 本卷对应节 |
|---|---|---|---|
| L0 物理 | 机房、机柜、设备、线缆、调试口 | 一切归零（物理接触 = 完全控制） | 2.1 |
| L1 硬件/固件 | CPU、DRAM、UEFI、BMC、TPM、磁盘 | 持久化到重装系统都杀不掉 | 2.2 |
| L2 虚拟化 | Hypervisor、VM 逃逸、多租户 | 一虚机突破 = 全宿主沦陷 | 2.3 |
| L3 内核/OS | Linux/Windows 内核、驱动、sysctl | 权限任意提升、杀软失效 | 2.4 |
| L4 身份 | AD/Entra、IAM、证书、密钥、会话 | 横向移动、域控沦陷 | 2.5 |
| L5 服务/中间件 | SSH/RDP/Web/DB/队列/缓存 | 未授权访问、RCE | 2.6 |
| L6 应用/API | 业务逻辑、鉴权、数据 | 越权、注入、数据泄露 | 2.7 |
| L7 数据 | 数据库、对象存储、备份 | 勒索、泄露、合规事故 | 2.8 |
| L8 云/容器 | IAM、安全组、K8s、镜像 | 一个错误配置 = 全量暴露 | 2.9 |
| L9 供应链/CI | 依赖、流水线、制品 | 一次投毒 = 全客户中招 | 2.10 |
| L10 检测 | 日志、SIEM、EDR、SOC | 被打了也不知道 | 2.11 |

---

## 2.1 L0 物理与环境安全测试

| 测试项 | 怎么测 | 判定标准 |
|---|---|---|
| 机房门禁与访客 | 尾随测试（授权下）、访客登记抽查 | 无授权人员无法进入 |
| 机柜锁 | 抽查机柜是否上锁、钥匙管理 | 全部上锁、有领用登记 |
| 机箱入侵检测 | BIOS 中是否启用，现场触发测试 | 启用且能告警 |
| **调试口暴露** | 检查主板 JTAG/SWD/UART 排针、串口是否引出 | 无外露调试口 |
| USB/可移动介质 | 服务器 USB 口是否封堵、是否禁用大容量存储 | 封堵或策略禁用 |
| 控制台线缆 | 是否遗留 console 线、KVM 可达性 | 收纳管控 |
| 网络设备物理可达 | 交换机/防火墙 console 口是否被非授权人员触及 | 网络间上锁 |
| 供电与制冷 | UPS、双路供电、温度监控 | 冗余 + 告警 |
| 环境监控 | 温湿度、水浸、烟感 | 有监控并联动告警 |
| 存储介质销毁 | 报废硬盘处置记录（消磁/物理粉碎） | 有记录可查 |
| 打印/白板/便签 | 敏感信息遗留（口令、拓扑图） | 清桌政策 |

---

## 2.2 L1 硬件、固件与可信根测试

### 2.2.1 UEFI/BIOS

```bash
# Linux 下核查
[ -d /sys/firmware/efi ] && echo "UEFI 模式" || echo "Legacy BIOS（应升级）"
bootctl status 2>/dev/null | head -20
mokutil --sb-state          # Secure Boot 状态：期望 enabled
mokutil --list-enrolled     # 已注册的密钥
mokutil --dbx               # 吊销列表是否更新（BlackLotus 类绕过依赖旧 DBX）
dmidecode -t bios           # BIOS 版本与发布日期
fwupdmgr get-devices ; fwupdmgr security   # HSI（Host Security ID）评分
```

```powershell
# Windows 下核查
Confirm-SecureBootUEFI                       # 期望 True
Get-SecureBootPolicy                         
Get-CimInstance Win32_BIOS | Select SMBIOSBIOSVersion, ReleaseDate
# 检查是否存在可疑的 UEFI 启动项
bcdedit /enum firmware
Get-ChildItem HKLM:\SYSTEM\CurrentControlSet\Control\SecureBoot\State
```

| 测试项 | 判定 |
|---|---|
| Secure Boot 是否启用且为**标准模式**（非自定义/关闭） | 必须 enabled |
| DBX 吊销列表是否最新 | 否则 BlackLotus 类可绕过 |
| 是否有 UEFI/BIOS 管理口令 | 必须有 |
| 固件版本是否为最新、有无已知 CVE | 对照公告 |
| **SPI flash 完整性校验** | 用 `chipsec` 或厂商工具校验固件哈希 |
| 启动顺序是否禁止从外部介质启动 | 必须禁止或加口令 |
| Intel Boot Guard / AMD 硬件验证启动 | 高价值场景核查 |
| 是否启用机箱入侵检测 | 启用 |

### 2.2.2 BMC / 带外管理（**最容易被忽略的高危面**）

| 测试项 | 怎么测 | 判定 |
|---|---|---|
| BMC 网络位置 | 是否与业务网同网段 | **必须独立管理网/VPN** |
| 默认口令 | 授权下核查/有限尝试 | 必须改；known CVE 对照 |
| 固件版本 | 版本与公告比对 | 无未修高危 |
| 可达性 | 从业务 VLAN/办公网扫描 623/443/22 | 不可直达 |
| 功能最小化 | 是否关闭 IPMI over LAN、串口 over LAN、KVM 自动重定向 | 关闭非必要 |
| 虚拟介质 | 是否可被挂载（等于物理接触） | 禁用或强管控 |
| 日志与告警 | BMC 登录日志是否外发 | 必须外发 |
| 账号治理 | 是否有共享账号、离职账号 | 逐人账号 + MFA |

### 2.2.3 TPM / 可信根 / 磁盘加密

| 测试项 | 怎么测 |
|---|---|
| TPM 2.0 是否启用 | `tpm2_getcap properties-fixed` / Windows `tpm.msc` |
| 是否做 Measured Boot / 远程证明 | PCR 值是否被采集与校验 |
| 全盘加密 | Linux LUKS（`lsblk -f` 看 crypto_LUKS）；Windows BitLocker（`manage-bde -status`） |
| 密钥是否绑定 TPM + **启动 PIN** | 只有 TPM 无 PIN → 可被物理提取 |
| 休眠文件是否加密/禁用休眠 | 防止冷启动恢复密钥 |
| SED 硬件加密 | 若依赖 SED，需验证其实现（多数不如软件加密可靠） |

### 2.2.4 DMA 与外设

```bash
# IOMMU 是否启用（防 DMA 攻击的关键）
dmesg | grep -i -E "DMAR|IOMMU|AMD-Vi"
# 期望看到 DMAR: Intel(R) Virtualization Technology for Directed I/O 或 AMD-Vi
cat /proc/cmdline | grep -E "intel_iommu=on|amd_iommu=on"
ls /sys/kernel/iommu_groups/     # 有内容说明 IOMMU 生效
```

| 测试项 | 判定 |
|---|---|
| IOMMU/VT-d/AMD-Vi 启用 | 必须启用 |
| Thunderbolt / 热插拔 PCIe 策略 | 高敏服务器禁用或需授权 |
| 内核 DMA 保护（Windows） | 启用 |
| 是否禁止未签名驱动加载 | WDAC/HVCI、Linux 模块签名 |

---

## 2.3 L2 虚拟化与多租户隔离测试

| 测试项 | 怎么测 | 风险 |
|---|---|---|
| Hypervisor 版本与补丁 | ESXi/KVM/Hyper-V/Xen 版本对照公告 | 逃逸漏洞 |
| **管理口暴露** | vCenter/ESXi/Proxmox 管理界面是否可达业务网/公网 | 管理面直达 = 全量控制 |
| 虚拟机逃逸 | 授权下用已知 CVE PoC 在**测试环境**验证 | 一虚机 → 全宿主 |
| 资源隔离 | CPU/内存/磁盘 IO 是否有隔离与限额 | 邻居噪声与侧信道 |
| **跨租户侧信道** | 是否允许不可信租户与高敏负载同机（缓存时序可探测） | Spectre 类 |
| 快照与模板 | 快照是否含敏感数据、模板是否加固 | 数据泄露 |
| 虚拟网络 | vSwitch/VLAN 隔离、混杂模式、MAC 欺骗/伪传输策略 | 虚拟网内嗅探 |
| 虚拟磁盘加密 | VM 加密、vSAN 加密 | 静态数据 |
| 剪贴板/拖拽/共享目录 | 是否禁用（防 VM ↔ Host 通道） | 逃逸辅助 |
| 控制台访问 | 是否可绕过操作系统直接改配置 | 权限绕过 |
| 容器逃逸（容器场景） | privileged、hostPath、hostNetwork、docker.sock 挂载、`/proc/sys` 可写 | 容器 → 宿主 |

---

## 2.4 L3 内核与操作系统测试

### 2.4.1 Linux 专项

```bash
# ① 内核与补丁
uname -a ; cat /etc/os-release
rpm -qa kernel 2>/dev/null ; dpkg -l | grep linux-image
needs-restarting -r                     # 是否需要重启（RHEL）

# ② 提权面排查
find / -perm -4000 -type f 2>/dev/null          # SUID
find / -perm -2000 -type f 2>/dev/null          # SGID
find / -writable -type d 2>/dev/null | grep -v -E "^/(proc|sys|dev|tmp)"   # 全局可写目录
grep -r "NOPASSWD" /etc/sudoers /etc/sudoers.d/ 2>/dev/null
getcap -r / 2>/dev/null                          # 危险 capabilities

# ③ 内核参数（sysctl 加固）
sysctl -a 2>/dev/null | grep -E \
 "net.ipv4.ip_forward|net.ipv4.conf.all.(accept_redirects|send_redirects|accept_source_route|rp_filter|log_martians)| \
  net.ipv6.conf.all.accept_redirects|kernel.(kptr_restrict|dmesg_restrict|perf_event_paranoid|yama.ptrace_scope)| \
  fs.suid_dumpable|fs.protected_(hardlinks|symlinks)|kernel.randomize_va_space|net.core.bpf_jit_harden"

# ④ 模块与加载控制
lsmod
cat /proc/sys/kernel/modules_disabled            # 1 = 禁止再加载（高安全场景）
modprobe -n -v usb-storage                       # 禁用大容量存储模块（物理防护）
grep -r "blacklist" /etc/modprobe.d/

# ⑤ 强制访问控制
getenforce ; sestatus        # SELinux 期望 Enforcing
aa-status                   # AppArmor

# ⑥ 审计与完整性
systemctl is-active auditd ; auditctl -l
aide --check 2>/dev/null || echo "未部署 FIM"
rpm -Va 2>/dev/null | head -30              # 包完整性校验（RPM）

# ⑦ 计划任务 / 服务 / 持久化
systemctl list-unit-files --state=enabled
ls -la /etc/cron* /var/spool/cron/ /etc/systemd/system/
cat /etc/rc.local 2>/dev/null

# ⑧ 用户与认证
awk -F: '($3==0){print $1}' /etc/passwd                  # 有哪些 UID 0
awk -F: '($2==""){print $1}' /etc/shadow                 # 空口令
awk -F: '($2!="x"&&$2!="*"){print $1}' /etc/passwd       # 影子口令未启用
lastlog | grep -v "Never"                                # 活跃账号
cat /etc/pam.d/system-auth | grep -E "pam_(faillock|pwhistory|pwquality)"
grep -E "^(PermitRootLogin|PasswordAuthentication|PermitEmptyPasswords|X11Forwarding|MaxAuthTries|AllowUsers|AllowGroups)" /etc/ssh/sshd_config

# ⑨ 网络
ss -tulpn ; ss -tulpn | grep LISTEN
iptables -L -n -v ; nft list ruleset 2>/dev/null
cat /etc/hosts.allow /etc/hosts.deny 2>/dev/null

# ⑩ 容器/命名空间（若宿主）
docker ps --format '{{.Names}}\t{{.HostConfig.Privileged}}' 2>/dev/null
```

**Linux 判定要点**：SELinux/AppArmor 为 Enforcing；`kptr_restrict=1`、`dmesg_restrict=1`、`perf_event_paranoid≥2`、`yama.ptrace_scope≥1`；ASLR 开启；核心转储受限；审计规则覆盖关键事件；无异常 SUID；UID 0 只有 root；SSH 禁用 root+口令；fail2ban 部署。

### 2.4.2 Windows 专项

```powershell
# ① 补丁与版本
Get-HotFix | Sort-Object InstalledOn -Descending | Select -First 20
[System.Environment]::OSVersion.Version

# ② 凭据与内存保护
Get-CimInstance Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard   # Credential Guard / HVCI
reg query "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v RunAsPPL                 # LSASS 保护 = 2
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" /v LocalAccountTokenFilterPolicy

# ③ 网络与协议
Get-SmbServerConfiguration | Select RequireSecuritySigning, EnableSMB1Protocol, EncryptData
Get-WindowsOptionalFeature -Online -FeatureName SMB1Protocol                      # 应为 Disabled
reg query "HKLM\SYSTEM\CurrentControlSet\Services\NetBT\Parameters" /v NodeType   # 禁用 NetBIOS
Get-DnsClientNrptRule ; Get-ItemProperty HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient # LLMNR/多播禁用

# ④ PowerShell 与脚本安全
Get-MpPreference | Select -ExpandProperty AttackSurfaceReductionRules_Ids
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
Get-ExecutionPolicy -List
$env:__PSLockdownPolicy                    # 约束语言模式

# ⑤ 审计策略
auditpol /get /category:*

# ⑥ 服务/任务/持久化
Get-Service | Where {$_.Status -eq 'Running'} | Select Name, DisplayName
Get-ScheduledTask | Where {$_.State -ne 'Disabled'}
Get-CimInstance -Namespace root\Subscription -ClassName __EventFilter        # WMI 持久化（2026 高频）
Get-CimInstance -Namespace root\Subscription -ClassName __EventConsumer
Get-CimInstance -Namespace root\Subscription -ClassName __FilterToConsumerBinding

# ⑦ 本地管理员与 LAPS
Get-LocalGroupMember Administrators
Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwd | Select Name, ms-Mcs-AdmPwd  # LAPS

# ⑧ 驱动与代码完整性（防 BYOVD）
Get-WindowsDriver -Online -All | Select OriginalFileName, Version, SignerName   # 查签名/可疑驱动
Get-CimInstance Win32_SystemDriver | Select Name, PathName, State

# ⑨ 共享与文件权限
Get-SmbShare
icacls "C:\" /verify /t 2>$null | Select-String "错误" | Select -First 20
```

**Windows 判定要点（2026 优先级）**：
1. **HVCI + 驱动阻止列表**（BYOVD 是 2026 勒索内置能力，必须能拦脆弱驱动）；
2. Credential Guard + LSASS PPL（防凭据窃取）；
3. SMB 签名强制 + SMBv1 关闭；
4. ASR 规则 + PowerShell 脚本块/模块日志 + 约束语言模式；
5. WMI 事件订阅纳入监控（2026 主流持久化手法）；
6. LAPS 全覆盖；
7. RDP 不裸奔 + NLA + 限制来源。

### 2.4.3 通用主机加固核查（可脚本化）

| 类别 | 检查项 | 期望 |
|---|---|---|
| 补丁 | 高危补丁 SLA、EoL 系统 | 7–14 天内修复；无 EoL |
| 账户 | 默认账号、共享账号、停用未删、空口令 | 全部清零 |
| 口令 | 复杂度、历史、锁定、过期 | 满足策略 |
| MFA | 所有管理入口 | 全覆盖（且为抗钓鱼 MFA） |
| 远程 | SSH/RDP 加固、来源限制 | 密钥/NLA + 白名单 |
| 端口 | 非业务端口 | 关闭 |
| 服务 | 非必要服务 | 卸载（非仅停止） |
| 防火墙 | 主机防火墙 | 默认拒绝 |
| 日志 | 采集关键事件 + 外发 | 是 |
| 时间 | NTP 同步 | 是 |
| 加密 | 静态加密（LUKS/BitLocker） | 是 |
| 完整性 | FIM（AIDE/Tripwire/EDR） | 是 |
| 恶意代码 | EDR/AV 在线且更新 | 是 |
| 备份 | 备份 + **恢复演练记录** | 是 |
| 共享 | 匿名/Everyone 共享 | 无 |

---

## 2.5 L4 身份、访问与凭据测试（**2026 最关键的战场**）

### 2.5.1 认证体系

| 测试项 | 怎么测 | 判定 |
|---|---|---|
| **MFA 是否抗钓鱼** | 用 AiTM 代理（Evilginx/同类）在**受控环境**中继：短信 / TOTP / 推送是否可绕过；FIDO2/通行密钥是否阻断 | 仅 FIDO2 + 域名绑定能阻断；其他全部可被中继 |
| 条件访问 | 设备合规、地理位置、风险评分是否生效 | 生效 |
| 会话治理 | 会话有效期、刷新令牌策略、撤销是否即时 | 短时效 + 可即时吊销 |
| 会话令牌防护 | 令牌是否绑定设备/网络；异常使用是否告警 | 绑定 + 告警 |
| 密码策略 | 长度、黑名单（含已泄露密码库）、喷洒防护 | 长度优先，禁常见密码 |
| 账户锁定与限速 | 在线爆破是否触发锁定 | 有锁定与限速 |
| 服务账号 | 是否有过期不轮换的长期密钥 | 全部纳管轮换 |
| 遗留协议 | NTLMv1/LM、基本认证（IMAP/POP/SMTP Basic）、旧 TLS | 全部禁用 |
| 单点登录 | 身份源唯一、下游应用是否强制走 SSO | 强制 |
| OAuth 同意治理 | 用户可否随意授权第三方应用 | 需管理员审批 |
| 特权访问 | PAM/PIM 即时授权、审批、录屏 | 有 |
| 离职流程 | 账号停用时效、会话吊销、令牌吊销 | 即时 |

### 2.5.2 Active Directory 专项

| 测试项 | 手法 | 判定 |
|---|---|---|
| 到域控的最短路径 | BloodHound/SharpHound 出图，看从普通用户到 Domain Admins 几跳 | 跳数越少越危险 |
| Kerberoasting | 请求服务票据 → 离线破解；检查服务账号是否用弱口令/是否加 gMSA | 应无法破解 |
| AS-REP Roasting | 查找 `DONT_REQ_PREAUTH` 账号 | 应为 0 |
| **AD CS 滥用** | `certipy find -u ... -target ...` 找可滥用的证书模板（ESC1–ESC8） | 无危险模板 |
| DCSync 权限 | 检查非域控主体是否具 `DS-Replication-Get-Changes` | 仅受保护主体 |
| ACL 滥用 | 敏感 ACL（GenericAll/WriteDacl/WriteOwner/ForceChangePassword） | 治理到最小 |
| 委派 | 无约束委派 / 约束委派 / 基于资源的委派滥用 | 无无约束委派 |
| krbtgt 轮换 | 上次轮换时间 | 定期（事件后立即） |
| 敏感组嵌套 | Domain Admins/Enterprise Admins/Schema Admins 成员 | 最小化且无日常账号 |
| 分层管理 | Tier 0/1/2 是否隔离，管理员是否在普通工作站登录 | 严格隔离 |
| 组策略 | GPO 变更审计、SYSVOL 里的明文口令 | 无明文口令 |
| LDAP | 是否强制签名与通道绑定（防 NTLM 中继） | 强制 |
| 打印机 | Print Spooler 是否关闭（域控必须关） | 域控关闭 |
| 域控加固 | 域控是否只跑 DC 角色、是否可上网、是否有 EDR | 加固 |

### 2.5.3 密钥与机密管理

| 测试项 | 怎么测 | 判定 |
|---|---|---|
| 硬编码密钥 | 代码/配置/镜像/历史提交全量扫描（gitleaks/trufflehog） | 0 命中 |
| 密钥集中管理 | 是否用 KMS/Vault，还是散落在配置文件 | 集中管理 |
| 轮换 | 轮换周期与自动化 | 有周期且可自动 |
| 证书 | 有效期监控、私钥保护、吊销机制、弱算法 | 有监控、无 SHA1/1024 位 |
| SSH 密钥 | 是否无口令、是否复用、是否有授权清单 | 受控 |
| API 密钥 | 是否写死在前端/移动端 | 禁止 |
| 云角色 | 长期 AK/SK vs 临时凭证/工作负载身份联合 | 用临时凭证 |

---

## 2.6 L5 服务与中间件测试

| 服务 | 必测项 | 典型风险 |
|---|---|---|
| **SSH** | 禁 root 登录、禁口令、限用户/组、限来源、限速、算法白名单、版本 | 弱算法、可爆破、可转发（隧道） |
| **RDP** | NLA、来源限制、MFA、账户锁定、会话超时 | 裸奔公网 = 最常见初始访问 |
| **Web 服务器** | 版本隐藏、目录列表、危险模块、解析漏洞、默认页、请求限制、超时 | 解析漏洞 getshell、DoS |
| **Tomcat/JBoss/WebLogic** | 管理台口令、未授权、历史 RCE | 直接 RCE |
| **Redis/Memcached** | 未授权访问、写文件、主从复制 RCE、SSRF 打 Redis | 未授权 = 直接拿 shell |
| **MySQL/PostgreSQL** | 未授权、弱口令、公网监听、文件读写权限、备份泄露 | 数据泄露、提权 |
| **MongoDB/Elasticsearch** | 未授权访问（历史上大规模勒索） | 数据泄露 |
| **Docker API (2375)** | 未授权 = 直接控制宿主 | 宿主沦陷 |
| **K8s API / kubelet / etcd** | 匿名认证、未授权端口、etcd 无鉴权 | 集群沦陷 |
| **消息队列**（Kafka/RabbitMQ） | 未授权、明文传输 | 数据泄露 |
| **Jenkins / GitLab** | 未授权、脚本执行、凭据泄露 | 供应链投毒入口 |
| **DNS** | 域传送、递归开放、缓存投毒防护 | 信息泄露、劫持 |
| **邮件** | SPF/DKIM/DMARC、开放中继、网关漏洞 | 仿冒、入口 |
| **SNMP** | v2c community、v3 加密 | 信息泄露 |
| **NTP** | monlist 放大、未授权 | DDoS 放大 |
| **文件共享** | SMB 匿名、NFS 导出、FTP 匿名 | 数据泄露 |
| **管理后台** | 弱口令、未授权访问、是否在公网 | 直接控制 |

**测试手法统一模板**：
```bash
# 端口与服务指纹
nmap -sS -sV -sC -p- --open <target> -oA scan_full
# 常见未授权服务快速探测（仅授权环境）
nmap --script "redis-info,mongodb-info,docker-version,elasticsearch,smb-enum-shares,ftp-anon" <target>
```

---

## 2.7 L6 应用与 API 测试（OWASP Top 10:2025 全项）

> 详细用例见**第六卷**。此处只列覆盖面与"必须人工测"的部分。

| 类别 | 覆盖点 | 是否扫描器可测 |
|---|---|---|
| A01 越权（含 SSRF） | IDOR、功能级越权、水平/垂直越权、SSRF 打元数据与内网 | ❌ **必须人工**（扫描器不知道哪个订单属于谁） |
| A02 安全配置错误 | 默认口令、目录列表、详细错误、备份/源码泄露（.git/.svn/.bak/.env）、CORS、开放桶、安全头缺失 | 部分 |
| A03 软件供应链 | 依赖 CVE、依赖混淆、CI/CD 权限、制品签名、SBOM | 部分 |
| A04 加密失败 | 明文传输、弱算法、证书校验、硬编码密钥、可预测随机数 | 部分 |
| A05 注入 | SQL/命令/SSTI/LDAP/XPath/NoSQL/日志/表达式/SpEL | ✅ |
| A06 不安全设计 | 业务流程绕过、金额/库存/优惠券、验证码逻辑 | ❌ **纯人工** |
| A07 认证失败 | 爆破无锁定、验证码复用/绕过、重置逻辑、MFA 绕过、会话固定 | 部分 |
| A08 完整性失败 | 反序列化、未签名更新、自动更新走 HTTP | 部分 |
| A09 日志与**告警**失效 | 关键操作不记日志、有日志无告警、日志注入 | ❌ 人工 |
| A10 异常处理不当 | **fail-open**（鉴权异常却放行）、栈泄露、资源未释放、状态不一致 | ❌ 人工 + 模糊测试 |
| XSS / CSRF / 点击劫持 | 三类 + CSP/SameSite 有效性 | ✅ |
| 文件上传 | 类型绕过、内容检测绕过、路径穿越、解析 getshell | 部分 |
| XXE / 反序列化 | 外部实体、JNDI/LDAP | 部分 |
| 业务逻辑与竞争 | 并发下单/提现/库存、流程跳过 | ❌ 人工（Turbo Intruder） |
| API 专项 | BOLA/BOPLA/BFLA、资源消耗、GraphQL 深查询、影子/僵尸 API、Introspection 泄露 | 部分 |
| 客户端 | 前端硬编码、JS 泄露接口、本地存储、DOM XSS | 部分 |

---

## 2.8 L7 数据、存储与备份测试

| 测试项 | 怎么测 | 判定 |
|---|---|---|
| 数据分类分级 | 是否有清单、是否标注 | 有 |
| 敏感数据发现 | 全库/全盘扫描找明文身份证/手机/卡号/密钥 | 发现即整改 |
| 权限最小化 | 应用账号是否 DBA、是否可读写文件 | 最小权限 |
| 注入与越权读 | SQL 注入、越权查询 | 不可 |
| 静态加密 | TDE/列加密/磁盘加密 | 是 |
| 传输加密 | 内网是否也加密（不要假设内网可信） | 是 |
| 对象存储 | 桶策略、预签名 URL、列举权限、跨账号共享 | 无公开 |
| 快照 | 快照共享、快照含敏感数据 | 受控 |
| **备份不可变性** | 用生产凭据能否删掉备份（真实勒索第一步） | **必须不可变/隔离凭据** |
| 备份隔离 | 备份网络/账号是否与生产域独立 | 独立 |
| 离线副本 | 是否有离线/异地/不可变副本（3-2-1-1-0） | 有 |
| **恢复演练** | 真实恢复一次并计时 | **有记录，RTO 达标** |
| 数据保留与销毁 | 留存周期、安全销毁 | 有 |
| DLP | 外发通道管控、敏感内容识别 | 有效 |

> **3-2-1-1-0 原则**：3 份副本、2 种介质、1 份异地、1 份**不可变/离线**、**0 错误**（演练验证可恢复）。

---

## 2.9 L8 云与容器测试

### 2.9.1 云（AWS/Azure/GCP/阿里云/腾讯云通用）

| 层面 | 测试项 | 判定 |
|---|---|---|
| 身份 | 根账号 MFA、无长期 AK/SK、`*:*` 策略、`iam:PassRole` 提权路径、跨账号信任 | 无高危 |
| 网络 | 安全组 0.0.0.0/0、公网 IP 直挂、无出网管控、VPC 对等连接 | 最小暴露 |
| 计算 | **IMDSv1 是否禁用**（SSRF → 偷临时凭证）、实例角色权限 | 强制 IMDSv2 |
| 存储 | 公开桶、快照公开、跨账号共享、加密 | 无公开 |
| 密钥 | KMS 使用、轮换、密钥策略 | 受控 |
| 日志 | 操作审计（CloudTrail/操作审计/Cloud Audit Logs）开启 + 跨账号/不可变存储 | 开启 |
| 监控 | GuardDuty/Defender 类威胁检测是否开启 | 开启 |
| 治理 | 多账号/组织策略（SCP/Azure Policy/组织策略）是否阻止危险配置 | 有护栏 |
| 无服务器 | Lambda/函数角色、环境变量密钥、公网触发 | 最小权限 |
| 托管服务 | 数据库/缓存/搜索引擎是否公网可达 | 仅内网 |

### 2.9.2 容器与 Kubernetes

| 测试项 | 怎么测 | 判定 |
|---|---|---|
| API Server | 匿名认证、`--insecure-port`、是否公网 | 关闭匿名 |
| RBAC | `cluster-admin` 绑定范围、通配符权限 | 最小权限 |
| kubelet | 10250/10255 未授权 | 关闭只读端口 + 鉴权 |
| etcd | 是否 TLS + 客户端证书、是否暴露 | 是 |
| Pod 安全 | privileged、hostPID/hostNetwork/hostIPC、hostPath、以 root 运行、允许特权提升 | 全部禁止 |
| 准入控制 | Pod Security Admission / OPA Gatekeeper / Kyverno 是否强制基线 | 强制 restricted |
| 网络 | NetworkPolicy 默认拒绝、东西向加密（mTLS/服务网格） | 有 |
| 密钥 | Secret 是否加密（encryption-at-rest）、是否用外部密钥管理 | 是 |
| 镜像 | 漏洞扫描、是否来自可信仓库、是否签名、是否以 digest 拉取 | 全通过 |
| 供应链 | 构建可复现、SBOM、SLSA 级别、镜像 tag 不可覆盖 | 有 |
| 运行时 | 运行时检测（Falco/Tetragon）、异常进程/网络行为 | 有 |
| 逃逸 | docker.sock 挂载、cgroup release_agent、内核提权 | 不可逃逸 |
| 日志 | 审计日志、工作负载日志是否外发 | 是 |

---

## 2.10 L9 供应链与 CI/CD 测试

| 测试项 | 怎么测 | 判定 |
|---|---|---|
| SBOM | 是否生成 CycloneDX/SPDX 并持续更新 | 有 |
| 依赖漏洞 | SCA 扫描，含**传递依赖** | 有流程 |
| 依赖混淆 | 私有包名是否在公共源被同名高版本抢占 | 已防护 |
| 流水线权限 | Runner 是否共享、fork PR 是否拿到密钥、变量是否可被 PR 读取 | 已隔离 |
| 制品完整性 | 签名（cosign/Sigstore）、可溯源、tag 不可变 | 有 |
| 仓库防护 | 分支保护、强制评审、签名提交、禁止 force push | 有 |
| 密钥扫描 | 提交前/历史提交扫描 | 有 |
| 部署授权 | 生产部署是否需要审批、是否可回滚 | 有 |
| 第三方组件 | 开源组件的维护活跃度、是否含恶意代码 | 有评估 |
| 供应商 | 第三方远程维护通道是否常开、是否有审计 | 受控 |

---

## 2.11 L10 日志、监控与检测能力测试（紫队）

> **"有没有洞"和"被打了知不知道"是两件事。** 这一节是 2026 年最该补的。

| 测试项 | 怎么测 | 合格标准 |
|---|---|---|
| 日志覆盖 | 关键事件清单（登录/登出/提权/策略变更/进程创建/网络连接/文件访问/云 API 调用）是否采集 | 全覆盖 |
| 日志外发 | 是否实时外发到集中平台 | 是，且本地不可篡改 |
| 防篡改 | 攻击者删日志是否被发现 | 能发现 |
| 留存 | 留存期是否满足合规与取证需要 | ≥ 6–12 个月（关键 1 年+） |
| 时间同步 | 全链路 NTP | 是 |
| 检测覆盖 | 用 Atomic Red Team / Caldera 逐条模拟 ATT&CK 关键 TTP | 关键 TTP 有检测 |
| **MTTD** | 从模拟动作发生到告警产生的时间 | 有 SLA（关键场景建议 ≤ 分钟级~小时级） |
| **MTTR** | 从告警到隔离/处置完成的时间 | 有 SLA |
| EDR 有效性 | 常见落地、注入、凭据访问、BYOVD 是否被拦或告警 | 主流手法被拦 |
| 告警质量 | 告警是否带主机+用户+进程链上下文 | 可直接研判 |
| 误报率 | 抽样统计 | 可接受范围内 |
| 剧本与演练 | 是否有应急响应预案 + 演练记录 | 有且演练过 |
| 溯源能力 | 能否回答"谁、什么时候、从哪、做了什么、影响了什么" | 能 |

---

## 2.12 服务器稳定性与韧性测试（**用户重点：服务器稳不稳**）

> 安全之外，"稳不稳"决定业务连续性。这一节建议每季度至少跑一次。

### 2.12.1 性能与压力测试

| 类型 | 目的 | 方法 | 关注指标 |
|---|---|---|---|
| 基准测试（Baseline） | 建立性能基线 | 固定并发下的压测 | TPS/QPS、P50/P95/P99 延迟、错误率 |
| 负载测试 | 验证日常+峰值容量 | 逐步加压到预期峰值 | 拐点在哪 |
| 压力测试 | 找到崩溃点 | 持续加压直到失败 | 崩溃点、失败模式是否优雅 |
| **耐久/浸泡测试** | 发现内存泄漏、句柄泄漏、连接池耗尽 | 中等负载持续 **24–72 小时** | 内存/句柄/线程数是否单调增长 |
| 尖峰测试 | 突发流量（秒杀/热点） | 瞬间拉高并发 | 是否雪崩、是否有排队/限流 |
| 容量规划 | 支撑业务增长预测 | 建模 + 实测 | 扩容阈值 |

**关键指标**：CPU、内存、磁盘 IOPS/吞吐/延迟、网络带宽/PPS、连接数（含 TIME_WAIT）、GC（JVM）、数据库连接池、线程池队列、缓存命中率、队列积压、错误率、P99 延迟。
**资源红线建议**：CPU 持续 > 70%、内存 > 80%、磁盘使用率 > 80%、磁盘 IO await 明显升高 → 触发扩容。

### 2.12.2 故障注入与混沌工程

| 注入场景 | 验证什么 |
|---|---|
| 杀进程 / 杀容器 | 自愈、自动重启、是否影响请求 |
| 重启节点 / 宿主机 | 漂移、选主、数据是否一致 |
| 网络延迟/丢包/分区 | 超时、重试、熔断是否生效 |
| 依赖服务不可用（DB/缓存/MQ/三方 API） | 降级策略、是否有兜底 |
| 磁盘满 / IO 高 | 是否优雅报错而非崩溃 |
| CPU 打满 / 内存打满 | 限流、OOM 行为 |
| 时钟偏移 | 证书校验、令牌校验、分布式一致性 |
| DNS 失效 | 是否有本地缓存/备用解析 |
| 证书过期 | 是否提前告警（**最常见的自愈失败**） |
| 单可用区故障 | 跨 AZ 容灾是否真的生效 |
| 消息重复/乱序 | 幂等性 |

**混沌工程原则**：先有稳态指标（SLO）→ 提出假设 → 注入最小爆炸半径的故障 → 验证假设 → 记录偏差 → 修复。**生产环境注入必须有熔断开关与即时回滚**。

### 2.12.3 高可用与灾备演练

| 演练项 | 判定 |
|---|---|
| 主备切换（数据库/中间件/负载均衡） | 切换成功，RTO 达标 |
| 跨机房/跨可用区切换 | 成功，数据一致 |
| 备份恢复 | **真实恢复并启动服务**，RTO/RPO 达标 |
| 回滚 | 发布失败可回滚，回滚后数据兼容 |
| 全站压测 | 生产全链路（建议低峰） |
| 降级演练 | 非核心功能可关闭且核心可用 |
| 应急预案演练 | 走完整流程（发现→定级→通报→隔离→取证→恢复→复盘） |

**RTO/RPO 目标**：先由业务定义，再反推技术方案；没有定义的 RTO/RPO 等于没有灾备。

### 2.12.4 抗 DoS / DDoS 测试（**受控环境**）

| 层次 | 测试内容 |
|---|---|
| L3/L4 | SYN Flood、UDP Flood、ACK Flood、放大攻击（DNS/NTP/memcached/CLDAP）防护有效性 |
| L7 | HTTP Flood、慢速攻击（Slowloris/Slow POST）、CC 攻击、API 高频 |
| 应用层 | 大文件上传、复杂查询、深分页、正则 ReDoS、解压炸弹、GraphQL 深查询 |
| 架构 | CDN/WAF/高防是否生效、源站是否隐藏、是否有容量冗余、是否有上游清洗 |
| 依赖 | 第三方限流、短信/邮件轰炸防护 |
| 演练 | 与运营商/云厂商的应急联动流程是否跑通过 |

### 2.12.5 可观测性与 SLO

| 项 | 内容 |
|---|---|
| 指标（Metrics） | RED（Rate/Errors/Duration）+ USE（Utilization/Saturation/Errors） |
| 日志（Logs） | 结构化、有 traceId、可检索 |
| 链路追踪（Tracing） | 全链路打通，可定位慢点 |
| 告警 | 基于 **SLO 燃烧率**而非静态阈值；告警有 runbook |
| 仪表盘 | 关键业务 + 关键依赖一屏可见 |
| 值班 | On-call 轮值、告警分级、升级路径 |

---

---

# 第三卷 · 网络与通信安全测试（协议栈逐层）

## 3.1 L2 数据链路层

| 测试项 | 怎么测 | 风险/判定 |
|---|---|---|
| **ARP 欺骗** | 受控环境发伪造 ARP 响应，看能否中间人 | 无 DAI（动态 ARP 检测）= 可 MITM |
| **MAC 泛洪 / CAM 表溢出** | 端口安全是否限制 MAC 数量 | 未限制 = 交换机退化成集线器 |
| **VLAN 跳跃** | 尝试 DTP 协商、双重 tagging | 未关 DTP、Native VLAN 未改 = 可跨 VLAN |
| STP 攻击 | 注入 BPDU 抢根桥 | 未开 BPDU Guard / Root Guard |
| 私有 VLAN / 端口隔离 | 同网段主机能否互访 | 应按策略隔离 |
| DHCP 欺骗 | 起假 DHCP 服务器 | 未开 DHCP Snooping = 可下发假网关/DNS |
| 802.1X（有线） | 未授权设备接入能否上网 | 未启用 = 插线即入网 |
| MAB / NAC | 设备合规性检查 | 未部署 = 无准入 |
| 镜像口/分光口 | 是否有人接了未授权抓包设备 | 排查物理与配置 |
| 光纤/铜缆 | 是否可被搭线窃听（老建筑常见） | 物理排查 |

**交换机加固核查**：关闭 DTP、改 Native VLAN、开端口安全/BPDU Guard/Root Guard/DHCP Snooping/DAI、未用端口 shutdown 并放入隔离 VLAN、管理 VLAN 独立、禁用 Telnet/HTTP。

## 3.2 L3 网络层与路由

| 测试项 | 怎么测 | 判定 |
|---|---|---|
| IP 地址与网段规划 | 是否有台账、是否有私接网段 | 有台账 |
| 路由协议安全 | OSPF/IS-IS/BGP/RIP 是否启用认证 | 必须认证（MD5/Keychain/TCP-AO） |
| BGP 前缀劫持防护 | 是否部署 RPKI/ROA 校验、是否有路由监控告警 | 有 |
| 源地址验证 | 是否做 uRPF（防 IP 欺骗） | 是 |
| ICMP | 是否限制类型与速率（防 SMURF/隧道） | 限速 |
| IP 分片 | 边界是否丢弃异常分片/是否重组后检测 | 有策略 |
| TTL 过期/ traceroute | 是否泄露内网拓扑 | 边界过滤 |
| NAT/ACL 有效性 | 从外/内做规则复核 | 无 any-any |
| 分段验证 | 办公/生产/访客/IoT/管理/备份 各段互访矩阵 | 按矩阵严格控制 |
| **出网管控（Egress）** | 能否连任意外网、能否用 DNS/ICMP/非常规端口外传 | **必须有白名单**（这是拦住勒索外传的关键） |
| IPv6 | 是否启用但未管理（IPv6 双栈绕过 IPv4 防火墙是经典盲区） | 要么管理要么关闭 |
| 组播/广播 | 边界是否阻断 | 阻断 |
| 隧穿 | GRE/IPIP/6in4 是否可被用于绕边界 | 阻断未授权隧道 |

## 3.3 L4 传输层

| 测试项 | 内容 |
|---|---|
| TCP 参数 | SYN Cookie、连接数限制、半开连接限制、TIME_WAIT 复用 |
| UDP | 放大攻击面（DNS/NTP/SNMP/memcached/CLDAP）是否暴露公网 |
| 端口扫描防护 | 是否有速率限制/黑名单（减缓侦察） |
| 会话超时 | 防火墙/负载均衡/应用会话超时是否设置 |
| 长连接 | 是否有心跳异常检测（C2 常用长连接） |
| 负载均衡 | 健康检查、会话保持、源 IP 透传（是否被伪造） |

## 3.4 DNS 安全

| 测试项 | 怎么测 | 判定 |
|---|---|---|
| 域传送 | `dig axfr @ns example.com` | 仅允许从服务器 |
| 递归开放 | 从外部测试递归查询 | 仅内网递归，对外关闭 |
| DNSSEC | 是否签名 + 是否校验 | 关键域名启用 |
| 缓存投毒防护 | 源端口随机化、0x20 编码、查询 ID 随机 | 有 |
| DNS 隧道检测 | 用 iodine/dnscat2 类工具在授权环境测试是否可外传 | **必须能检测/阻断** |
| DDoS 防护 | 递归服务器是否被用于放大 | 关闭对外递归 |
| 内部信息泄露 | 内部域名解析记录是否可被外部查询 | 内外分离（split-horizon） |
| 证书透明日志 | 内部域名是否签发过公网证书（泄露主机名） | 监控 CT log |
| DoH/DoT | 是否绕过本地 DNS 策略（也是外传通道） | 有策略 |
| 域名/注册商 | 注册商账号 MFA、域名锁定、自动续费 | 有 |

## 3.5 TLS / PKI 与加密

```bash
# 全量 TLS 配置体检（最实用的一条命令）
testssl.sh --full https://example.com
# 或
sslyze --regular example.com:443
# 协议与套件快速检查
nmap --script ssl-enum-ciphers -p 443 <host>
openssl s_client -connect host:443 -tls1_2
```

| 测试项 | 判定标准 |
|---|---|
| 协议版本 | 仅 TLS 1.2+（**TLS 1.0/1.1/SSLv3 全部禁用**） |
| 加密套件 | 仅 AEAD（AES-GCM/ChaCha20-Poly1305）；禁 CBC、RC4、3DES、NULL、EXPORT |
| 密钥交换 | ECDHE/DHE（前向保密）；禁静态 RSA 密钥交换；参数 ≥ 2048 位 DH / P-256 |
| 压缩 | CRIME → **必须禁用 TLS 压缩** |
| 重协商 | 安全重协商 / 禁客户端发起重协商 |
| 降级防护 | TLS_FALLBACK_SCSV / 禁 fallback |
| 证书 | 有效、未过期（**监控 <30 天告警**）、域名匹配、完整链、非 SHA-1、RSA ≥2048 / ECDSA ≥256 |
| 吊销 | OCSP stapling、CRL 可达 |
| HSTS | 启用且 max-age 足够（含 preload 更佳） |
| 私钥 | 保护（HSM/KMS）、无共享、无泄露 |
| 内部流量 | 是否也加密（零信任：内网不可信） |
| 算法敏捷性 | 是否可快速替换算法（为后量子迁移做准备） |
| 后量子 | 是否已在评估 ML-KEM 混合（TLS 1.3 X25519Kyber768 类） |

## 3.6 VPN / 远程接入 / SD-WAN

| 测试项 | 判定 |
|---|---|
| 协议 | 禁用 PPTP、IKEv1、L2TP 无 IPsec；用 IKEv2/IPsec 或现代 TLS VPN |
| 加密 | 强套件、证书或 EAP-TLS 认证 |
| MFA | 强制且抗钓鱼 |
| 补丁 | **VPN/防火墙是 2025–2026 头号初始访问来源**，补丁 SLA 应最短 |
| 暴露面 | 管理口与 VPN 是否分离；是否可被扫描发现 |
| 权限 | 是否全网直通 vs 最小授权（**全隧直通 = 一台中毒机横扫内网**） |
| 客户端合规 | 是否检查设备状态（EDR/补丁/合规）再放行 |
| 会话 | 超时、并发限制、异常地理位置告警 |
| 日志 | VPN 登录日志是否进 SIEM |
| 替代方案 | 是否能用 ZTNA 替代传统 VPN（按应用授权而非按网络） |

## 3.7 邮件安全（最常见攻击入口）

| 测试项 | 怎么测 | 判定 |
|---|---|---|
| SPF | `dig TXT example.com` | 存在且 `-all`（硬失败） |
| DKIM | 选择器查询，验证签名 | 启用且密钥 ≥2048 |
| DMARC | `dig TXT _dmarc.example.com` | `p=quarantine` 或 `p=reject` |
| 入站防护 | 是否有高级反钓鱼（含 AiTM 链接防护）、附件沙箱 | 有 |
| 出站管控 | 是否防数据外发、防内部账号被滥发 | 有 |
| 邮件规则滥用 | 是否监控异常收件箱规则/转发（入侵后常用） | 有监控 |
| 附件类型 | 是否拦截高风险类型（ISO/IMG/LNK/HTA/JS/VBS、SVG 内含脚本） | 拦截或沙箱 |
| URL 重写与时效 | 链接点击时是否实时检测（防 AiTM 站点已下线绕过） | 有 |
| 内部仿冒 | 是否有显示名仿冒（CEO  fraud）防护 | 有 |
| 遗留协议 | 基本认证（IMAP/POP/SMTP AUTH）是否禁用 | 禁用 |
| 归档与取证 | 邮件归档与 eDiscovery 能力 | 有 |

## 3.8 边界防护设备

| 设备 | 测试项 |
|---|---|
| 防火墙 | 规则有效性（无 any-any、无冗余、有注释、有定期评审）、管理口隔离、HA、日志 |
| **IPS/IDS** | 是否真正开启阻断（很多只开告警）、规则库时效、是否做分片重组、是否覆盖内部流量 |
| WAF | 规则模式（阻断/仅告警）、是否绕过（编码/分块/参数污染）、误报率、API 防护 |
| 抗 DDoS | 阈值、清洗能力、源站隐藏 |
| 上网行为/出网网关 | 是否做外传管控（Rclone/网盘/云存储域名） |
| 堡垒机 | 是否所有运维必经、是否录屏、是否可绕行 |
| 蜜罐 | 是否有诱饵账号/文件/主机（提高发现概率） |

## 3.9 网络管理与运维通道

| 测试项 | 判定 |
|---|---|
| 管理协议 | 仅 SSHv2/HTTPS/SNMPv3；禁 Telnet/HTTP/SNMPv1/v2c |
| 管理网络 | 独立管理 VLAN/VRF，仅堡垒机可达 |
| 设备账号 | 集中认证（TACACS+/RADIUS）+ MFA + 逐人账号 |
| 配置备份 | 定期备份 + 变更审计 |
| 固件 | 版本与公告对照（**边缘设备补丁是 2026 头等大事**） |
| console 口 | 物理保护 |
| 日志 | 设备日志全部外发 |

---

# 第四卷 · 无线与射频安全测试（全频谱）

> 本卷覆盖：Wi-Fi 全代际（含 802.11 帧级攻击与学术漏洞）+ 企业 802.1X + Wi-Fi 7/6E + 流氓 AP + 客户端 + RF 物理层 + 蓝牙/Zigbee/RFID/Sub-GHz/GNSS/蜂窝。

## 4.1 无线测试范围与红线（重申）

| 项目 | 要求 |
|---|---|
| 授权 | 授权书必须写**具体物理地址**（楼栋/楼层/园区）；共享楼宇需物业同意 |
| 绝不触碰 | 邻居/路人/运营商网络 —— 只做被动观测记录 |
| 需单独签字 | deauth、DoS、Evil Twin 引流真实用户、EAP 凭证捕获 |
| 优先实验室 | 用**镜像生产配置的实验室 SSID**做破坏性验证 |
| 中止机制 | 谁可叫停、如何叫停、如何回滚 |

## 4.2 Wi-Fi 协议代际与攻击面总表

| 代际 | 协议/特性 | 关键攻击 | 是否仍要测 |
|---|---|---|---|
| 802.11b/g | **WEP** | RC4 弱 IV、ARP 重放/chopchop，数分钟破解 | ✅ 查僵尸 SSID（打印机/扫码枪/老摄像头） |
| 802.11i | **WPA-TKIP** | MIC 攻击（Beck-Tews）、密钥恢复 | ✅ 查遗留 |
| — | **WPA2-PSK** | 4 次握手离线字典、**PMKID 无客户端**、弱口令、PSK 复用 | ✅ **重点** |
| — | **WPA2-Enterprise** | Evil Twin 抓 MSCHAPv2、证书校验缺失、EAP 降级 | ✅ **重点** |
| — | **WPS** | PIN 在线爆破、Pixie-Dust 离线 | ✅ **重点**（分支/家用常忘关） |
| 802.11w | **PMF** | 未开 → 伪造 deauth 掉线 + 为 Evil Twin 铺路 | ✅ **必测** |
| — | **KRACK**（2017, Vanhoef） | 重放握手 msg3 → nonce 重用，可解密/注入 | ✅ 补丁核查 |
| — | **FragAttacks**（2021, Vanhoef） | 分片与聚合混淆，**影响几乎所有 Wi-Fi 设备** | ✅ 补丁核查 |
| — | **Dragonblood**（2019） | WPA3 SAE 侧信道 + 降级 | ✅ **必测** |
| — | **WPA3-Personal（SAE）** | 抗离线破解；剩余：过渡模式降级、侧信道、弱口令在线猜测 | ✅ **必测** |
| — | **WPA3-Enterprise 192-bit** | GCMP-256 + EAP-TLS；PKI 运维 | ✅ 高敏必测 |
| — | **OWE（增强开放）** | 只加密不认证 | ✅ 访客网 |
| Wi-Fi 6/6E | 6 GHz | 仅允许 WPA3/OWE，**无过渡模式**；PSC 扫描/RNR | ✅ 新部署 |
| Wi-Fi 7 | **MLO / 802.11be** | 多链路配置不一致、GCMP-256 强制、Beacon Protection、hostapd MLO CVE（如 CVE-2026-58374，2.12 修复） | ✅ **新部署必测** |
| 802.11az | 增强测距 | 加密 FTM，涉及定位与流氓 AP 检测 | 高安全场景 |

## 4.3 无线测试八步流程

| 步骤 | 内容 | 产出 |
|---|---|---|
| ① 范围与验收标准 | 楼栋/楼层/SSID/时间窗/禁止项；验收分"必过/应过/探索"三级 | 测试计划 |
| ② **RF 测绘**（先被动） | 走场测 SNR、重传率、同邻频干扰、盲区、外溢 | 热力图 + 信道图 |
| ③ 资产清点 | 全部 BSSID/ESSID/加密/信道/厂商/客户端 | 无线资产清单 |
| ④ 认证测试 | WEP→WPA2→WPA3→WPS→802.1X 逐项 | 认证层问题 |
| ⑤ 基础设施 | 流氓 AP、管理口、固件、控制器、RADIUS | 基础设施问题 |
| ⑥ 客户端测试 | probe 泄露、证书校验、随机 MAC、自动连接 | 客户端问题 |
| ⑦ 后连接 | 分段、隔离、横向、到管理面 | 分段验证 |
| ⑧ 监测与报告 | WIDS 时效、SIEM 接入、修复与复测 | 报告 |

**验收标准写法示例**：
> "所有 WPA3 SSID 的 PMF 必须为 required；在财务区放置伪造 Beacon 后，WIDS 必须在 2 分钟内告警并定位到楼层。"

## 4.4 认证层测试（逐协议）

### 4.4.1 WEP / WPA-TKIP 遗留

```bash
airmon-ng check kill ; airmon-ng start wlan0
airodump-ng wlan0mon                                   # 全局扫描
airodump-ng -c 6 --bssid <BSSID> -w cap wlan0mon      # 定向抓包
aireplay-ng --arpreplay -b <BSSID> -h <MY_MAC> wlan0mon   # 加速 IV
aircrack-ng cap-01.cap                                  # 破解
```
判定：出现 WEP/TKIP → **严重**；能解出密钥 → 立即下线该设备。

### 4.4.2 WPA2-PSK

```bash
airodump-ng -c 6 --bssid <BSSID> -w wpa2 wlan0mon
aireplay-ng -0 5 -a <BSSID> -c <CLIENT> wlan0mon      # 仅授权下
aircrack-ng -w wordlist.txt wpa2-01.cap
# 更隐蔽：PMKID（无需客户端）
hcxdumptool -i wlan0mon -o pmkid.pcapng --enable_status=1
hcxpcapngtool -o pmkid.hash pmkid.pcapng
hashcat -m 22000 wpa2.hc22000 wordlist.txt            # 推荐（握手+PMKID）
hashcat -m 16800 pmkid.hash wordlist.txt
```

| 检查项 | 报告写法 |
|---|---|
| 可捕获握手/PMKID | "可捕获认证材料" |
| 弱口令被离线破解 | "WPA2 口令可离线破解（已还原明文）" |
| PSK 跨 AP/跨站点复用 | "预共享密钥全网复用" |
| 未轮换 / 离职人员仍知 | "PSK 从未轮换" |
| 802.11w 未启用 | "管理帧保护缺失" |

### 4.4.3 WPS

```bash
wash -i wlan0mon -C                       # 发现
reaver -i wlan0mon -b <BSSID> -vv         # 在线 PIN 爆破
bully wlan0mon -b <BSSID> -p <pin>
reaver -i wlan0mon -b <BSSID> -K 1        # Pixie-Dust（离线，脆弱芯片秒出）
```
判定：WPS 开启 → 不合格；PIN 爆破或 Pixie-Dust 成功 → 高；失败无锁定 → 中。

### 4.4.4 WPA3（**重点，很多团队只测到 WPA2 就停**）

| 检查项 | 怎么测 | 报告写法 |
|---|---|---|
| **过渡模式降级** | 起伪造 WPA2 AP 用同名 SSID，看 WPA3 客户端是否降级关联 | "WPA3 过渡模式可降级至 WPA2" |
| **Dragonblood** | 核对 AP/客户端补丁；受控环境做 SAE 侧信道/降级验证 | "存在 Dragonblood 风险（未打补丁）" |
| **PMF 是否 required** | 抓 Beacon/RSN IE 看 **MFPR 位**；固件升级的老 AP 常是 optional | "802.11w 未强制" |
| SAE 组与策略一致性 | 抽查生产 AP：SAE group、过渡模式开关、密码套件（配置漂移） | "AP 配置漂移" |
| 口令熵 | SAE 抗离线但仍有有限在线猜测面 | "WPA3-Personal 口令熵不足" |
| OWE | 访客网是否从"开放"升级到 OWE | "访客网未启用 OWE" |
| Beacon Protection | 6 GHz / Wi-Fi 7 场景核查 | "未启用 Beacon Protection" |
| 客户端兼容性 | 抽样终端（含 IoT/扫码枪/PDA）能否稳定关联 WPA3-only | "存在无法接入 WPA3 的遗留终端" |

### 4.4.5 WPA2/WPA3-Enterprise（802.1X / EAP / RADIUS）

```bash
eaphammer --cert-wizard
eaphammer -i wlan0 --essid CorpWiFi --channel 6 --auth wpa-eap --creds
# 或 hostapd-wpe 记录用户名 + MSCHAPv2 challenge/response
hostapd-wpe hostapd-wpe.conf
asleap -C <challenge> -R <response> -W wordlist.txt
# 凭证可否破解
hashcat -m 5500 mschapv2.txt wordlist.txt
```

| 检查项 | 怎么测 | 判定 |
|---|---|---|
| **客户端是否校验服务器证书** | 用自签证书起 Evil Twin，看客户端是否告警/拒绝 | 不校验 = **严重**（802.1X 形同虚设） |
| CA 与服务器名是否固定 | 检查 supplicant 是否 pin 了 CA 与 CN/SAN | 未 pin = 高 |
| EAP 方法 | 是否允许弱 EAP（LEAP/EAP-MD5）；PEAP-MSCHAPv2 vs **EAP-TLS** | 弱方法 = 高 |
| MSCHAPv2 可离线破解 | asleap / hashcat -m 5500 | 可破解 = 账号沦陷 |
| 凭证中继 | 抓到的凭证能否用于 VPN/邮件/OWA | 可中继 = 影响放大 |
| RADIUS 共享密钥 | 强度审计 | 弱密钥 |
| RADIUS 可达性 | 无线客户端 VLAN 能否直连 RADIUS/管理面 | 应隔离 |
| 证书生命周期 | 有效期、CRL/OCSP、私钥保护 | PKI 运维 |
| 账号治理 | 离职账号吊销、动态 VLAN 是否被绕过 | 越权 |
| 配置分发 | 是否由 MDM/GPO 强制下发（防用户点"仍要连接"） | 未管控 = 高 |
| **AD CS / 证书模板** | 与 2.5.2 联动审计 | 危险模板 |

**最强修复**：**EAP-TLS（客户端证书）** + MDM/GPO 强制证书校验与服务器名固定。

## 4.5 Wi-Fi 6E / Wi-Fi 7 专项（2025–2026 新部署必测）

| 检查项 | 怎么测 | 判定 |
|---|---|---|
| 6 GHz 仅 WPA3/OWE | 抓 Beacon/RSN；6 GHz **不允许过渡模式** | 出现 WPA2 = 配置错误 |
| **MLO 安全一致性** | 2.4/5/6 GHz 各链路上分别验证：加密、认证、VLAN、ACL、固件、监控是否**完全一致** | 任一链路弱于其他 = 高 |
| MLO 握手与密钥同步 | 模拟链路切换/中断，验证 SAE 与密钥跨链路同步 | 状态机不同步 |
| PMF 在 MLO 下严格性 | 模拟 deauth 下的鲁棒管理帧处理（认证实验室最常见失败点） | 不完整 PMF |
| GCMP-256 / SAE-EXT-KEY | Wi-Fi 7 要求：Personal 用 GCMP256+SAE-EXT-KEY；Enterprise 提供 GCMP256 | 仍用 AES-128 = 不满足 |
| Beacon Protection | 是否启用（部分版本自动开启，需核对实际行为） | 未启用 |
| 固件 CVE | 对照公告（hostapd MLO 类 DoS，2.12 已修） | 版本落后 |
| 遗留客户端兼容 | Wi-Fi 5/6 客户端是否被 MLO 调度饿死 | 可用性 |
| 频谱合规 | 6 GHz CAC/DFS、发射功率、区域配置 | 合规 |

## 4.6 基础设施攻击

| 攻击 | 怎么测 | 判定 |
|---|---|---|
| **Shadow AP（影子 AP）** | 现场频谱 + 协议扫描找资产清单外的发射源；定向天线物理定位 | 私接路由器/随身热点/vendor 设备 |
| **Rogue AP（伪装 AP）** | 起同名 SSID，看 WIDS 是否告警与自动遏制 | 无告警 = 监测失效 |
| **Evil Twin 引流** | 同 SSID + 更强信号（+ deauth，授权下），看客户端是否自动迁移 | 自动连 = 客户端加固失效 |
| **KARMA / Known-Beacons** | 对客户端历史 probe 的 SSID 批量响应 | 客户端可被诱导 |
| 开放网络自动连接 | 客户端是否保存/自动连开放网络 | 配置问题 |
| WIDS 时效 | 记录"放伪造 Beacon"到"告警"的时间 | 超出 SLA |

## 4.7 客户端侧测试

| 检查项 | 怎么测 |
|---|---|
| Probe Request 泄露 | 抓空口 probe，看广播了哪些历史 SSID（公司名/家庭/酒店） |
| MAC 随机化 | 扫描阶段是否启用随机 MAC |
| 证书校验 | 见 4.4.5 |
| 已保存网络卫生 | 是否残留公司 SSID（配合 KARMA） |
| 客户端隔离（P2P） | 访客网两台客户端能否互访（**必须阻断**） |
| MDM/策略锁定 | supplicant 配置是否被锁定，用户能否手工关闭校验 |
| 终端防护 | 接入无线的设备是否有 EDR/合规检查（NAC） |

## 4.8 后连接：无线 → 内网

> **"访客网能到内网"是无线测试里最常见的高危发现**：大堂一把椅子 = 一个内网立足点。

| 测试 | 合格标准 |
|---|---|
| VLAN 分段 | 访客/员工/IoT/语音/管理 各自独立 VLAN/VRF |
| 访客 → 内网 | **全不通**（含内部 DNS、管理口、文件共享） |
| 客户端隔离 | 访客网内互不可达 |
| 到管理面 | 仅管理 VLAN 可达 AP/控制器/交换机/RADIUS |
| 出网路径 | 访客独立出口 + 内容过滤 |
| 横向移动 | 拿到无线客户端后无横向路径 |
| 与有线等价 | 无线权限 ≤ 同类有线角色 |
| IoT/OT 分段 | 独立 SSID + 独立 VLAN + ACL，禁止主动外连 |

## 4.9 基础设施与管理平面

| 检查项 | 判定 |
|---|---|
| AP/控制器管理口 | 不可从客户端 VLAN 直达 |
| 默认/共享口令 | 零容忍 |
| 管理协议 | 仅 HTTPS/SSH/SNMPv3 |
| 固件 | 有清单、有更新流程、无落后版本 |
| **配置漂移** | 抽查多台 AP 的安全参数是否与基线一致 |
| 云管理门户 | MFA、权限、审计日志 |
| 物理安全 | 防拆、防复位、POE 口管控、机柜上锁 |
| 配置备份与变更记录 | 有 |

## 4.10 RF 物理层与频谱

| 检查项 | 怎么测 |
|---|---|
| **信号外溢** | 在街道/停车场/相邻楼层测公司 SSID 可用强度（可关联即风险） |
| 覆盖与盲区 | 走场测 SNR、重传率、漫游阈值，比对设计勘测 |
| 干扰 | 频谱分析找非 Wi-Fi 干扰源（微波炉、蓝牙、无线摄像头、雷达、LED 电源） |
| 管理帧占比 | 异常高 = 攻击或最小速率配置错误 |
| 最小速率 | 过低拉低整体容量 |
| 抗 deauth | 未开 PMF 时掉线影响评估 |
| **定向定位** | 用定向天线/Yagi 定位异常发射源 |
| 频谱合规 | 功率、DFS/CAC、信道合法性 |

## 4.11 其他无线协议（范围涵盖时）

| 协议 | 测试内容 | 工具 |
|---|---|---|
| **BLE / 蓝牙** | 广播嗅探、GATT 未授权读写、配对降级、Just Works 无 MITM、KNOB、BlueBorne 类、信标伪造、BLE 中继（无钥匙进入） | Ubertooth、nRF52840、bettercap BLE、BLEah |
| **Zigbee / Thread** | 入网密钥捕获、重放、未加密命令、默认 link key、Zigbee 3.0 降级 | KillerBee、Z3sec、ApiMote |
| **Z-Wave** | 密钥交换与重放 | 专用嗅探工具 |
| **RFID / NFC** | MIFARE Classic 克隆、UID 改写、**中继攻击**、门禁凭证复制 | Proxmark3、Flipper Zero（授权） |
| **Sub-GHz / LoRa** | 重放、滚动码（rolling code）可预测性、门控/遥测 | HackRF、YardStick One + rfcat |
| **无钥匙进入/TPMS** | RollJam 类重放与干扰 | SDR（严格授权） |
| **GNSS/GPS** | 欺骗（spoofing）与干扰（jamming）—— 影响时间同步（NTP/PTP 依赖 GPS） | HackRF + gps-sdr-sim（**仅限屏蔽室**） |
| **蜂窝 / 伪基站** | 私接 4G/5G CPE 造成后门；伪基站短信 | 现场排查 + 出网流量审计 |
| **无线键盘鼠标** | injecting / 键鼠劫持（MouseJack 类） | 专用硬件 |
| **红外 / 超声 / 光** | 光侧信道、声侧信道（学术）、红外遥控重放 | 专用设备 |

**默认排除项**（要写进范围文档，避免被认为"测过"）：sub-GHz 工业遥测、BLE、Zigbee/Z-Wave、蜂窝/专网 5G、RFID、GNSS —— 除非单独采购。

## 4.12 无线持续监测

| 检查项 | 合格标准 |
|---|---|
| WIDS/WIPS 或 AP 监测模式 | 有，全时段 |
| 流氓 AP 检测与定位 | 能发现并定位 |
| 自动遏制 | 可配置（生产需防误伤邻居） |
| 告警时效 | ≤ 约定分钟数 |
| 日志入 SIEM | 认证事件、AP 变更、RADIUS 拒绝、漫游异常、降级尝试 |
| 告警质量 | 能定位到 AP/楼层/信道 |
| 降级与 deauth 监控 | 有专门检测规则 |

---

<!--V4-->


