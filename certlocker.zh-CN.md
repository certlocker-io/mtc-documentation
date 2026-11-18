# CertLocker — 存储与安装

英文原文：[certlocker.md](certlocker.md)

CertLocker 是 ACME 客户端和证书管理器。本文涵盖独立证书与地标相对证书的取回、保留和安装，以及 CertLocker 应用 Trust 所做的 TLS 握手探测。签发、预发目录和 profile 目录见 [TrustAsia 报告](trust-asia.zh-CN.md)。仓库索引是 [README](README.zh-CN.md)。

![CertLocker](images/certlocker-logo.svg)

## 1. 实现与测试范围

| 项 | 值 |
| --- | --- |
| 角色 | 一笔订单上两张证书的 ACME 客户端、库存和安装 |
| 预发目录 | `https://acme.mtc-staging.trustasia.com/acme/v2/directory` |
| 运行 | 2026 年 8 月。公开描述：[CertLocker，2026 年 10 月 1 日](https://certlocker.io/blog/wake-up-200-day-certificates-mtcs/) |
| 标识符 | `mtc.certlocker.dev` |
| 该订单发送的 profile | `mtc-47d` |
| 记录的工作流状态 | 证书已签发，然后是地标等待，然后是地标已签发，然后是完成 |
| 证据目标 | [draft-elkins-plants-enterprise-mtc-deployment-01](https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01) 第 11 节测试 1 的取回与保留一半，以及 [第 5.3.11.3 节](https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01#section-5.3.11.3) 中的证书管理器一栏 |
| 联系方式 | [mtc@certlocker.io](mailto:mtc@certlocker.io) |

支持级别：**已测试的互操作性，仅限预发环境**。该订单针对 TrustAsia 的预发 CA。所得证书继承该 CA 在 Chrome 抗量子根计划中的 `UNTRUSTED_VALIDATION_ONLY` 状态。

CertLocker 自己的 ACME 目录为 CertLocker 签发的证书公告 `mtc7d` 和 `mtc47d`。8 月的订单发往 TrustAsia 预发目录，profile 字符串是 TrustAsia 的 `mtc-47d`。

8 月的服务器是 TrustAsia 对照草案第 05 版所描述的修订。本文中的节号来自 [draft-ietf-plants-merkle-tree-certs-08](https://www.ietf.org/archive/id/draft-ietf-plants-merkle-tree-certs-08.html)。

## 2. 客户端原样复用的部分，以及它为 MTC 所做的事

账户、订单、授权、质询和最终化仍是未改变的 RFC 8555。在该订单之上，MTC 特有的客户端行为是：

1. 发现 `meta.profiles`，并按 [draft-ietf-acme-profiles](https://datatracker.ietf.org/doc/draft-ietf-acme-profiles/) 在 `newOrder` 中发送所选名称。8 月的订单发送了 `mtc-47d`。
2. 订单为 `valid` 时，下载订单 `certificate` URL 上的证书。该响应是独立证书。CertLocker 把它存为该名称的标准证书。
3. 跟随同一订单的地标资源。该资源尚未就绪时，保留独立证书记录并继续轮询。8 月的工作流把这一点显示为证书已签发与地标已签发之间的间隔。
4. 把地标相对证书存为第二条记录，关联到同一订单和同一名称。完成意味着两条记录都存在并且已经关联。

MTC 草案第 9.1 节把尚未就绪的地标定义为 `acme-optional-alternate` URL 上带有 `Retry-After` 的 HTTP 202，并要求客户端独立于独立证书重试该 URL。8 月的状态符合这一形态：独立证书记录先于地标记录存在，地标等待是同一工作流的一部分。那次运行的 `Link` 头和 202 响应没有提交到本仓库，因此这里的头名称是草案为该工作流所表现出的形态使用的名称。

下载时，支持 MTC 的客户端发送 `Accept: application/pem-certificate-chain-with-properties`（第 9.2 节），以便响应可以携带 `CertificatePropertyList`。属性列表是信任锚标识符随字节一起传送的方式。8 月订单捕获的属性列表尚未进入本仓库。该订单的库存记录 profile、标识符，以及两条证书记录之间的关联。

## 3. 两条记录

一次签发产生两张证书。CertLocker 把它们保持为两条记录和一条工作流。

| 工作流状态 | 记录 | 草案工件 | 对安装的影响 |
| --- | --- | --- | --- |
| 证书已签发 | `mtc.certlocker.dev` 的标准证书 | 独立证书，第 6.3 节。包含证明和共签在证书内部。 | 可以安装。只有 CA 共签方配置的依赖方可以使用该工件。 |
| 地标等待 | 独立证书记录不变 | `acme-optional-alternate` 仍未就绪 | 等待延长工作流。它不撤回独立证书记录。 |
| 地标已签发 | 第二条记录，同一订单，同一名称 | 地标相对证书，第 6.4 节。指向地标子树的包含证明。 | 安装在独立证书旁边。第 6.4 节要求认证方在部署任何地标相对证书的同时部署一张独立证书。 |
| 完成 | 两条记录已关联 | 同一次签发，两种证明 | 库存可以从该工作流打开任一证书。 |

地标证书是已经存储的日志条目的另一种证明。用地标字节替换独立证书字节，会丢掉尚无地标的依赖方仍然需要的共签。8 月的工作流关联了这些记录，并让两者都保留在原处。

与这一对一起保留的元数据，如该工作流所示：

- DNS 标识符
- 订单上发送的 profile 名称（本次运行为 `mtc-47d`）
- 产生两张证书的那笔订单
- 哪条记录是独立证书，哪条是地标相对证书

后来在 `mtc-47d-07` 下的订单是不同的 CA 和日志。profile 字符串是记录的一部分，以便两代保持区别。profile 过渡见 [TrustAsia 报告](trust-asia.zh-CN.md)。

## 4. 取回、保留与验证

测试 1 要求证书管理器保留工件字节和元数据。这是存储要求。检查包含证明和共签是依赖方的要求，见 MTC 草案第 4.3 节和第 7.2 节。

| 操作 | CertLocker 在本次测试中所做的事 |
| --- | --- |
| 取回 | 从订单下载独立证书，等待之后再下载地标相对证书。 |
| 字节保留 | 把每次响应存为自己的证书记录。地标响应没有覆盖独立证书记录。 |
| 元数据 | 工作流历史显示证书已签发、等待、地标已签发和完成，并从工作流链接到两条证书记录。 |
| 握手探测 | Trust 对端点打开 TLS 握手，并判断该握手是否会支持 MTC。会支持时，Trust 提供 MTC。探测随后把正在提供的指纹与 Trust 中存储的指纹相比较。 |
| 包含证明和共签 | 保留在已存储的证书内部，供依赖方评估。管理器的结果是被保留的工件。Chrome 或其他具备 MTC 能力的依赖方执行密码学检查。 |

部署草案第 5.1 节把库存与实时握手所呈现的内容分开。处于完成状态的工作流意味着两件工件都在库存中。Trust 是针对握手的检查：它进行探测，在该握手能够支持 MTC 时提供 MTC，并比较指纹，以确认正在提供的证书就是 Trust 所存储的证书。

## 5. 安装

安装按记录进行。

**独立证书。** 在证书已签发时，标准证书及其私钥是该名称的凭据。交付沿用 CertLocker 已经用于 ACME 证书的路径：把密钥和证书字节写到目标，重新加载服务，并探测监听端。证书字节就是 ACME 响应，原样写入。地标等待与这次写入无关。第 9.1 节要求客户端不要因为可选替代证书而阻塞已可用证书的部署。

**地标相对证书。** 在地标已签发时，第二张证书作为同一名称的附加凭据交付。独立证书凭据保留。MTC 草案第 8 节是同时持有两者的服务器的选择规则：ClientHello 指示地标时使用地标相对证书，指示 CA 时使用独立证书。

**自动化。** 两条记录由同一套无需安装代理的机制安装，见 [Conquering the Last Mile](https://certlocker.io/blog/conquering-the-last-mile/)。远程机器运行 SSH。CertLocker 不在其上安装代理。一段短脚本指定要写入的文件和激活它们的命令。`install` 通过 SFTP 复制证书或密钥，`exec` 通过 SSH 运行一条命令，`audit` 记录结果。每次 `exec` 都是一次新的 SSH 会话。自动化附在订单上，并针对所选记录运行，因此独立证书和地标相对证书各自保留自己的字节。一台安装了 nginx 的主机如下：

```
install certificate.chain to /etc/nginx/ssl/fullchain.pem mode 644
install certificate.privateKey to /etc/nginx/ssl/privkey.pem mode 600
exec sudo nginx -t
exec sudo systemctl reload nginx
audit true Deployed certificate.name to host.name
```

路径和重新加载命令随服务而变。Apache、HAProxy，以及通过 Windows OpenSSH 的 IIS，使用同一种语言。访问使用限定范围的主机令牌，路径可以经过堡垒机，只要目标正在运行 SSH。一条自动化把同一张证书安装到每一台选定的主机上，因此由多台机器提供的一个名称，就是这一段脚本的多节点铺开。Trust 随后探测每台主机，并把正在提供的指纹与它所存储的指纹相比较。

**Trust。** [Trust](https://trust.certlocker.io) 是 CertLocker 在证书存储之后处理握手的应用。[mtcdemo.trustasia.cn](https://mtcdemo.trustasia.cn/) 是已经在 TLS 层处理默克尔树证书的端点：一次握手，一张被选中的证书，如 [TrustAsia 报告](trust-asia.zh-CN.md) 所述。Trust 探测该握手，并判断它是否会支持 MTC。会支持时，Trust 提供 MTC。不会支持时，握手停留在传统证书上。

探测也检查安装是否正确。对每张已存储的证书，包括独立证书记录和地标相对记录，Trust 把它所持有的指纹与端点正在提供的指纹相比较。匹配意味着监听端上的证书就是 Trust 中的证书。证书临近过期时 Trust 告警。默认情况下，证书寿命经过 75% 后，它自动续期该证书，安装替换证书，并再次探测。

8 月的公开记录显示这一序列的存储一侧：两条记录都存在，独立证书记录早于地标记录，并且只有在它们关联之后工作流才完成。已公布的记录中没有某个具名代理在每次状态转换时的重新加载日志。与第 5.1 节对应的握手检查是 Trust 的探测，使用上面的指纹比较。

## 6. 互操作发现

| 发现 | 对 ACME 客户端或证书管理器的影响 |
| --- | --- |
| 独立证书和地标相对证书在不同时间到达，并且属于同一笔订单。 | 建模为两个已存储工件和一次签发。在二者之间呈现一个状态。8 月的名称是证书已签发、地标等待、地标已签发和完成。 |
| 地标等待可以长于独立证书已经可用的时刻。 | 独立证书一到达就持久保存该记录，并继续轮询地标 URL。对新部署，第 9.1 节规定在该轮询继续时安装独立证书。续期时，只有在已经在服务中的证书仍然有效期间，客户端才可以等待地标。 |
| 这次运行之后，该目录上的 profile 名称发生了变化。 | 8 月的请求使用 `mtc-47d`。2026 年 11 月 18 日，使用该名称的新订单会产生 `invalidProfile`。在 `newOrder` 之前刷新 `meta.profiles`。在已存储的记录上保留原来的 profile 字符串。细节见 [TrustAsia 报告](trust-asia.zh-CN.md)。 |
| 目录上有 `renewalInfo`，地标有自己的 `Retry-After`。 | 按地标自己的时间表轮询地标 URL。这次等待是当前订单的第二张证书。Trust 的续期是后续订单：默认在证书寿命的 75% 时运行。 |
| 存在未知的目录成员。 | 忽略它们。预发目录包含一个，记录在 TrustAsia 报告中。 |
| 两代 47 天 profile 使用不同的 CA 和日志。 | 已存储的 `mtc-47d` 证书和一张新的 `mtc-47d-07` 证书是分开的库存项。安装新的一张，并不会仅凭 profile 名称就停用旧证书的信任锚配置。 |

## 7. 复现与限制

### 已公布的 8 月运行

1. 把 ACME 客户端指向 `https://acme.mtc-staging.trustasia.com/acme/v2/directory`。
2. 为测试者所控制的名称创建订单。已公布的订单使用 `mtc.certlocker.dev` 和 `"profile": "mtc-47d"`。
3. 完成域名控制验证和最终化。
4. 把订单 `certificate` URL 的正文存为独立证书记录。一并记录 `Link` 头。
5. 轮询地标 URL，直到它返回一张证书。把该正文存为第二条记录。把两条记录都关联到该订单。
6. 安装独立证书，不必等第 5 步结束。第 5 步返回时，把地标证书安装在它旁边。
7. 从 Trust 探测监听端的 TLS 握手。当握手能够支持 MTC 时，提供 MTC。把正在提供的指纹与 Trust 中存储的指纹相比较。

今天重复第 2 步，需要目录所公告的 profile。2026 年 11 月 18 日这些名称是 `mtc-7d-07` 和 `mtc-47d-07`。先获取目录，再使用该集合。

### 本仓库今天可以展示的内容

已公布的证据是工作流结果和预发目录：

- 标识符 `mtc.certlocker.dev`
- profile `mtc-47d`
- 状态：证书已签发、地标等待、地标已签发、完成
- 两条已关联的证书记录
- 2026 年 11 月 18 日获取的目录和 profile 页面，引用在 [TrustAsia 报告](trust-asia.zh-CN.md) 中
- `newOrder` 请求和订单对象
- 独立证书响应，包括 `Link` 和 `Retry-After`
- 地标响应
- 每条记录已存储的 PEM，账户标识符已去除
- 接受安装的监听端的探测观察

### 有待进一步测试的内容

| 部署草案测试 | 本客户端的状态 |
| --- | --- |
| 测试 1：注册并保留独立工件，并同时保留同一订单的地标相对工件 | 8 月已在预发 CA 上测试。字节样本仍待加入本仓库。 |
| 测试 1：实时确认端点呈现的是已存储的证书 | Trust 探测握手，在握手支持 MTC 时提供 MTC，并把正在提供的指纹与 Trust 中存储的指纹相比较。`mtc.certlocker.dev` 的一次捕获探测结果仍待加入。 |
| 测试 2–4 和 10：握手期间的信任锚标识符选择 | 由 [mtcdemo.trustasia.cn](https://mtcdemo.trustasia.cn/) 演练，它在 TLS 内部选择证书。Trust 探测该握手，并在握手能够支持 MTC 时提供 MTC。省略 `trust_anchors` 的客户端从同一主机收到传统证书。 |
| 测试 5：续期 | Trust 默认在证书寿命的 75% 时续期，然后由探测检查正在提供的指纹。证书临近过期时 Trust 也会告警。 |
| 测试 6、7 和 12：吊销、紧急撤回、商业支持和故障恢复 | 有待进一步测试。 |

## 8. 参考文献

- MTC 草案，第 9 节：<https://www.ietf.org/archive/id/draft-ietf-plants-merkle-tree-certs-08.html#section-9>
- 部署草案，测试 1：<https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01#section-11>
- 8 月工作流：<https://certlocker.io/blog/wake-up-200-day-certificates-mtcs/>
- Trust：<https://trust.certlocker.io>
- 安装自动化：<https://certlocker.io/blog/conquering-the-last-mile/>
- 预发目录：<https://acme.mtc-staging.trustasia.com/acme/v2/directory>
- 联系方式：<mailto:mtc@certlocker.io>
