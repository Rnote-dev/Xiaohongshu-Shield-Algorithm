# Xiaohongshu Shield Algorithm — 小红书 APP 逆向 Shield 签名算法纯算还原

> 最新测试通过：**小红书安卓 APP v9.19.2** — 2026-03-10 00:41:50

<p align="center">
  <a href="https://rnote.dev/">Rnote API 官网</a> ·
  <a href="https://rnote.dev/docs/guide">API 文档</a> ·
  <a href="https://docs.rnote.dev/">在线调试</a> ·
  <a href="https://t.me/Evil_Geek">Telegram</a>
</p>

---

## 算法概述

小红书 APP 的网络请求依赖一套名为 **Shield** 的签名保护机制，对应请求头中的 `shield` 字段。该签名由客户端 native 层（`libxyass.so`）生成，用于服务端校验请求合法性。

本项目通过逆向分析 `libxyass.so`，将 Shield 算法从 ARM 汇编完整还原为纯 Python 实现，**不依赖任何模拟器或真机环境**。

---

## 签名流程

Shield 签名的生成分为三个阶段，对应代码中的三个核心类：

```
签名输入 (sign_str)
       │
       ▼
┌─────────────┐     device_id 派生密钥
│   XYAes     │◄──  XOR 固定魔数
│  AES 解密   │     TBox 白盒密钥扩展
└──────┬──────┘
       │ aes_result
       ▼
┌─────────────┐     sign_str + aes_result
│   XYMd5     │◄──  混合输入
│  变种 MD5   │     自定义 K 表 + 位运算
└──────┬──────┘
       │ md5_result
       ▼
┌─────────────┐     md5_result
│   XYData    │◄──  RC4 变种流密码
│  数据编码   │     查表置换 + Base64
└──────┬──────┘
       │
       ▼
  Shield 签名 ("XY" 前缀的 Base64 字符串)
```

### 阶段 1：AES 白盒解密（`XYAes`）

- **密钥派生**：将 `device_id` 转为 hex，与 4 个固定魔数（`0xF1892131`、`0xFF001123`、`0xF1001356`、`0xF1234890`）进行 XOR，作为 AES-128 初始密钥
- **白盒密钥扩展**：使用预计算的 T-Box 查找表（`TBox_1` ~ `TBox_8`）替代标准 AES S-Box，实现白盒 AES 密钥扩展，共 10 轮
- **逆序初始化**：密钥扩展完成后，对轮密钥进行逆序交换（`aes_init`），配合逆向 T-Box 变换，为解密模式做准备
- **CBC 解密**：使用扩展后的轮密钥对 HMAC 密钥（或默认 `xor_byte` 向量）执行 AES-128-CBC 解密，输出 96 字节（`0x60`）结果

### 阶段 2：变种 MD5 摘要（`XYMd5`）

- **初始化向量**：使用标准 MD5 IV（`0x67452301`、`0xEFCDAB89`、`0x98BADCFE`、`0x10325476`），但以逆序存放
- **自定义 K 表**：使用小红书专有的 `md5_k` 常量表（64 个 32 位整数），替代标准 MD5 的正弦常量表
- **混合输入**：将签名字符串（URL + 参数 + 设备信息）与 AES 解密结果拼接，作为 MD5 输入
- **分块处理**：按 64 字节分块，每块执行 4 轮共 64 步变换，包含自定义的位移量和逻辑函数（F/G/H/I）
- **多轮变换**：对拼接后的数据执行多次 `md5_update` + `md5_transform`，输出 16 字节摘要

### 阶段 3：RC4 变种流密码 + 编码（`XYData`）

- **初始化表**：基于 MD5 输出，使用 KSA（Key Scheduling Algorithm）初始化 256 字节置换表
- **PRGA 流密码**：使用变种 RC4 的 PRGA（Pseudo-Random Generation Algorithm）生成密钥流，与中间数据 XOR
- **最终编码**：对流密码输出进行字节重排和 Base64 编码，生成以 `"XY"` 为前缀的最终签名字符串

---

## 核心数据结构

### GlobalData — 全局配置与查找表

| 字段 | 说明 |
| --- | --- |
| `build_id` | APP 版本号编码，如 `9192809` 对应 v9.19.2 |
| `device_id` | 设备唯一标识（UUID 格式），用于 AES 密钥派生 |
| `hmac` | HMAC 密钥（Base64），从真机 `shared_prefs/s.xml` 提取 |
| `TBox_1` ~ `TBox_8` | 白盒 AES 查找表，从 `libxyass.so` 的 `.rodata` 段提取 |
| `md5_k` | 自定义 MD5 K 常量表（64 个 uint32） |
| `dword_AAB90` | AES 轮常量（Rcon）|
| `xor_byte` | 默认 XOR 向量，无 HMAC 时的降级输入 |

### ShieldSdk — 异步安全封装

```python
sdk = ShieldSdk(
    build_id="9192809",       # APP 版本号
    device_id="your-uuid",    # 设备 ID
    hmac_key="base64-key",    # HMAC 密钥（可选）
)

signature = await sdk.get_shield(sign_str)
```

- 使用 `asyncio.Lock` 保证并发安全（GlobalData 为全局可变状态）
- 每次调用自动重置 `xor_byte`，防止状态污染

---

## 签名输入格式

`sign_str` 的拼接规则：

> 此拼接规则基于对小红书 APP v9.19.2 的逆向分析，可能随版本更新而变化。请根据实际请求调整输入字符串。

```
URI + QueryString + xy_common_params + xy_direction + xy_platform_info + xy_scene + payload
```

示例：
```
/api/sns/v5/note/comment/list?note_id=69adc8e6000000000e00f4fa&start=&num=15&show_priority_sub_comments=0&source=explore&...
```

---

## HMAC 密钥提取

Shield 算法需要设备的 HMAC 密钥来生成有效签名。该密钥存储在小红书 APP 的 SharedPreferences 中，需要 **Root 权限**从真机提取。

### 提取步骤

1. 确保手机已 Root，并通过 USB 连接电脑
2. 执行以下 ADB 命令提取 HMAC 密钥：

```bash
adb shell "su -c 'cat /data/data/com.xingin.xhs/shared_prefs/s.xml'" 2>&1
```

3. 在输出的 XML 中找到 HMAC 相关的 `<string>` 节点，其值为 Base64 编码的密钥
4. 将提取的值填入 `GlobalData.hmac` 或 `ShieldSdk` 的 `hmac_key` 参数

### 注意事项

- 需要 **Root 权限**才能访问 `/data/data/com.xingin.xhs/` 目录
- HMAC 密钥与设备绑定，不同设备的密钥不同
- 如果没有 HMAC 密钥，算法会使用默认的 `xor_byte` 向量作为降级方案

---

## 快速使用

### 安装依赖

```bash
pip install loguru
```

### 同步调用

```python
from shield_sdk import GlobalData, get_shield

# 配置设备参数
GlobalData.build_id = str(9192809)
GlobalData.device_id = "your-device-uuid"

# HMAC密钥 (从真机 /data/data/com.xingin.xhs/shared_prefs/s.xml 提取)
# 提取命令: adb shell "su -c 'cat /data/data/com.xingin.xhs/shared_prefs/s.xml'" 2>&1
GlobalData.hmac = "your-hmac-base64-key"

# 生成签名
sign_input = "/api/sns/v5/note/comment/list?note_id=xxx&num=15"
signature = get_shield(sign_input)
print(f"Shield 签名: {signature}")
```

### 异步调用

```python
import asyncio
from shield_sdk import ShieldSdk

async def main():
    sdk = ShieldSdk(
        build_id="9192809",
        device_id="your-device-uuid",
        hmac_key="your-hmac-base64-key",  # 从真机 s.xml 提取
    )
    signature = await sdk.get_shield("/api/sns/v5/note/comment/list?note_id=xxx&num=15")
    print(f"Shield 签名: {signature}")

asyncio.run(main())
```

---

## 测试验证

```
$ python shield_sdk.py
2026-03-10 00:41:50.xxx | INFO     | get_shield:1441 - Shield 加密输入: /api/sns/v5/note/comment/list?note_id=69adc8e6000000000e00f4fa&...
2026-03-10 00:41:50.xxx | INFO     | get_shield:1453 - Shield 加密输出: XYxxxxxxxxxxxxxxxx...
```

已通过小红书安卓 APP **v9.19.2**（2026-03-10）服务端验证，签名被正常接受。

---

## 完整 API 服务

如果你需要的不仅仅是 Shield 签名，而是完整的小红书数据采集 + 互动操作 API 服务：

**[Rnote API](https://rnote.dev/)** — 专业的小红书 APP 数据 API 服务平台

- **22+ API 端点**：笔记详情、评论、搜索、用户、商品、话题、点赞、收藏、关注
- **智能账号池**：多账号自动轮换，内置风控检测与冷却机制
- **按需计费**：仅成功请求扣费，失败请求零费用
- **在线调试**：基于 Apifox 的 [在线调试环境](https://docs.rnote.dev/)，输入 API Key 即可测试
- **多语言 SDK**：cURL / Python / JavaScript / TypeScript / Java / Go / PHP 示例
- **经销商体系**：永久折扣 + 邀请返利

无需自行处理签名、账号、风控、设备管理等复杂问题，一个 API Key 即可调用所有接口。

👉 [免费注册](https://rnote.dev/auth/register) ｜ [API 文档](https://rnote.dev/docs/guide) ｜ [定价](https://rnote.dev/pricing)

---

## 联系方式

- **Telegram**：[@Evil_Geek](https://t.me/Evil_Geek)（推荐，快速响应）
- **Email**：Support@rnote.dev

---

## 免责声明

本项目仅供学习和研究目的，旨在探索移动端安全防护机制的实现原理。请遵守相关法律法规，不得将本项目用于任何违法违规活动。使用者需自行承担因使用本项目而产生的一切风险和责任。

---

## 许可证

Apache License 2.0 — 详见 [LICENSE](LICENSE)

---

<!-- SEO Keywords -->
<!--
小红书Shield算法 小红书签名算法 小红书x-mini-mua 小红书APP签名 小红书逆向
小红书纯算 小红书白盒AES 小红书MD5 小红书RC4 小红书libxyass
Xiaohongshu Shield Xiaohongshu signature RedNote Shield algorithm
x-mini-mua x-mini-sig x-mini-gid x-mini-s1 xy-common-params xy-platform-info
小红书爬虫 小红书采集 小红书数据 小红书API 小红书接口
小红书APP逆向 小红书SO逆向 小红书native签名 小红书加密算法
App Shield 算法还原 纯算还原 Python逆向 Android逆向 SO逆向
libxyass.so shield_sdk AES白盒 白盒密码学 white-box AES
小红书反爬 小红书风控 小红书签名校验 小红书请求签名
RedNote API Xiaohongshu API 小红书数据平台 Rnote API
小红书安卓 小红书Android 小红书v9 小红书2026
Xiaohongshu scraper RedNote scraper Little Red Book API
social media reverse engineering mobile app security research
小红书HMAC 小红书shared_prefs 小红书s.xml adb提取密钥
-->
