# KuGouMusicApi 项目静态分析报告

## 1. 项目概述

**项目名称**: KuGouMusicApi  
**版本**: 1.5.1  
**类型**: Node.js 音乐 API 服务  
**许可证**: MIT  
**核心依赖**: axios, express, crypto-js, node-forge, pako, qrcode  

### 1.1 项目定位

本项目是酷狗音乐的第三方 Node.js 版 API 封装，工作原理为**跨站请求伪造 (CSRF)**——通过伪造请求头、签名参数等方式调用官方 API。项目提供 HTTP 服务和程序化调用两种使用方式。

### 1.2 目录结构

```
/workspace/
├── app.js                  # 程序化 API 入口（编程式调用）
├── index.js                # HTTP 服务启动入口
├── main.js                 # 包主入口（pkg 打包用）
├── server.js               # Express 服务器核心（路由注册、中间件）
├── interface.d.ts          # TypeScript 类型定义
├── package.json            # 项目配置
├── tsconfig.json           # TypeScript 配置
├── module/                 # API 模块目录（160+ 模块）
│   ├── login_cellphone.js  # 手机号登录
│   ├── song_url.js         # 获取音乐URL
│   ├── search.js           # 搜索
│   ├── lyric.js            # 歌词获取
│   ├── register_dev.js     # 设备注册(dfid)
│   └── ...
├── util/                   # 核心工具模块
│   ├── index.js            # 工具统一导出
│   ├── crypto.js           # 加密算法（AES/RSA/MD5/SHA1）
│   ├── helper.js           # 请求签名算法
│   ├── request.js          # HTTP 请求封装（axios）
│   ├── util.js             # 通用工具函数
│   ├── config.json         # 平台配置（appid/clientver等）
│   ├── runtime.js          # 运行时配置（CLI参数/代理）
│   ├── apicache.js         # API 响应缓存中间件
│   ├── memory-cache.js     # 内存缓存实现
│   └── generate_simulate.js # 行为指纹模拟生成
└── public/                 # 静态文件（前端验证页面、WASM等）
```

---

## 2. 入口与启动流程

### 2.1 三种入口方式

| 入口文件 | 用途 | 启动方式 |
|---------|------|---------|
| [app.js](file:///workspace/app.js) | 程序化调用库 | `require('./app')` |
| [index.js](file:///workspace/index.js) | HTTP 服务开发模式 | `npm run dev` (nodemon) |
| [main.js](file:///workspace/main.js) | pkg 打包后的 CLI 入口 | `./app_win` 等二进制 |

### 2.2 完整启动流程

```
app.js / index.js
  ↓
require('./util/runtime').applyCliOverrides()
  ├─ 解析命令行参数 (--proxy, --platform, --port 等)
  └─ 覆盖环境变量
  ↓
require('./server').startService()
  ├─ 加载 .env 环境变量（dotenv）
  ├─ 生成全局 GUID（MD5 哈希的 UUID v4）
  ├─ 生成 serverDev（10位随机大写字符串）
  ├─ consturctServer() 构建 Express 应用
  │   ├─ CORS 跨域中间件
  │   ├─ Cookie 解析中间件
  │   ├─ 平台标识 Cookie 注入中间件（MID/GUID/DEV/MAC/WEBGL）
  │   ├─ 请求体解析（json/urlencoded/raw）
  │   ├─ 静态文件服务（public/、docs/）
  │   ├─ API 响应缓存（2分钟，apicache）
  │   └─ getModulesDefinitions() 扫描 module/ 目录注册路由
  └─ app.listen(port, host) 启动监听
```

---

## 3. 核心模块详解

### 3.1 工具统一导出 [util/index.js](file:///workspace/util/index.js)

**作用**: 作为 `util/` 目录的门面模块，统一导出所有工具函数，并根据 `process.env.platform` 选择标准版或概念版(lite)配置。

**导出分类**:

| 类别 | 导出项 | 数量 |
|-----|-------|-----|
| 配置常量 | `apiver`, `appid`, `clientver`, `wx_appid`, `wx_secret`, `srcappid`, `isLite` | 8 |
| 加密函数 | `cryptoAesEncrypt`, `cryptoAesDecrypt`, `cryptoMd5`, `cryptoRSAEncrypt`, `cryptoSha1`, `rsaEncrypt2`, `playlistAesEncrypt`, `playlistAesDecrypt` | 8 |
| 请求函数 | `createRequest` | 1 |
| 签名函数 | `signKey`, `signParams`, `signParamsKey`, `signCloudKey`, `signatureAndroidParams`, `signatureRegisterParams`, `signatureWebParams` | 7 |
| 工具函数 | `randomString`, `decodeLyrics`, `parseCookieString`, `cookieToJson`, `randomNumber`, `calculateMid` | 6 |

**平台切换逻辑**:
```javascript
const isLite = process.env.platform === 'lite';
const useAppid = isLite ? liteAppid : appid;
const useClientver = isLite ? liteClientver : clientver;
```

### 3.2 配置文件 [util/config.json](file:///workspace/util/config.json)

```json
{
  "wx_appid": "wx79f2c4418704b4f8",        // 微信小程序 AppID（标准版）
  "wx_lite_appid": "wx72b795aca60ad321",   // 微信小程序 AppID（概念版）
  "wx_secret": "4efcab88b700769e376e3f6087b8abc9",     // 微信密钥（标准版）
  "wx_lite_secret": "33e486041e5e25729a4e3d2da7502f9a", // 微信密钥（概念版）
  "srcappid": 2919,       // 来源应用 ID
  "appid": 1005,          // 应用 ID（标准版）
  "apiver": 20,           // API 版本
  "clientver": 20489,     // 客户端版本（标准版）
  "liteAppid": 3116,      // 应用 ID（概念版）
  "liteClientver": 11440  // 客户端版本（概念版）
}
```

---

## 4. 加密与安全模块

### 4.1 核心加密模块 [util/crypto.js](file:///workspace/util/crypto.js)

#### 4.1.1 RSA 公钥

项目内置两个 RSA 1024-bit 公钥（PKCS#8 格式），用于登录等敏感接口的参数加密：

| 用途 | 公钥变量 | 环境 |
|-----|---------|-----|
| 标准版 | `publicRasKey` | `platform !== 'lite'` |
| 概念版 | `publicLiteRasKey` | `platform === 'lite'` |

公钥通过 `rsaKeyCache` (Map) 进行缓存，避免重复解析 PEM。

#### 4.1.2 加密方法一览

| 方法名 | 算法 | 模式/填充 | 用途 |
|-------|------|----------|-----|
| `cryptoMd5(data)` | MD5 | - | 签名生成、哈希计算 |
| `cryptoSha1(data)` | SHA1 | - | 哈希计算 |
| `cryptoAesEncrypt(data, opt)` | AES-256-CBC | PKCS7 | 请求参数加密（登录等） |
| `cryptoAesDecrypt(data, key, iv)` | AES-256-CBC | PKCS7 | 响应数据解密 |
| `cryptoRSAEncrypt(data, publicKey)` | RSA-1024 | 无填充(裸加密) | 加密 AES 密钥等短数据 |
| `rsaEncrypt2(data)` | RSA-1024 | PKCS#1 v1.5 | 设备注册等接口 |
| `playlistAesEncrypt(data)` | AES-128-CBC | PKCS7 | 歌单/设备数据加密 |
| `playlistAesDecrypt(data)` | AES-128-CBC | PKCS7 | 歌单/设备数据解密 |

#### 4.1.3 AES 加密详细流程

**cryptoAesEncrypt (通用 AES)**:

```
输入: data, { key?, iv? }
  │
  ├─ 若无 key/iv: 生成 16 位随机 key → MD5 → 取前 32 字符为密钥
  │                                  → 取后 16 字符为 IV
  ├─ 若有 key 无 iv:  key MD5 → 前 32 字符为密钥，后 16 字符为 IV
  └─ 若两者都有: 直接使用
  ↓
AES-256-CBC 加密（PKCS7 填充）
  ↓
输出: { str: hex密文, key: 原始密钥 } 或 hex 字符串（当传入 key+iv 时）
```

**playlistAesEncrypt (歌单 AES)**:

```
输入: data
  ↓
生成 6 位随机 key (小写字母+数字)
  ↓
key MD5 → 前 16 字符 = 加密密钥, 后 16 字符 = IV
  ↓
AES-128-CBC 加密（PKCS7 填充）
  ↓
输出: { key: 6位随机串, str: Base64密文 }
```

#### 4.1.4 RSA 加密详细流程

**cryptoRSAEncrypt (裸 RSA)**:

```
输入: data, publicKey?
  ↓
data → Uint8Array (normalizeBuffer)
  ↓
若长度 < keyLength: 左侧补零（高位补零）
  ↓
裸 RSA 加密（modPow）: 消息^e mod n
  ↓
输出: hex 字符串（大写，在 login_cellphone 中 toUpperCase()）
```

**rsaEncrypt2 (PKCS#1 v1.5)**:

```
使用 node-forge 的 RSAES-PKCS1-V1_5 填充
  ↓
输出: hex 字符串
```

### 4.2 签名模块 [util/helper.js](file:///workspace/util/helper.js)

所有签名算法均基于 **MD5 哈希 + 盐值拼接**。

#### 4.2.1 签名方法一览

| 方法名 | 盐值 | 适用接口 | 签名公式 |
|-------|------|---------|---------|
| `signatureWebParams` | `NVPh5oo715z5DIWAeQlhMDsWXXQV4hwt` | Web 版 API | `MD5(salt + 排序后的key=value串 + salt)` |
| `signatureAndroidParams` | 标准版: `OIlwieks28dk2k092lksi2UIkp`<br>概念版: `LnT6xpN3khm36zse0QzvmgTZ3waWdRSA` | Android 版 API（最常用） | `MD5(salt + 排序后的key=value串 + data + salt)` |
| `signatureRegisterParams` | `1014` | 设备注册 | `MD5("1014" + 排序后的所有值 + "1014")` |
| `signParams` | `R6snCXJgbCaj9WFRJKefTMIFp0ey6Gza` | 通用 sign | `MD5(排序后的key+value串 + data + salt)` |
| `signKey` | 标准版: `57ae12eb6890223e355ccfcb74edf70d`<br>概念版: `185672dd44712f60bb1736df5a377e82` | 音乐URL等接口 | `MD5(hash + salt + appid + mid + userid)` |
| `signCloudKey` | `ebd1ac3134c880bda6a2194537843caa0162e2e7` | 云盘接口 | `MD5("musicclound" + hash + pid + salt)` |
| `signParamsKey` | 同 signatureAndroidParams | sign 参数 | `MD5(appid + salt + clientver + data)` |

#### 4.2.2 签名算法共性

1. **参数排序**: 所有签名都会对参数 key 按字母顺序排序
2. **盐值两端**: 盐值通常拼接在明文的首尾两端
3. **MD5 输出**: 全部为 32 位小写 hex 字符串

---

## 5. HTTP 请求模块

### 5.1 请求封装 [util/request.js](file:///workspace/util/request.js)

#### 5.1.1 核心函数: `createRequest(options)`

**方法签名**:
```javascript
createRequest(options: {
  method: 'get' | 'post' | 'GET' | 'POST',
  url: string,
  baseURL?: string,           // 默认 "https://gateway.kugou.com"
  params?: Record<string, any>,
  data?: any,
  headers?: Record<string, any>,
  encryptType: 'android' | 'web' | 'register',
  cookie: Record<string, string>,
  encryptKey?: boolean,
  clearDefaultParams?: boolean,
  notSignature?: boolean,
  ip?: string,
  realIP?: string,
  responseType?: string,
}) => Promise<{
  status: number,    // 200 成功, 502 失败
  body: any,
  cookie: string[],
  headers: Record<string, string>,
}>
```

#### 5.1.2 请求处理流程

```
1. 从 Cookie 提取设备标识
   ├─ dfid: 设备指纹 ID（register_dev 返回）
   ├─ mid: 设备 MID（server.js 通过 calculateMid 生成）
   ├─ uuid: 固定为 '-'
   ├─ token: 用户登录令牌
   └─ userid: 用户 ID

2. 构建默认请求参数
   ├─ dfid, mid, uuid, appid, clientver, clienttime
   └─ 有 token/userid 时追加

3. 生成签名（根据 encryptType）
   ├─ android: signatureAndroidParams (默认)
   ├─ web: signatureWebParams
   └─ register: signatureRegisterParams

4. 配置请求头
   ├─ User-Agent: "Android15-1070-11083-46-0-DiscoveryDRADProtocol-wifi"
   ├─ kg-rc, kg-thash, kg-rec, kg-rf（内部标识头）
   ├─ dfid, clienttime, mid
   └─ X-Real-IP, X-Forwarded-For（IP 透传）

5. 发送请求（axios）
   └─ 支持 HTTP 代理（KUGOU_API_PROXY 环境变量）

6. 处理响应
   ├─ 解析 Set-Cookie → 格式化
   ├─ 检测 ssa-code 响应头（二次验证标识）
   ├─ 响应体 JSON 解析
   └─ 若有 ssa-code，自动生成 sid/edt 附加到 body
```

#### 5.1.3 SSA 二次验证机制

当响应头包含 `ssa-code` 或 `SSA-CODE` 时，表示需要进行二次安全验证（如滑块验证码、行为验证）。此时系统会自动调用 `generateSimulate()` 生成模拟的行为指纹：

- **edt**: AES 加密的行为数据（Base64）
- **sid**: RSA-OAEP 加密的 AES 密钥（Base64）

这些数据会被附加到响应体中，客户端可用于后续验证请求。

---

## 6. 服务器与路由

### 6.1 服务器核心 [server.js](file:///workspace/server.js)

#### 6.1.1 模块扫描与路由注册

**函数**: `getModulesDefinitions(modulesPath, specificRoute, doRequire)`

扫描规则:
- 仅扫描 `.js` 结尾的文件
- 跳过以 `_` 开头的文件（内部模块）
- 文件列表**倒序排列**（与 `app.js` 保持一致）
- 路由生成: 文件名去 `.js` 后缀，`_` 替换为 `/`，加 `/` 前缀
  - 例: `user_detail.js` → `/user/detail`
  - 例: `login_cellphone.js` → `/login/cellphone`

#### 6.1.2 Express 中间件栈（从上到下）

| 顺序 | 中间件 | 作用 |
|-----|-------|-----|
| 1 | CORS 跨域 | 设置 Access-Control-* 头，OPTIONS 返回 204 |
| 2 | Cookie 解析 | 手动解析 Cookie 字符串为对象，挂载到 `req.cookies` |
| 3 | 平台标识注入 | 自动注入 KUGOU_API_MID/GUID/DEV/MAC/WEBGL/PLATFORM 等 Cookie |
| 4 | 请求体解析 | json (5mb) / urlencoded (5mb) / raw (10mb) |
| 5 | 静态文件服务 | public/ 目录直接访问，docs/ 挂载到 /docs |
| 6 | API 缓存 | 2 分钟响应缓存（apicache），仅缓存 200 响应 |
| 7 | 动态路由 | module/ 目录下的所有 API 模块 |

#### 6.1.3 平台标识 Cookie 注入

每个请求都会自动注入以下 Cookie（客户端未提供时）:

| Cookie Key | 生成方式 | 用途 |
|-----------|---------|-----|
| `KUGOU_API_PLATFORM` | `process.env.platform` | 平台类型标识 |
| `KUGOU_API_MID` | `calculateMid(GUID)` | 设备 MID（MD5→十进制大整数） |
| `KUGOU_API_GUID` | 环境变量 或 启动时生成的 UUID v4 MD5 | 设备全局唯一标识 |
| `KUGOU_API_DEV` | 环境变量 或 10位随机大写串 | 开发设备标识 |
| `KUGOU_API_MAC` | 环境变量 或 `02:00:00:00:00:00` | 设备 MAC 地址 |
| `KUGOU_API_WEBGL` | 环境变量 或 `generateWebGLHash()` | WebGL 指纹哈希 |

HTTPS 环境下自动添加 `SameSite=None; Secure` 属性。

#### 6.1.4 路由处理器工作流程

```
请求到达
  ↓
1. 解析 query/body 中的 cookie 字符串 → JSON
  ↓
2. 合并参数: { cookie: { req.cookies + query.cookie }, ...query, ...body }
  ↓
3. 解析 Authorization 头 → 合并到 cookie
  ↓
4. 调用模块函数 module(query, requestFactory)
   └─ requestFactory: 注入客户端 IP 后调用 createRequest
  ↓
5. 处理响应 Cookie → 写入 Set-Cookie 头
  ↓
6. 返回响应: res.header(headers).status(status).send(body)
  ↓
异常: 返回错误响应（status + body）
```

---

## 7. 行为指纹模拟模块

### 7.1 模块概述 [util/generate_simulate.js](file:///workspace/util/generate_simulate.js)

**核心功能**: 在服务端生成模拟的用户行为指纹数据，用于绕过酷狗的行为检测机制。

**加密方案**: 混合加密（AES + RSA）
- 明文（行为数据）→ **AES-128-CBC** → **EDT**（Base64）
- AES 密钥 → **RSA-OAEP SHA-256** → **SID**（Base64）

### 7.2 固定密钥

| 密钥 | 值 | 用途 |
|-----|-----|-----|
| RSA 公钥 | 2048-bit SPKI 格式（内置） | 加密 AES 密钥 |
| AES IV | `kugousecurity123` (16字节) | AES CBC 初始化向量 |
| 哨兵值 | `0xFFFFFFFF - 随机(0-20)` | 事件序列结束标记 |

### 7.3 行为数据生成

模拟的用户行为事件类型:

| 类型码 | 事件类型 | 格式 |
|-------|---------|-----|
| 3 | 鼠标/触摸移动 | `3,时间差,子索引,X,Y` |
| 5 | 滚动/计时事件 | `5,时间差,事件索引` |
| 6 | 窗口事件（resize/load） | `6,时间差,事件索引,宽,高` |

**鼠标轨迹生成**: 使用**三阶贝塞尔曲线** + 随机抖动，模拟真人鼠标移动特性:
- 起点抖动大（3px），终点抖动小（0.5px）
- 起步慢、中间快、结束减速
- 每隔 12 帧插入滚动事件

### 7.4 完整生成流程

```
generateSimulate(mid, userid, dfid, webglHash)
  │
  ├─ 生成随机 AES-128 密钥: MD5(randomString(16)).substring(0, 16)
  │
  ├─ 随机化鼠标轨迹参数
  │   ├─ 起点: X(200-600), Y(200-500)
  │   ├─ 终点: X(500-700), Y(80-150)
  │   └─ 采样点: 30~60 个
  │
  ├─ generateEDTData() 生成行为数据
  │   ├─ 两个初始 type-5 零事件（各带哨兵）
  │   ├─ 1 个窗口事件 (750x500)
  │   ├─ 3 个滚动事件
  │   ├─ 贝塞尔曲线鼠标轨迹（每12帧插入滚动事件）
  │   └─ 结束微调位置
  │
  ├─ 拼接明文:
  │   `mid=...;userid=...;dfid=...;webgl=...;webdriver=0;ts=...;data=...`
  │
  ├─ AES-128-CBC 加密 → EDT（Base64）
  │
  └─ RSA-OAEP SHA-256 加密 AES 密钥 → SID（Base64）
```

---

## 8. 缓存模块

### 8.1 API 缓存 [util/apicache.js](file:///workspace/util/apicache.js)

基于 NeteaseCloudMusicApi 的 apicache 改造，提供 Express 中间件级别的响应缓存。

**主要特性**:
- 支持内存缓存和 Redis 缓存
- 按时间字符串设置过期（如 "2 minutes"）
- 支持按分组清除
- ETag/304 协商缓存支持
- 缓存命中率统计（位压缩存储，支持 100/1000/10000/100000 时间窗口）
- 支持 JSONP 模式 URL 去参

**项目中的使用**:
```javascript
app.use(cache('2 minutes', (_, res) => res.statusCode === 200));
```
即所有成功响应（状态码 200）缓存 2 分钟。

**绕过缓存方式**: 附加不同的 `timestamp` 查询参数。

### 8.2 内存缓存 [util/memory-cache.js](file:///workspace/util/memory-cache.js)

简单的键值对内存缓存实现，作为 apicache 的默认存储后端。

**API**:
- `add(key, value, time, timeoutCallback)` - 添加条目并设置自动过期
- `get(key)` - 获取完整条目（含元数据）
- `getValue(key)` - 仅获取值
- `delete(key)` - 删除条目
- `clear()` - 清除所有

---

## 9. 通用工具函数 [util/util.js](file:///workspace/util/util.js)

| 函数 | 签名 | 用途 |
|-----|------|-----|
| `randomString(len=16)` | `(number) => string` | 生成随机字符串（大写字母+数字，36字符池） |
| `randomNumber(len=16)` | `(number) => string` | 生成随机数字字符串 |
| `parseCookieString(cookie)` | `(string) => string` | 移除 Cookie 中的 Domain/path/expires/HttpOnly 字段 |
| `cookieToJson(cookie)` | `(string) => Object` | Cookie 字符串转 JSON 对象 |
| `decodeLyrics(val)` | `(string|Uint8Array|Buffer) => string` | KRC 歌词解码（XOR + zlib） |
| `calculateMid(str)` | `(string) => string` | 计算设备 MID（MD5 → 十进制大整数） |
| `getGuid()` | `() => string` | 生成 UUID v4 格式 GUID |
| `generateWebGLHash()` | `() => string` | 生成 WebGL 指纹哈希（FNV-1a 64-bit） |

### 9.1 KRC 歌词解码算法

```
加密歌词 (Base64 / Uint8Array / Buffer)
  ↓
跳过前 4 字节（文件头标识）
  ↓
XOR 异或解密（16字节密钥循环使用）
  密钥: [64, 71, 97, 119, 94, 50, 116, 71, 81, 54, 49, 45, 206, 210, 110, 105]
  ↓
pako.inflate (zlib 解压)
  ↓
明文字符串 (UTF-8)
```

### 9.2 MID 计算算法

```
输入字符串 → MD5 哈希 (32位 hex)
  ↓
将 hex 字符串视为 16 进制大整数
  ↓
逐位计算: sum += digit * 16^(position)
  ↓
输出: 十进制字符串（使用 big-integer 库避免精度丢失）
```

---

## 10. 运行时配置 [util/runtime.js](file:///workspace/util/runtime.js)

### 10.1 CLI 参数支持

| 参数 | 对应环境变量 | 说明 |
|-----|------------|-----|
| `--proxy=...` | `KUGOU_API_PROXY` | HTTP 代理地址 |
| `--platform=...` | `platform` | 平台类型（lite 等） |
| `--guid=...` | `KUGOU_API_GUID` | 设备 GUID |
| `--dev=...` | `KUGOU_API_DEV` | 开发设备标识 |
| `--mac=...` | `KUGOU_API_MAC` | 设备 MAC 地址 |
| `--port=...` | `PORT` | 服务器端口 |

### 10.2 代理配置解析

支持 `KUGOU_API_PROXY` 环境变量，格式: `http(s)://[user:pass@]host:port`

- 使用 Node.js 内置 `URL` 解析
- 支持 HTTP/HTTPS 代理协议
- 支持带认证信息的代理
- 结果缓存，避免重复解析

---

## 11. 典型业务流程分析

### 11.1 手机号登录流程 [module/login_cellphone.js](file:///workspace/module/login_cellphone.js)

**接口路径**: `/login/cellphone`  
**目标地址**: `https://loginserviceretry.kugou.com/v7/login_by_verifycode`

**完整流程**:

```
1. 生成 AES 密钥对（随机），加密手机号和验证码
   encrypt = cryptoAesEncrypt({ mobile, code })
   → { str: hex密文, key: 16位随机串 }

2. 生成 t1/t2 令牌（仅概念版 lite）
   ├─ t2 = AES 加密 (GUID|salt|MAC|DEV|时间戳, liteT2Key, liteT2Iv)
   └─ t1 = AES 加密 (|时间戳, liteT1Key, liteT1Iv)
   概念版固定密钥:
     - liteT2Key: fd14b35e3f81af3817a20ae7adae7020
     - liteT2Iv:  17a20ae7adae7020
     - liteT1Key: 5e4ef500e9597fe004bd09a46d8add98
     - liteT1Iv:  04bd09a46d8add98

3. 组装请求参数
   ├─ plat: 1
   ├─ support_multi: 1
   ├─ t1, t2（仅 lite）
   ├─ clienttime_ms: 当前时间戳
   ├─ mobile: 脱敏手机号（前2位+*****+第11位）
   ├─ key: signParamsKey(clienttime_ms)  // MD5(appid+salt+clientver+time)
   ├─ pk: RSA加密({clienttime_ms, key: encrypt.key}).toUpperCase()
   └─ params: encrypt.str  // AES加密的手机号+验证码

4. POST 请求 → loginserviceretry.kugou.com

5. 处理响应
   └─ 若 status === 1 且有 secu_params:
      → AES 解密 secu_params（使用 encrypt.key）
      → 提取 token/userid/vip_token 等
      → 写入响应 Cookie: t1, token, userid, vip_type, vip_token
```

### 11.2 设备注册 (dfid) 流程 [module/register_dev.js](file:///workspace/module/register_dev.js)

**接口路径**: `/register/dev`  
**目标地址**: `https://userservice.kugou.com/risk/v2/r_register_dev`

**完整流程**:

```
1. 构建设备信息数据（约 30+ 项）
   ├─ 内存/存储: availableRamSize, availableRomSize, availableSDSize
   ├─ 电池: batteryLevel, batteryStatus
   ├─ 设备标识: brand, device, manufacturer, imei, imsi, uuid
   └─ 传感器: accelerometer, gravity, gyroscope, light, magnetic, ...

2. playlistAesEncrypt 加密设备数据
   → { key: 6位随机串, str: Base64密文 }

3. RSA 加密 AES 密钥 + 用户信息
   p = rsaEncrypt2({ aes: key, uid: userid, token })

4. POST 请求
   ├─ URL: /risk/v2/r_register_dev
   ├─ params: { part: 1, platid: 1, p }
   ├─ data: AES 加密的设备信息（Base64）
   └─ responseType: arraybuffer

5. 响应处理
   ├─ playlistAesDecrypt 解密响应体（用相同的 AES 密钥）
   └─ 若成功，提取 dfid → 写入 Cookie
```

### 11.3 获取音乐 URL 流程 [module/song_url.js](file:///workspace/module/song_url.js)

**接口路径**: `/song/url`  
**目标地址**: `https://gateway.kugou.com/v5/url`  
**路由头**: `x-router: trackercdn.kugou.com`

**关键参数**:

| 参数 | 说明 |
|-----|-----|
| `hash` | 音乐 hash（小写） |
| `album_id` | 专辑 ID |
| `album_audio_id` | 专辑音频 ID (MixSongID) |
| `quality` | 音质: 128/320/flac/high/viper_atmos 等 |
| `free_part` | 是否返回试听部分 |
| `page_id` | 页面 ID（标准版/概念版不同） |

**签名**: 使用 `encryptKey: true` 触发 `signKey` 签名（MD5(hash + salt + appid + mid + userid)）

### 11.4 歌词获取流程 [module/lyric.js](file:///workspace/module/lyric.js)

**接口路径**: `/lyric`  
**目标地址**: `https://lyrics.kugou.com/download`

**参数**:
- `id`: 歌词 ID（从 /search/lyric 获取）
- `accesskey`: 歌词访问密钥（从 /search/lyric 获取）
- `fmt`: `krc`（逐字）或 `lrc`（普通）
- `decode`: 传入则返回解码后的歌词

**解码逻辑**:
- 若 `fmt=lrc` 或 `contenttype!==0`: Base64 → UTF-8 字符串
- 若 `fmt=krc` 且 `contenttype===0`: Base64 → decodeLyrics (XOR + zlib)

---

## 12. 程序化调用入口 [app.js](file:///workspace/app.js)

### 12.1 动态加载机制

```javascript
fs.readdirSync(path.join(__dirname, 'module'))
  .reverse()      // 倒序排列
  .forEach(file => {
    // 加载模块
    let fileModule = require(path.join(__dirname, 'module', file));
    // 提取函数名（文件名去 .js 后缀）
    let fn = file.split('.').shift();
    // 创建包装函数
    obj[fn] = (data = {}) => {
      // cookie 字符串自动转对象
      if (typeof data.cookie === 'string') 
        data.cookie = cookieToJson(data.cookie);
      // 调用模块函数，注入请求工厂（延迟加载避免循环依赖）
      return fileModule(
        { ...data, cookie: data.cookie || {} },
        (...args) => require('./util/request').createRequest(...args)
      );
    };
  });
```

### 12.2 最终导出结构

```javascript
module.exports = {
  ...require('./server'),    // startService, getModulesDefinitions
  ...require('./util/request'), // createRequest
  ...obj                     // 160+ API 函数
};
```

**使用示例**:
```javascript
const api = require('./app');

// 登录
const loginRes = await api.login_cellphone({
  mobile: '13800138000',
  code: '123456'
});

// 搜索（携带 Cookie）
const searchRes = await api.search({
  keywords: '海阔天空',
  cookie: `token=${loginRes.body.token};userid=${loginRes.body.userid}`
});
```

---

## 13. 模块签名规范

所有 API 模块（module/ 目录下）遵循统一的函数签名规范:

```javascript
/**
 * @param {Object} params - 请求参数
 * @param {Object} params.cookie - Cookie 键值对
 * @param {Function} useAxios - 请求工厂函数
 * @returns {Promise<{
 *   status: number,
 *   body: any,
 *   cookie: string[],
 *   headers?: Object
 * }>}
 */
module.exports = (params, useAxios) => {
  // 构造 dataMap 参数
  const dataMap = { ... };
  
  // 发起请求
  return useAxios({
    url: '/vX/...',
    method: 'GET' | 'POST',
    params: dataMap,      // GET 参数
    data: dataMap,        // POST 数据
    encryptType: 'android' | 'web' | 'register',
    headers: { ... },
    cookie: params?.cookie || {},
    // 其他选项: baseURL, encryptKey, responseType 等
  });
};
```

---

## 14. 安全要点与注意事项

### 14.1 安全机制汇总

| 机制 | 实现位置 | 用途 |
|-----|---------|-----|
| RSA 公钥加密 | crypto.js | 保护登录密钥等敏感短数据 |
| AES-CBC 加密 | crypto.js | 保护请求参数和响应数据 |
| MD5 请求签名 | helper.js | 防止请求参数篡改 |
| 多平台密钥隔离 | config.json + crypto.js | 标准版/概念版使用不同密钥 |
| 设备指纹 (dfid) | register_dev.js | 设备标识与风控 |
| 行为指纹 (sid/edt) | generate_simulate.js | 绕过二次验证/行为检测 |
| 多标识组合 | server.js (Cookie注入) | GUID/MID/DEV/MAC/WEBGL 多维度标识 |

### 14.2 重构安全性保障要点

进行重构时，以下内容必须保持完全一致，否则将导致 API 不可用:

1. **签名算法**（helper.js 中所有函数）: 盐值、排序方式、拼接顺序、哈希算法
2. **加密算法参数**: AES 模式(CBC)、填充(PKCS7)、密钥派生方式(MD5)
3. **RSA 公钥**: PEM 内容、加密方式（裸加密/PKCS#1 v1.5）
4. **KRC 歌词解密**: XOR 密钥、文件头偏移量、压缩算法(zlib)
5. **行为指纹格式**: 事件编码格式、哨兵值机制、明文拼接格式
6. **默认请求头**: User-Agent、kg-* 系列内部头
7. **默认参数**: appid、clientver、dfid、mid、uuid 的组合
8. **设备注册数据结构**: 所有字段名及默认值

### 14.3 敏感信息

`util/config.json` 中包含微信小程序的 AppID 和 Secret，属于敏感凭据，在公开部署时需注意保护。

---

## 15. 依赖关系图

```
app.js (程序化入口)
├── server.js
│   ├── util/util.js
│   ├── util/crypto.js
│   ├── util/request.js
│   └── util/apicache.js
│       └── util/memory-cache.js
│
├── util/request.js
│   ├── util/helper.js
│   │   └── util/crypto.js
│   ├── util/util.js
│   ├── util/config.json
│   ├── util/runtime.js
│   └── util/generate_simulate.js
│       ├── util/util.js
│       └── (crypto-js, node-forge)
│
└── module/*.js (160+ 模块)
    └── util/index.js (统一入口)
        ├── util/crypto.js
        ├── util/helper.js
        ├── util/request.js
        ├── util/util.js
        ├── util/config.json
        └── util/runtime.js
```

---

## 16. TypeScript 类型定义

[interface.d.ts](file:///workspace/interface.d.ts) 提供了完整的类型定义，包括:

- **基础类型**: `CommonParams`, `ApiResponse<T>`, `PaginatedParams`, `CookieMap`
- **枚举类型**: `SongQuality` (15种音质), `SearchType`, `CommentSort`, `FmMode` 等
- **请求参数类型**: 按功能分类的 60+ 参数接口（登录、用户、歌单、专辑、音乐、搜索...）
- **导出函数声明**: 每个 API 模块对应的函数签名

---

## 17. 部署与运行环境

### 17.1 环境变量支持

| 变量名 | 默认值 | 说明 |
|-------|-------|-----|
| `PORT` | 3000 | 服务端口 |
| `HOST` | '' (所有接口) | 监听地址 |
| `platform` | - | 平台类型（lite=概念版） |
| `KUGOU_API_PROXY` | - | HTTP 代理地址 |
| `KUGOU_API_GUID` | 自动生成 | 设备 GUID |
| `KUGOU_API_DEV` | 自动生成 | 开发设备标识 |
| `KUGOU_API_MAC` | `02:00:00:00:00:00` | MAC 地址 |
| `KUGOU_API_WEBGL` | 自动生成 | WebGL 指纹 |
| `CORS_ALLOW_ORIGIN` | origin 或 * | 跨域允许来源 |

### 17.2 部署方式

- **本地开发**: `npm run dev` (nodemon 热重载)
- **生产运行**: `npm start` 或 `node app.js`
- **Vercel 部署**: [vercel.json](file:///workspace/vercel.json)
- **Docker 部署**: [Dockerfile](file:///workspace/Dockerfile)
- **二进制打包**: `pkg` (支持 Windows/Linux/macOS/arm64)

---

## 18. 总结

本项目是一个结构清晰的酷狗音乐 API 反向工程实现，核心技术栈为:

1. **Express** 提供 HTTP 服务
2. **Axios** 作为 HTTP 客户端转发请求
3. **crypto-js + node-forge** 实现加密解密（AES/RSA/MD5/SHA1）
4. **动态模块加载** 实现路由自动注册
5. **多级缓存** 提升响应速度（apicache 2分钟缓存）
6. **行为指纹模拟** 绕过风控检测

项目模块划分清晰，`util/` 目录封装核心能力，`module/` 目录按功能拆分 API，每个模块职责单一。
