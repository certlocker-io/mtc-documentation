# 独立 MTC 的 ACME 互操作性

英文原文：[README.md](README.md)

本文记录通过 ACME 签发并取回独立默克尔树证书（Merkle Tree Certificate，MTC）的实现证据，以及独立证书与随后的地标相对证书的存储和安装。

本仓库是 [TrustAsia](trust-asia.zh-CN.md)（签发）与 [CertLocker](certlocker.zh-CN.md)（存储和安装）的联合说明。

读者应已了解 ACME（[RFC 8555](https://www.rfc-editor.org/rfc/rfc8555)）以及 PLANTS 默克尔树证书相关工作。

## 规范

| 文档 | 在本报告中的作用 |
| --- | --- |
| [draft-ietf-plants-merkle-tree-certs-08](https://datatracker.ietf.org/doc/draft-ietf-plants-merkle-tree-certs/) | 协议。ACME 签发以及 `acme-optional-alternate` 取回方式见第 9 节。CA 标识符见第 5.1 节。独立证书与地标相对证书的构造见第 6.3 节和第 6.4 节。 |
| [draft-ietf-acme-profiles](https://datatracker.ietf.org/doc/draft-ietf-acme-profiles/) | 客户端在 `newOrder` 上如何指名 MTC 产品。 |
| [draft-elkins-plants-enterprise-mtc-deployment-01](https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01) | 要求提供本证据的部署草案。 |

以下结果是该部署草案这些部分的实现证据：

- [第 5.1 节](https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01#section-5.1)：签发并取回完整的独立工件，并由证书管理器保留该工件。
- [第 5.3.11.3 节](https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01#section-5.3.11.3)：由实际执行测试的各方给出的厂商结果，含明确的支持级别、版本和依赖。
- [第 11 节](https://datatracker.ietf.org/doc/html/draft-elkins-plants-enterprise-mtc-deployment-01#section-11)测试 1 中的签发与取回部分：向兼容的 CA 注册一张独立 MTC，并在证书管理器中保留工件字节和元数据。

## 报告

| 方 | 范围 | 报告 |
| --- | --- | --- |
| [![TrustAsia](images/trustasia-logo.svg)](trust-asia.zh-CN.md) | ACME 服务器、签发日志、共签、独立证书，以及随后的地标相对证书 | [trust-asia.zh-CN.md](trust-asia.zh-CN.md) |
| [![CertLocker](images/certlocker-logo.svg)](certlocker.zh-CN.md) | profile 选择、取回、字节保留、成对记录，以及两张证书的安装。联系 [mtc@certlocker.io](mailto:mtc@certlocker.io) | [certlocker.zh-CN.md](certlocker.zh-CN.md) |

## 实际演练的内容

2026 年 8 月，CertLocker 向 TrustAsia 的 MTC 预发 CA 为 `mtc.certlocker.dev` 完成了一笔 ACME 订单，使用的 profile 名称为 `mtc-47d`。该工作流记录了独立证书，等待之后，又记录了地标相对证书，并将两条记录关联起来。这次运行是预发环境的互操作性结果。这些预发 CA 列在 Chrome 抗量子根计划中，状态为 `UNTRUSTED_VALIDATION_ONLY`。

2026 年 11 月 18 日，同一预发目录为新订单公告 `mtc-7d-07` 和 `mtc-47d-07`。8 月的 profile 名称仍为已经接受的订单保留。细节和目录文档见 TrustAsia 报告。

[mtcdemo.trustasia.cn](https://mtcdemo.trustasia.cn/) 在 TLS 层处理默克尔树证书。该服务器持有一张传统证书、一张独立 MTC 和一张地标相对 MTC，并根据 ClientHello 中的 `trust_anchors` 选择其中一张。CertLocker 的应用 Trust 探测该握手。当握手能够支持 MTC 时，Trust 提供 MTC。

## 支持级别

类别遵循部署草案第 5.3.11.3 节。

| 能力 | 级别 | 描述位置 |
| --- | --- | --- |
| TrustAsia 预发 CA：ACME 订单、域名控制验证、签发独立 MTC，以及同一订单上随后的地标相对证书 | 已测试的互操作性，仅限预发环境 | [trust-asia.zh-CN.md](trust-asia.zh-CN.md) |
| CertLocker：选择目录 profile、存储独立工件、等待、存储地标相对工件，并保持两条记录关联 | 已测试的互操作性，仅限预发环境 | [certlocker.zh-CN.md](certlocker.zh-CN.md) |
| 在证书管理器中保留证书字节和签发元数据 | 已按成对库存记录测试。本仓库尚无字节级样本 | [certlocker.zh-CN.md](certlocker.zh-CN.md) |
| 依赖方对包含证明和共签的验证 | 这是证书管理器的存储结果。验证仍由依赖方完成 | [certlocker.zh-CN.md](certlocker.zh-CN.md) |
| 在传统证书、独立 MTC 和地标相对 MTC 之间进行 TLS 握手选择 | 已在 [mtcdemo.trustasia.cn](https://mtcdemo.trustasia.cn/) 上测试 | [trust-asia.zh-CN.md](trust-asia.zh-CN.md) |
| Trust 探测握手，并在握手支持 MTC 时提供 MTC | 已测试的互操作性，预发演示 | [certlocker.zh-CN.md](certlocker.zh-CN.md) |
| 在证书寿命的 75% 时续期，并在临近过期时告警 | Trust 的默认行为。探测将已存储的指纹与正在提供的证书相比较 | [certlocker.zh-CN.md](certlocker.zh-CN.md) |
| 多节点铺开 | CertLocker 已能用一条自动化把一张证书安装到每一台选定的主机 | [certlocker.zh-CN.md](certlocker.zh-CN.md) |
| 吊销、紧急撤回、故障恢复 | 相对于测试 6、7 和 12，有待进一步测试 | 两份报告 |

## 有待进一步测试的内容

Trust 是针对演示主机的实时握手检查：它进行探测，在该握手能够支持 MTC 时提供 MTC，并比较指纹以确认安装。默认情况下，Trust 在证书寿命达到 75% 时续期，并在临近过期时告警。吊销和故障恢复仍属于第 11 节测试 6、7 和 12 的进一步测试。
