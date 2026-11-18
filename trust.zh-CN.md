# Trust — 创建 MTC 订单

英文原文：[trust.md](trust.md)

Trust 是 CertLocker 的应用。用户可以在 [trust.certlocker.io](https://trust.certlocker.io) 免费注册。本文是在该应用中创建一笔默克尔树证书订单的操作路径。签发、预发目录和 profile 目录见 [TrustAsia 报告](trust-asia.zh-CN.md)。两条证书记录的存储和安装见 [CertLocker 报告](certlocker.zh-CN.md)。仓库索引是 [README](README.zh-CN.md)。

一笔 MTC 订单就是普通的 ACME 工作流，其目录 profile 指名一种 MTC 产品。Trust 没有单独的 MTC 画面。订单从 **ACME → 出站订单（Outbound orders）** 创建。

应用界面是英文。下文在中文说明后用括号保留界面上的英文标签。

## 1. 下单之前

订单由 Trust 中已经存在的 ACME 客户端发出。

1. 打开 **ACME → ACME 客户端（ACME Clients）**。
2. 打开指向 MTC 目录的客户端。预发互操作环境的目录是 `https://acme.mtc-staging.trustasia.com/acme/v2/directory`。
3. 在客户端详情中使用 **更新配置档（Update Profiles）**。Trust 从该目录刷新 `meta.profiles`，并把名称存到客户端上。
4. 确认该客户端未被暂停。已暂停的客户端不能用于新订单。

2026 年 11 月 18 日，该目录公告的名称是 `mtc-7d-07` 和 `mtc-47d-07`。新订单的 profile 列表应显示这两个名称。8 月的名称 `mtc-47d` 已不再公告；在新订单上选择它会产生 `invalidProfile`。保留的 profile 仍为已经接受的订单提供服务。细节见 [TrustAsia 报告](trust-asia.zh-CN.md)。

`mtc-7d-07` 是 7 天、免于吊销的产品。`mtc-47d-07` 是携带吊销信息的 47 天产品。二者各有自己的 CA 和日志。

## 2. 打开订单表单

1. 打开 **ACME → 出站订单（Outbound orders）**。
2. 选择 **创建（Create）**。

Trust 打开 **创建 ACME 订单（Create ACME order）**。表单分三步：**客户端（Client）**、**证书（Certificates）** 和 **自动化（Automation）**。**下一步（Next）** 和 **返回（Back）** 在步骤之间移动。**创建订单（Create order）** 在最后一步可用，并提交 `POST /acme/workflow-create`。响应中包含工作流 id 时，Trust 打开该工作流。

## 3. 第 1 步 — 客户端

| 字段 | MTC 订单应如何设置 |
| --- | --- |
| ACME 客户端（ACME client） | 第 1 节中的客户端。必填。选中后会加载该客户端的目录 profile。 |
| Profile（可选） | 一个已公告的 MTC profile：`mtc-7d-07` 或 `mtc-47d-07`。只有选中带有 profile 的客户端后，该控件才会出现。**无 profile（默认）（No profile (default)）** 不会向 CA 请求 MTC 产品。 |
| 质询类型（Challenge type） | HTTP-01、DNS-01 或 DNS-PERSIST-01。必填。这决定每个域名授权如何验证。 |
| DNS 客户端（可选） | 在 DNS-01 和 DNS-PERSIST-01 时显示。DNS 客户端创建 TXT 记录。不能与第 3 步的质询自动化同时使用。 |

**下一步（Next）** 要求已选择 ACME 客户端。

## 4. 第 2 步 — 证书

两个证书选择器都保持在“新建证书”上。签发完成时 Trust 创建独立证书记录；CA 签发地标相对证书时 Trust 再创建该记录。关联已有记录，用于订单应附到库存中已有证书的情形。

| 字段 | 新 MTC 订单应如何设置 |
| --- | --- |
| 独立证书（可选） | **新建独立证书（New standalone certificate）**。 |
| 地标证书（可选） | **新建地标证书（New landmark certificate）**。 |
| 证书算法 | RSA、P-256 或 P-384。默认是 P-256。Trust 生成密钥对时使用该算法。 |
| 自动续期（Auto renew） | 默认开启。Trust 在证书寿命的 75% 时续期。 |
| 域名（每行一个） | 本订单的 DNS 名称。每一行成为一条授权。必填。 |
| 有效期（Expiry） | 可选。**不早于（Not before）** 和 **不晚于（Not after）** 必须是今天或更晚。 |
| 私钥（Private Key） | 可选。展开该面板以提供 PKCS#8 密钥，或留空，由 Trust 在签发时生成。 |

**下一步（Next）** 要求至少一个有效域名。

## 5. 第 3 步 — 自动化

这一步的每个控件都是可选的。可以一个都不设置就创建订单。

| 字段 | 何时运行 |
| --- | --- |
| 质询自动化（可选） | 质询就绪之后，适用于任何质询类型。不能与第 1 步的 DNS 客户端同时选择。 |
| 独立证书自动化（可选） | 独立证书签发之后。 |
| 地标自动化（可选） | 地标相对证书签发之后。 |
| 创建访问令牌（Create access token） | 勾选后，Trust 为本订单生成的每张证书创建一个证书访问令牌。 |

**创建订单（Create order）** 启动工作流。

## 6. 工作流显示的内容

新行出现在 **出站订单（Outbound orders）** 上。打开它以跟随时间线。已打开的工作流会随状态变化而更新。每个方块是一条工作流历史：状态，以及 Trust 记录该状态的时间。

状态按顺序为：

1. 订单已排程（Order Scheduled）
2. 订单待处理（Order Pending）
3. 订单已提交（Order Submitted）
4. 质询等待中（Challenges Waiting）
5. 质询已触发（Challenges Triggered）
6. 质询已验证（Challenges Verified）
7. 提交最终订单（Issue Final Order）
8. 等待订单（Wait for Order）
9. 证书已签发（Certificate Issued）
10. 等待地标（Wait for Landmark）
11. 地标已签发（Landmark Issued）
12. 完成（Complete）

对 MTC 订单，最后四个状态是与传统证书不同的部分：

- **证书已签发（Certificate Issued）。** Trust 已存储独立证书。该方块链接到这条记录。地标仍在等待时，独立证书就可以安装。
- **等待地标（Wait for Landmark）。** CA 尚未分配地标。ACME 服务器返回了 `Retry-After` 时，该方块会显示 **在此时间后重试（Retry after）**，直到该时刻。Trust 保留独立证书记录并继续轮询。
- **地标已签发（Landmark Issued）。** Trust 已把地标相对证书存为第二条记录，关联到同一订单和同一名称。该方块链接到这条记录。独立证书记录仍然保留。
- **完成（Complete）。** 两条记录都存在，并且已经关联。

详情中还会显示 ACME 客户端；使用了 DNS 客户端时显示该客户端；选择了 profile 时显示 profile 名称和说明。

**等待订单（Wait for Order）** 和 **等待地标（Wait for Landmark）** 是两次不同的等待。等待订单是独立证书之前的延迟。等待地标是同一订单上第二张证书之前的延迟。续期是由自动续期驱动的后续订单，不是这次地标等待。

失败时，工作流页脚在最后一个方块下方显示消息、错误，以及任何 ACME 问题的子问题。**取消订单（Cancel order）** 停止仍在进行中的工作流。**删除订单（Delete order）** 移除该工作流记录。

## 7. 证书去向

证书已签发和地标已签发是两条库存记录，不是两笔订单。用地标字节替换独立证书字节，会丢掉尚无地标的依赖方仍然需要的共签。Trust 保留两条记录，并把它们关联起来。

profile 字符串与订单一起存储。在 `mtc-47d` 下签发的证书和在 `mtc-47d-07` 下签发的证书是不同的库存项，来自不同的 CA 和日志。

每条记录的安装，以及安装后 Trust 所做的握手探测，见 [CertLocker 报告](certlocker.zh-CN.md)。
