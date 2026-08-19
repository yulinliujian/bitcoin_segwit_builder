# Bitcoin SegWit Preimage Builder

![BIP143](https://img.shields.io/badge/BIP-143-blue) ![HTML5](https://img.shields.io/badge/HTML-5-orange) ![Pure Frontend](https://img.shields.io/badge/Pure%20Frontend-No%20Backend-green) ![Language](https://img.shields.io/badge/Language-%E4%B8%AD%E6%96%87-lightgrey)

BIP143 · 多 API 自动容错 UTXO 查询 · 离线签名构造工具

一个**纯前端、零后端**的 Bitcoin SegWit 交易构造工具：为 P2WPKH / P2SH-P2WPKH 输入构造 BIP143 签名预镜像（Preimage），配合离线私钥完成 ECDSA 签名后，合并 witness 数据生成可广播的原始交易。全部逻辑在浏览器本地运行，**私钥永不进入页面**。

## 功能特性

- **BIP143 Preimage 构造** — 按 BIP143 规范分解并生成指定输入的签名预镜像，逐字段高亮展示
- **多 API 自动容错 UTXO 查询** — Blockstream / Mempool.space / Blockchair / Blockchain.info / BTC.com / SoChain 六个数据源自动切换，单次查询 10 秒超时保护，任一失败自动降级
- **地址严格校验** — 内置 Base58 / Bech32 解码与 checksum 校验，输出地址支持 P2WPKH (`bc1q`)、P2SH-P2WPKH (`3`)、P2PKH (`1`)
- **实时手续费参考** — 基于 mempool.space 推荐费率（经济/半小时/快速）对比本笔费率，手续费过高自动高亮预警
- **BTC 实时价格** — 10 个交易所/行情 API 竞速取价（30 秒刷新），手续费同时显示 **聪 / BTC / USD**
- **二维码传输** — HASH256 预镜像与交易 hex 悬停即显二维码，方便扫码传输至签名设备
- **交易结构可视化** — 最终交易按字段逐行分解展示（version / marker+flag / inputs / witness / locktime），支持一键复制
- **双格式输出** — 完整 SegWit hex + 不含 Witness 的 hex（locktime 前预留占位，便于手动拼接 witness）
- **安全设计** — CSP 白名单限制外部连接；所有外部 API 数据经 hex/整数/地址格式 sanitize 后才进入页面状态

## 快速开始

直接浏览器打开 `bitcoin_segwit_builder.html` 即可，无需安装、无需服务器。

```text
构造交易                    →   离线签名                 →   合并签名广播
─────────────────────      ───────────────────      ─────────────────────
1. 添加输入地址(bc1q/3)      HASH256 + 私钥              1. 填入每个输入的
2. 查询并勾选 UTXO                                    DER 签名 + 压缩公钥
3. 添加收款地址与金额        secp256k1 ECDSA            2. 构造最终交易 hex
4. 选择签名 Input 序号       （硬件钱包/冷环境）          3. 广播交易
5. 生成 Preimage + HASH256
```

1. 打开页面，在「构造交易」标签页添加输入地址（`bc1q` 或 `3` 开头），点击「查询 UTXO」
2. 勾选要花费的 UTXO，添加收款地址与金额（聪）
3. 选择要签名的 Input 序号，点击 **「生成 Preimage + HASH256」**
4. 用你自己的离线私钥方案（硬件钱包、冷环境等）对 HASH256 做 secp256k1 ECDSA 签名，得到 DER 格式签名
5. 切到「合并签名广播」标签页，填入每个输入的 DER 签名与压缩公钥
6. 点击 **「构造最终交易 hex」**，通过以下任一方式广播：
   - [blockstream.info/tx/push](https://blockstream.info/tx/push)
   - [mempool.space/tx/push](https://mempool.space/tx/push)
   - `bitcoin-cli sendrawtransaction <hex>`

> 若某个输入缺少签名或公钥，页面会用占位符 `xxxx` 代替并在结果中提示补全。

## 支持的地址类型

| 类型 | 前缀 | 作为输入 | 作为输出 |
|---|---|---|---|
| P2WPKH（原生 SegWit v0） | `bc1q` | ✅ | ✅ |
| P2SH-P2WPKH（嵌套 SegWit） | `3` | ✅（需手动填写 scriptcode，如 `1976a914...88ac`） | ✅ |
| P2PKH（Legacy） | `1` | ❌ | ✅ |

不支持：Taproot（`bc1p`）、测试网地址（`tb1` / `m` / `n` / `2`）。

## 技术原理（BIP143）

以 P2WPKH 为例，签名预镜像按以下顺序拼接，再做 double-SHA256：

```text
version(4B) │ hash256(all outpoints) │ hash256(all sequences)
│ outpoint(签名输入) │ scriptcode │ amount(8B, LE) │ 0xffffffff
│ hash256(all outputs) │ locktime(4B) │ sighash(1B)
```

- **scriptcode（P2WPKH）**：`1976a914{20B hash160}88ac`，由地址自动推导
- **scriptcode（P2SH-P2WPKH）**：手动填入 redeemScript
- **sighash**：固定 `01000000`（SIGHASH_ALL），最终交易的 DER 签名追加 `01` 后缀
- 页面将 preimage 逐字段分解展示（hash256(inputs) / hash256(sequences) / hash256(outputs) 等），便于离线核对后签名

## 外部依赖

| 依赖 | 用途 |
|---|---|
| [js-sha256](https://cdnjs.cloudflare.com/ajax/libs/js-sha256/0.9.0/sha256.min.js) | SHA-256 实现 |
| [qrcodejs](https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js) | 二维码生成 |
| 外部 API（UTXO / 价格 / 费率） | 运行期数据查询，域名见页面 CSP 白名单 |

## 安全说明

- 本工具**不接触私钥**。签名必须由你自己的离线方案完成（硬件钱包、冷环境等）
- 页面通过 CSP 仅允许访问白名单内的 API 域名，且禁用了 `frame` / `object` 加载
- 所有外部 API 返回数据在进入页面状态前均经过 hex / 整数 / 地址格式 sanitize，并二次校验收款地址的 SPK
- **签名前务必逐一核对页面展示的收款地址与 SPK**

## 免责声明

本项目仅供学习与参考，不构成任何财务建议。加密货币交易涉及资产风险，请自行验证每一笔交易的数据，并对其正确性、安全性负全部责任。
