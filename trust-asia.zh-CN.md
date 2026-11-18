# TrustAsia — 独立默克尔树证书的 ACME 签发

英文原文：[trust-asia.md](trust-asia.md)

TrustAsia 运营预发认证机构。本文是签发报告。存储、字节保留和安装见 [CertLocker 报告](certlocker.zh-CN.md)。仓库索引是 [README](README.zh-CN.md)。

![TrustAsia](images/trustasia-logo.svg)

## 1. 实现与测试范围

| 项 | 值 |
| --- | --- |
| 角色 | 预发互操作环境的 ACME 服务器和 MTC CA |
| 所观察的目录 | `https://acme.mtc-staging.trustasia.com/acme/v2/directory` |
| 目录和 profile 页面的观察日期 | 2026 年 11 月 18 日 |
| 客户端互操作运行 | 2026 年 8 月，与 [CertLocker](certlocker.zh-CN.md) 一起，标识符 `mtc.certlocker.dev`，profile 名称 `mtc-47d` |
| 节号所引用的协议文本 | [draft-ietf-plants-merkle-tree-certs-08](https://www.ietf.org/archive/id/draft-ietf-plants-merkle-tree-certs-08.html)（2026 年 11 月 1 日） |
| profile 协商 | [draft-ietf-acme-profiles](https://datatracker.ietf.org/doc/draft-ietf-acme-profiles/) |
| 证据目标 | [draft-elkins-plants-enterprise-mtc-deployment-01](https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01) 的第 5.1 节、第 5.3.11.3 节，以及第 11 节测试 1 的签发一半 |

两次公开描述把 8 月的运行夹在中间：

- 2026 年 7 月 18 日，TrustAsia 对照 [draft-ietf-plants-merkle-tree-certs-05](https://datatracker.ietf.org/doc/draft-ietf-plants-merkle-tree-certs/) 描述了预发环境。同一描述也出现在 PQWS 研讨会笔记 [slides-pqws-s26-00](https://www.ietf.org/slides/slides-pqws-s26-00.pdf) 中。
- 2026 年 11 月 18 日获取的公开演示 [mtcdemo.trustasia.cn](https://mtcdemo.trustasia.cn/) 说明，该 TLS 演示实现的是草案第 07 版的 X.509 封装。同一天获取的目录提供草案第 07 版的 profile 名称 `mtc-7d-07` 和 `mtc-47d-07`。

因此，8 月的 CertLocker 订单针对的是当时部署的服务器。TrustAsia 曾相对草案第 05 版描述该服务器，订单选择的是不带草案后缀的 profile 名称 `mtc-47d`。第 7 节中的目录文档是 2026 年 11 月 18 日观察到的服务器。这是同一预发服务的不同修订。

第 5.3.11.3 节的支持级别：**已测试的互操作性，仅限预发环境**。预发 CA 和 TrustAsia 镜像列在 Chrome 抗量子根计划中，状态为 `UNTRUSTED_VALIDATION_ONLY`。本报告记录的结果是针对该范围的注册。就第 5.3.11.3 节而言，生产部署有待根计划默认分发这些信任锚，并有待围绕这些信任锚的受支持产品栈。

TrustAsia 还维护 acme.sh 的一个支持 MTC 的分支：[`trustasia-com/acme.sh`，分支 `feat/mtc-certificate-support`](https://github.com/trustasia-com/acme.sh/tree/feat/mtc-certificate-support)。该客户端独立于 CertLocker。8 月的结果是针对同一预发目录的第二个、单独的 ACME 客户端。

## 2. 预发 CA 签发什么

该服务有两个测试 CA。每个 CA 有自己的签发日志。地标分配和子树共签发生在日志追加之后；独立证书是携带包含证明和共签的工件，如 MTC 草案第 6.3 节所述。同一日志条目的地标相对证书在随后的地标之后产生，并从同一笔 ACME 订单提供，如第 9.2 节所述。

profile 页面给出产品区别：

| profile 系列 | 有效期 | 吊销 | CA 与共签方算法 |
| --- | --- | --- | --- |
| `mtc-7d`，以及草案第 07 版名称 `mtc-7d-07` | 7 天 | 免除 | ML-DSA-44 |
| `mtc-47d`，以及草案第 07 版名称 `mtc-47d-07` | 47 天 | 必需 | ML-DSA-44 |

profile 页面上的算法一行指的是 CA 和共签方。终端实体公钥算法是已签发证书的属性，将在加入样本 PEM 时与该 PEM 一起记录。

演示公布了预发环境的这些 CA 标识符。它们是依赖方按第 5.1 节配置的 CA ID：

| 实体 | OID |
| --- | --- |
| 7 天 CA | `1.3.6.1.4.1.44494.3.1.3` |
| 47 天 CA | `1.3.6.1.4.1.44494.3.1.4` |
| 7 天日志 1 | `1.3.6.1.4.1.44494.3.1.3.0.1` |
| 47 天日志 1 | `1.3.6.1.4.1.44494.3.1.4.0.1` |
| 镜像共签方 | `1.3.6.1.4.1.44494.3.2.1` |

如 PQWS 笔记所述，Geomys 为两条签发日志共签。因此，独立证书除 TrustAsia 镜像之外还带有一个外部共签。8 月证书解码后的共签方列表，是仍待加入的样本 PEM 的一部分。

## 3. ACME 工作流，以及为 MTC 改变了什么

### 原样复用、没有 MTC 专用动词的部分

账户注册、订单创建、授权、质询验证、最终化、订单的 `certificate` URL、吊销和密钥更换，都是 RFC 8555 资源。目录还公布 `renewalInfo`（[RFC 9773](https://www.rfc-editor.org/rfc/rfc9773)）。续期调度是后续测试；此处记录该资源的公告，是为了让客户端能看到它，它不属于 8 月的签发结果。

2026 年 11 月 18 日获取的目录 `meta` 对象包含 `termsOfService`、`website`、`caaIdentities`（`trustasia.com`）和 `profiles`。它不包含 `externalAccountRequired`。

还存在一个额外的目录成员。它的值是 Let's Encrypt 关于随机目录条目的讨论：`https://community.letsencrypt.org/t/adding-random-entries-to-the-directory/33417`。要求目录键集合封闭的客户端会在这台服务器上失败。忽略未知目录成员的客户端可以互操作。该成员名称不是稳定的 API。

### 服务器上 MTC 特有的部分

客户端通过把 `newOrder` 上的 `profile` 设为目录当前公告的名称来请求 MTC。profile 页面说明，未知或不兼容的名称会产生 `invalidProfile`。地标相对证书是同一订单的可选替代证书。

MTC 草案第 9.2 节是本环境正在实现的服务器行为：

1. 独立证书就绪时，订单变为 `valid`。订单的 `certificate` URL 提供该独立证书。
2. 独立证书响应携带一个指向地标相对证书的 `acme-optional-alternate` 链接。
3. 在下一个地标分配之前，该 URL 以 HTTP 202 和 `Retry-After` 响应。
4. 地标之后，同一 URL 提供地标相对证书。
5. 理解 MTC 的客户端发送 `Accept: application/pem-certificate-chain-with-properties`，并收到 `CertificatePropertyList`，其中包含独立证书和地标相对证书的信任锚标识符。

8 月的 CertLocker 运行为一笔订单存储了独立证书，随后又存储了地标证书。该结果符合这一形态。那次运行捕获的 `Link` 头和 `CertificatePropertyList` 尚未进入本仓库，因此本文不引用那些字节。下面的目录和 profile 文档已经获取并加以引用。

### profile 名称带有修订限定

profile 页面记录四个名称：

- `mtc-7d` 和 `mtc-47d`，不带草案后缀的名称。
- `mtc-7d-07` 和 `mtc-47d-07`，被描述为草案第 07 版的 profile，各自有自己的 CA 和日志。

该页面还说明过渡规则：保留的 profile 继续为已接受的订单和证书提供服务，新订单只使用目录公告的 profile。

2026 年 11 月 18 日，目录只公告：

```text
mtc-7d-07
mtc-47d-07
```

8 月的互操作选择了 `mtc-47d`。仍在新订单上发送 `mtc-47d` 的客户端会收到 `invalidProfile`。想要同一种 47 天产品的客户端应发送 `mtc-47d-07`，并跟随目录，而不是跟随缓存的名称。profile 页面自己的 `newOrder` 示例仍显示 `"profile": "mtc-7d"`，该名称已不再公告。目录具有权威性；该页面上的示例已经过时。

## 4. 服务器返回的工件

两种工件都是 X.509 证书，其证明是 MTC 证明，如 MTC 草案第 6.2 节所述。

| 工件 | 何时提供 | 携带什么 |
| --- | --- | --- |
| 独立证书 | 订单变为 `valid` 之后的订单 `certificate` URL | 指向已共签子树的包含证明，以及依赖方接受该子树所需的共签。第 6.3 节。 |
| 地标相对证书 | 地标分配之后的 `acme-optional-alternate` URL | 指向地标子树的包含证明。共签被省略，因为预期依赖方已经持有该地标。第 6.4 节。 |

独立证书是部署草案第 5.1 节要求客户端取回的完整工件。地标相对证书是同一次签发的第二种编码，在同一订单上返回。它更小，对尚未持有地标的依赖方没有用。

服务器当前的 profile 页面把 7 天和 47 天的草案第 07 版 profile 都描述为各有自己的 CA 和日志。在 `mtc-47d` 下签发的证书和在 `mtc-47d-07` 下签发的证书，不是来自同一条日志的可互换工件。客户端和库存应把 profile 名称与字节一起存储。

本修订不包含样本证书。公布一份样本需要来自已接受预发订单的、经过净化的 PEM，并去掉账户 URL 和任何订户数据。如果保留了 8 月订单的字节，它们会在保留的 `mtc-47d` profile 下提供，并且需要与在 `mtc-47d-07` 下签发的证书分开标注。

## 5. 演示主机上的 TLS 握手

ACME 服务返回证书。[mtcdemo.trustasia.cn](https://mtcdemo.trustasia.cn/) 在 TLS 层处理默克尔树证书。主机名 `mtcdemo.trustasia.com` 不能解析；应答的演示是 `.cn` 主机。

该主机为一个身份持有三套凭据：一条传统 X.509 链、一张独立 MTC 和一张地标相对 MTC。每次 TLS 1.3 握手，它读取 ClientHello 中的 `trust_anchors` 并选择其中一套，如 MTC 草案第 8 节所述：

| ClientHello | 服务器发送的证书 |
| --- | --- |
| 覆盖本证书的地标标识符或地标组 | 地标相对证书 |
| MTC CA 标识符，且没有可用的地标匹配 | 独立证书，携带包含证明和共签 |
| 没有匹配的 MTC 信任锚，包括省略 `trust_anchors` 的 ClientHello | 传统 X.509 链 |

服务器为该连接发送一张证书。2026 年 11 月 18 日获取的页面对照草案第 07 版描述这一选择，并记录：当 ClientHello 携带 `trust_anchors` 时，演示发送独立证书。

CertLocker 的应用 Trust 探测这一握手。当握手能够支持 MTC 时，Trust 提供 MTC。该行为见 [CertLocker 报告](certlocker.zh-CN.md)。

2026 年 9 月 3 日，带有 MTC 测试标志和 `cosigners.json` 2.0.6 的 Chrome Canary 155.0.8039.0，在该主机上验证了一张带有 Geomys 共签的独立证书，以及一张地标相对证书。

## 6. 互操作发现

| 发现 | 实现者应如何做 |
| --- | --- |
| profile 标识符增加了草案后缀。`mtc-47d` 签发了 8 月的证书。新订单需要 `mtc-47d-07` 或 `mtc-7d-07`。 | 每一笔新订单都读取 `meta.profiles`。把缓存的 profile 名称当作提示。遇到 `invalidProfile` 时刷新目录。 |
| 保留的 profile 仍为旧订单提供服务。 | 继续使用该订单已有的 `certificate` URL。不要仅仅因为目录从 `meta.profiles` 中去掉了旧名称就重新签发。 |
| profile 的 HTML 示例仍写着 `mtc-7d`。 | 以目录为准，不以 HTML 中的示例为准。 |
| 目录包含一个无法识别的成员，指向关于随机目录条目的讨论。 | 忽略未知的目录成员。 |
| 已公布 `renewalInfo`。 | 把 ARI 与地标取回分开实现。`acme-optional-alternate` 上的 `Retry-After` 是当前订单第二张证书的延迟。`renewalInfo` 是下一笔订单的窗口。 |
| 7 天产品和 47 天产品在吊销、寿命、CA 和日志上不同。 | 把 profile 和 CA ID 与证书一起存储。7 天工件属于免于吊销的类别。47 天工件属于携带吊销信息的类别。 |
| 草案第 07 版的 profile 使用与较早名称不同的 CA 和日志。 | 为较早预发 CA 配置的信任锚不会自动覆盖 `mtc-47d-07`。应在已签发证书中确认 CA ID。 |
| 地标晚于独立证书，这是有意的。 | 独立证书就绪时把订单置为 `valid`。把地标作为可选替代证书提供。该 URL 上的 HTTP 202 是规定的等待状态。 |

8 月的客户端选择了 `mtc-47d`，收到独立证书，等待，然后收到地标证书。名称变更这一发现来自该次运行之后获取的目录。

## 7. 可复现的服务器证据

2026 年 11 月 18 日以未认证的 `GET` 收集。为本说明没有创建账户。

### 目录

`GET https://acme.mtc-staging.trustasia.com/acme/v2/directory`

响应 `200`，`content-type: application/json`。随机目录成员已省略；它的值是第 3 节引用的社区 URL。其余每个字段均按返回原样列出：

```json
{
  "newNonce": "https://acme.mtc-staging.trustasia.com/acme/v2/new-nonce",
  "newAccount": "https://acme.mtc-staging.trustasia.com/acme/v2/new-account",
  "newOrder": "https://acme.mtc-staging.trustasia.com/acme/v2/new-order",
  "revokeCert": "https://acme.mtc-staging.trustasia.com/acme/v2/revoke-cert",
  "keyChange": "https://acme.mtc-staging.trustasia.com/acme/v2/key-change",
  "renewalInfo": "https://acme.mtc-staging.trustasia.com/acme/v2/renewal-info",
  "meta": {
    "termsOfService": "https://repository.trustasia.com/repo/sa/TrustAsia-SA_EN_V1.2.3.pdf",
    "website": "https://acme.mtc-staging.trustasia.com/",
    "caaIdentities": ["trustasia.com"],
    "profiles": {
      "mtc-7d-07": "https://acme.mtc-staging.trustasia.com/acme/v2/profiles#mtc-7d-07",
      "mtc-47d-07": "https://acme.mtc-staging.trustasia.com/acme/v2/profiles#mtc-47d-07"
    }
  }
}
```

### profile 文档

`GET https://acme.mtc-staging.trustasia.com/acme/v2/profiles` 返回 `text/html`，描述 `mtc-7d`、`mtc-47d`、`mtc-7d-07` 和 `mtc-47d-07`，并说明新订单跟随目录。

该页对 `newOrder` 的图示使用了当前目录不再公告的名称：

```json
{
  "profile": "mtc-7d",
  "identifiers": [{"type": "dns", "value": "www.example.org"}]
}
```

针对 2026 年 11 月 18 日所观察目录的新订单，使用一个已公告的名称：

```json
{
  "profile": "mtc-47d-07",
  "identifiers": [{"type": "dns", "value": "www.example.org"}]
}
```

`www.example.org` 是文档用标识符。它不是 8 月运行所用的名称。8 月的名称是 `mtc.certlocker.dev`，`"profile": "mtc-47d"`。

### 重复服务器观察的步骤

1. `GET` 目录 URL。记录 `meta.profiles` 和日期。
2. `GET` `meta.profiles` 所链接的 profile URL。记录有效期、吊销类别，以及 HTML 仍在记录哪些名称。
3. 把公告集合与任何从较早订单保存的 profile 名称相比较。
4. 要重复签发，注册账户，用一个已公告的 profile 创建订单，为测试者所控制的名称完成质询，并最终化。保存订单 URL、包含 `Link` 头的 `certificate` 响应、`acme-optional-alternate` URL 的第一次响应，以及最终的地标响应。在提交到本仓库之前，去掉账户密钥和订户联系数据。

### 本签发报告的已知限制

- 所引用的服务器文档来自 2026 年 11 月 18 日。8 月的证书在 `mtc-47d` 下签发，当时目录尚未缩减为 `-07` 名称。
- 经过净化的独立证书 PEM、它的 `Link` 头、HTTP 202 地标响应，以及地标 PEM，尚未进入本仓库。
- 终端实体密钥算法、证明长度和共签方集合需要那份 PEM。此处不作推断。
- 演示主机上的握手选择是第 5 节。ARI 续期、吊销和故障恢复仍是进一步测试。
- 预发账户的联系方式是 `mtc@trustasia.com`。

## 8. 参考文献

- MTC 草案：<https://datatracker.ietf.org/doc/draft-ietf-plants-merkle-tree-certs/>
- 第 9.2 节，ACME：<https://www.ietf.org/archive/id/draft-ietf-plants-merkle-tree-certs-08.html#section-9.2>
- 第 5.1 节，CA 标识符：<https://www.ietf.org/archive/id/draft-ietf-plants-merkle-tree-certs-08.html#section-5.1>
- ACME profiles：<https://datatracker.ietf.org/doc/draft-ietf-acme-profiles/>
- 部署草案，第 5.1 节、第 5.3.11.3 节和第 11 节：<https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01>
- 预发目录：<https://acme.mtc-staging.trustasia.com/acme/v2/directory>
- profile 页面：<https://acme.mtc-staging.trustasia.com/acme/v2/profiles>
- 演示：<https://mtcdemo.trustasia.cn/>
