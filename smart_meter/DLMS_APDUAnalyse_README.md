# DLMS_APDUAnalyse.c 模块架构文档

## 文件概述

| 项目 | 说明 |
|------|------|
| 文件名 | DLMS_APDUAnalyse.c |
| 功能 | DLMS/COSEM协议APDU分析和加密处理核心模块 |
| 代码行数 | 约3130行 |
| 主要职责 | APDU解析/封装、加密解密、数字签名、块传输处理、COSEM对象操作 |
| 条件编译 | `COMM_SUIT >= CS_SUIT1` 控制高级安全套件功能 |

## 整体架构图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        DLMS_APDUAnalyse.c 模块架构                       │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                         外部接口层                                 │  │
│  │  DataBuf(数据缓冲区) / DataLn(数据长度) / CommType(通信端口)      │  │
│  └───────────────────────────────┬────────────────────────────────────┘  │
│                                  │                                       │
│  ┌───────────────────────────────▼────────────────────────────────────┐  │
│  │                      APDU主控层 (收发入口)                         │  │
│  │                                                                    │  │
│  │   接收方向: FUC_DlmsApduAnalyse() ──→ 安全解包 ──→ COSEM处理      │  │
│  │   发送方向: COSEM处理 ──→ 数据组包 ──→ FUC_DlmsApduWrap()         │  │
│  └───────┬──────────────────────────────────────────┬─────────────────┘  │
│          │                                          │                    │
│  ┌───────▼───────────┐  ┌──────────────────────┐  ┌▼─────────────────┐  │
│  │   安全处理层       │  │   COSEM对象处理层    │  │   块传输层       │  │
│  │                    │  │                      │  │                  │  │
│  │ FUC_AlogGMACUnwrap│  │ FUC_GetCosemInfo     │  │ FUC_GBT_         │  │
│  │   (加密解包)       │  │   (对象信息解析)     │  │   Disassembly    │  │
│  │                    │  │                      │  │   (GBT解包)      │  │
│  │ FUC_AlogGMACWrap  │  │ FUC_CosemProcess     │  │                  │  │
│  │   (加密组包)       │  │   (对象操作主控)     │  │ FUC_GBT_         │  │
│  │                    │  │                      │  │   Reassembly     │  │
│  │ FUC_AlogDigital   │  │ FV_Comm_Operate      │  │   (GBT组包)      │  │
│  │ SignVerify         │  │ Process              │  │                  │  │
│  │   (签名验证)       │  │   (操作分发)         │  └──────────────────┘  │
│  │                    │  │                      │                        │
│  │ FUC_AlogDisital   │  │ FUC_WriteDataFram    │  ┌──────────────────┐  │
│  │ Sign               │  │   (写入数据帧)       │  │   参数管理层     │  │
│  │   (签名生成)       │  │                      │  │                  │  │
│  │                    │  │ FUC_CompositeData    │  │ FV_LoadServer    │  │
│  └────────────────────┘  │ Fram                 │  │ Para(加载参数)   │  │
│                          │   (复合帧构建)       │  │                  │  │
│                          │                      │  │ FV_LoadDedicated │  │
│                          │ FV_ErrorJudge        │  │ Key(加载专用密钥)│  │
│                          │   (错误判断)         │  │                  │  │
│                          │                      │  │ FV_CosemInfoInit │  │
│                          │ DlmsAdjustLen        │  │   (COSEM初始化)  │  │
│                          │   (长度编码工具)     │  └──────────────────┘  │
│                          └──────────────────────┘                        │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

## 函数调用关系图

```
接收数据流:
  外部 → FUC_DlmsApduAnalyse()
              ├── FUC_AlogDigitalSignVerify()    [数字签名验证]
              ├── FUC_AlogGMACUnwrap()           [加密解包]
              │       ├── 密钥类型判断(unicast/broadcast/dedicated/wrapped/agreed)
              │       ├── 安全策略验证
              │       ├── 调用计数器验证(防重放攻击)
              │       └── Decrypt_ByteData()      [AES-GCM解密]
              └── 安全策略合规性检查
                    ↓
         FUC_GetCosemInfo()                     [解析COSEM描述符]
                    ↓
         FUC_CosemProcess()                     [COSEM操作主循环]
              ├── FV_Comm_OperateProcess()       [按ClassID分发操作]
              │       ├── Comm_ClassID1()  ~ Comm_ClassID122()  [各COSEM类处理]
              │       └── 错误结果记录
              ├── FUC_WriteDataFram()            [写入数据帧缓存]
              └── FUC_CompositeDataFram()        [构建响应APDU]
                      ├── 读取响应(普通/块/列表)
                      ├── 写入响应(普通/块/列表)
                      ├── 执行响应
                      ├── 访问响应
                      └── FV_ErrorJudge()        [错误码映射]

发送数据流:
  响应数据 → FUC_DlmsApduWrap()
                ├── FUC_AlogGMACWrap()           [加密组包]
                │       ├── 密钥协商(ECDH/KDF)   [agreed-key模式]
                │       ├── 密钥包装(AES-WRAP)   [wrapped-key模式]
                │       ├── Encrypt_ByteData()    [AES-GCM加密]
                │       └── 构建安全头部
                └── FUC_AlogDisitalSign()         [数字签名生成]
                        └── FUC_EcdsaSignature()  [ECDSA签名]
```

## 全部函数详细说明

### 一、通用块传输(GBT)处理函数 (2个)

#### 1. FUC_GBT_Disassembly (L35-L98)
- **原型**: `unsigned char FUC_GBT_Disassembly(unsigned char* DataBuf, ST_GBT_DATA* st_GBTData, unsigned short* DataLn)`
- **功能**: 通用块传输解包 — 从GBT帧中提取有效数据
- **参数**:
  - `DataBuf`: 数据缓冲区(输入/输出)
  - `st_GBTData`: GBT状态结构指针
  - `DataLn`: 输出数据长度
- **返回值**: `0xFF`=成功, `0x00`=不是GBT帧, `0x01`=块号错误
- **核心逻辑**:
  1. 检查首字节是否为`D_general_block_transfer`
  2. 提取最后块标志(`D_GBT_LB_FLG`)
  3. 提取并验证块号连续性(`usServeBlockNumAck+1`)
  4. 提取并验证确认块号
  5. 解析数据长度(支持BER长格式: 0x80+N编码)
  6. `memmove`将有效数据移到缓冲区开头

#### 2. FUC_GBT_Reassembly (L107-L173)
- **原型**: `unsigned char FUC_GBT_Reassembly(unsigned char* DataBuf, ST_GBT_DATA* st_GBTData, unsigned short* usDataLn)`
- **功能**: 通用块传输组包 — 将数据封装为GBT帧
- **参数**:
  - `DataBuf`: 数据缓冲区(输入/输出)
  - `st_GBTData`: GBT状态结构指针
  - `usDataLn`: 数据长度(输入/输出，含帧头后更新)
- **返回值**: `0x00`=成功
- **核心逻辑**:
  1. 根据数据长度确定编码格式(≤0x80短格式, ≤0xFF需1字节长格式, >0xFF需2字节长格式)
  2. `memmove`将数据后移为帧头预留空间(7+ucLn字节)
  3. 写入`D_general_block_transfer`标签
  4. 区分服务器端/客户端写入不同的块号和确认号
  5. 写入长度字段(短格式直接写, 长格式写0x80+N+值)

---

### 二、加密和认证处理函数 (4个)

#### 3. FUC_AlogDigitalSignVerify (L255-L330)
- **原型**: `unsigned char FUC_AlogDigitalSignVerify(unsigned char* DataBuf, ST_DLMS_DIGITAL_PARA* st_Dlms_Digital_Para, unsigned char* PubKey, unsigned short* DataLn, unsigned char Force)`
- **功能**: 数字签名验证 — 使用ECDSA算法验证数据完整性和发送方身份
- **参数**:
  - `DataBuf`: 签名数据缓冲区(输入/输出，验证后移除签名头)
  - `st_Dlms_Digital_Para`: 数字签名参数(输出transaction-id/系统标题等)
  - `PubKey`: 发送方公钥(64字节)
  - `DataLn`: 数据长度(输入/输出)
  - `Force`: 强制模式标志(0=标准解析, 1=直接验证签名)
- **返回值**: `0x00`=验证成功, `0x01`=验证失败
- **核心逻辑**:
  - Force=0时: 解析transaction-id(8B) + originator-system-title(8B) + recipient-system-title(8B) + date-time + other-information + content + signature(64B)
  - Force=1时: 直接取末尾64字节作为签名
  - 调用`FUC_EcdsaVerify()`进行ECDSA验证
  - 验证后将content数据移到缓冲区开头
- **条件编译**: 仅在`COMM_SUIT >= CS_SUIT1`时编译

#### 4. FUC_AlogDisitalSign (L332-L402)
- **原型**: `unsigned char FUC_AlogDisitalSign(unsigned char* DataBuf, ST_DLMS_DIGITAL_PARA* st_Dlms_Digital_Para, unsigned char* PrivKey, unsigned short* DataLn, unsigned char Force)`
- **功能**: 数字签名生成 — 使用ECDSA算法对数据签名
- **参数**:
  - `DataBuf`: 待签名数据缓冲区(输入/输出，签名后添加签名数据)
  - `st_Dlms_Digital_Para`: 签名参数(含系统标题、transaction-id等)
  - `PrivKey`: 签名方私钥(32字节)
  - `DataLn`: 数据长度(输入/输出)
  - `Force`: 强制模式标志(1=直接追加签名, 0=构建完整签名帧)
- **返回值**: `TRUE`=签名成功, `FALSE`=签名失败
- **核心逻辑**:
  - Force=1: 直接调用`FUC_EcdsaSignature()`追加64字节签名
  - Force=0: 构建完整签名帧: tag(1B) + transaction-id(9B) + originator-sys-title(9B) + recipient-sys-title(9B) + date-time(1B) + other-info(1B) + content-length + content + signature-length(1B) + signature(64B)
- **条件编译**: 仅在`COMM_SUIT >= CS_SUIT1`时编译

#### 5. FUC_AlogGMACUnwrap (L414-L889)
- **原型**: `unsigned char FUC_AlogGMACUnwrap(unsigned char* DataBuf, unsigned char CommType, unsigned short* pusDataLn)`
- **功能**: DLMS加密解包 — 核心解密函数，支持所有DLMS加密类型
- **参数**:
  - `DataBuf`: 加密数据缓冲区(输入/输出，解密后为明文)
  - `CommType`: 通信端口类型
  - `pusDataLn`: 数据长度(输入/输出)
- **返回值**: `0x00`=成功, `0x01`=明文不需解密, `0xF1`=格式/密钥错误, `0xF2`=系统标题/计数器错误, `0xFF`=解密失败
- **支持的加密类型**:
  | 标签 | 类型 | 密钥来源 |
  |------|------|----------|
  | `D_glo_initiateRequest` | 全局加密-连接请求 | unicast/broadcast EK |
  | `D_glo_Get/Set/ActionRequest` | 全局加密-操作请求 | unicast/broadcast EK |
  | `D_ded_Get/Set/ActionRequest` | 专用加密-操作请求 | dedicated EK |
  | `D_general_glo_ciphering` | 通用全局加密 | identified/wrapped/agreed key |
  | `D_general_ded_ciphering` | 通用专用加密 | dedicated EK |
  | `D_general_ciphering` | 通用加密(Suite1) | identified/wrapped/agreed key |
- **密钥类型(ucKeyType)**:
  - `KT_UNICAST(0)`: 单播密钥 — EK[0:16], 按SC的BIT6区分
  - `KT_BROADCAST(1)`: 广播密钥 — EK[16:32], SC的BIT6=1
  - `KT_DEDICATED(3)`: 专用密钥 — 从`st_Dlms_DedKeyPara`获取
  - `KT_WRAPPED(4)`: 包装密钥 — 用MasterKey通过AES-UNWRAP解密
  - `KT_AGREED(5)`: 协商密钥 — 通过ECDH+KDF派生
- **核心逻辑**:
  1. 根据APDU标签判断加密类型，提取发起者系统标题
  2. 解析安全控制字节(SC): BIT5=加密, BIT4=认证, BIT6=密钥集, BIT0-1=安全套件
  3. 根据密钥类型加载对应加密密钥和认证密钥
  4. 验证安全策略合规性(请求加密/认证 vs 策略要求)
  5. 验证安全套件版本
  6. 比较调用计数器(Invoke Counter)，防重放攻击
  7. 更新调用计数器到非易失存储
  8. 根据SC的BIT4-5确定解密模式:
     - `0x10`: 仅认证(AES-GMAC) — 附加数据=SC+AK+明文, Tag=12字节
     - `0x20`: 仅加密(AES-GCM) — 无附加数据, 无Tag
     - `0x30`: 认证+加密(AES-GCM) — 附加数据=SC+AK, Tag=12字节
  9. 构建IV = SystemTitle(8B) + InvokeCounter(4B)
  10. 调用`Decrypt_ByteData()`执行AES-GCM/GMAC解密
- **agreed-key协商流程**:
  - 方式1(One-pass Diffie-Hellman): 临时密钥对生成 → ECDH → KDF
  - 方式2(Static Unified Model): 静态KA密钥 → ECDH → KDF

#### 6. FUC_AlogGMACWrap (L891-L1213)
- **原型**: `unsigned char FUC_AlogGMACWrap(unsigned char *DataBuf, unsigned short *DataLn)`
- **功能**: DLMS加密组包 — 核心加密函数，构建加密APDU
- **参数**:
  - `DataBuf`: 明文数据缓冲区(输入/输出，加密后为密文APDU)
  - `DataLn`: 数据长度(输入/输出)
- **返回值**: `0x00`=成功, `0x01`=失败
- **核心逻辑**:
  1. 根据SC的BIT4-5确定加密模式(与Unwrap对应)
  2. 构建附加数据(AAD): SC + AK + 明文(认证模式)或SC + AK(认证加密模式)
  3. 对于`D_general_ciphering`类型，额外构建: transaction-id + recipient-sys-title + originator-sys-title + date-time + other-info
  4. 处理密钥协商:
     - wrapped-key: 生成随机EK → AES-WRAP包装
     - agreed-key: ECDH协商 → KDF派生
  5. 构建IV = ServerSysTitle(8B) + InvokeCounter(4B)
  6. 调用`Encrypt_ByteData()`执行AES-GCM/GMAC加密
  7. 追加Tag(12字节，认证模式)
  8. 构建帧头: 标签 + 系统标题 + 密钥信息 + 长度 + SC + InvokeCounter
  9. 将帧头+密文+Tag组合为完整加密APDU

---

### 三、APDU分析和封装函数 (2个)

#### 7. FUC_DlmsApduAnalyse (L1216-L1269)
- **原型**: `unsigned char FUC_DlmsApduAnalyse(unsigned char* DataBuf, unsigned char CommType, unsigned short* pusDataLn)`
- **功能**: DLMS APDU分析主入口 — 接收方向的安全处理总调度
- **参数**:
  - `DataBuf`: APDU数据缓冲区(输入/输出)
  - `CommType`: 通信端口类型
  - `pusDataLn`: 数据长度(输入/输出)
- **返回值**: `0`=成功, `0x01`=签名验证失败, `0x02`=加密验证失败
- **核心逻辑**:
  1. 清零`st_Dlms_Digital_Para`
  2. 若首字节为`D_general_signing`，先验证数字签名
  3. 验证系统标题匹配(客户端/服务器)
  4. 清零`st_Dlms_EncryptPara`
  5. 调用`FUC_AlogGMACUnwrap()`执行解密
  6. 检查安全策略: 若策略要求签名但未签名，返回错误
  7. 若解密后仍为`D_general_signing`，再次验证签名(双层签名场景)
  8. 清除`guc_ReadClass15NotSecurityFlag`标志

#### 8. FUC_DlmsApduWrap (L1272-L1329)
- **原型**: `unsigned char FUC_DlmsApduWrap(unsigned char* DataBuf, unsigned short* DataLn)`
- **功能**: DLMS APDU封装主入口 — 发送方向的安全处理总调度
- **参数**:
  - `DataBuf`: APDU数据缓冲区(输入/输出)
  - `DataLn`: 数据长度(输入/输出)
- **返回值**: `0x00`=成功, `0x01`=失败
- **核心逻辑**:
  1. 检查`guc_ReadClass15NotSecurityFlag`(Class15读取可跳过安全)
  2. 判断是否需要加密:
     - 已标记加密(`D_Ciphering`)
     - 或已建立连接 + 非PC客户端 + 策略要求认证/加密响应
  3. 若需加密但未加载加密参数，根据策略确定SC值:
     - 仅认证: SC = 0x10 + Suite
     - 仅加密: SC = 0x20 + Suite
     - 认证+加密: SC = 0x30 + Suite
  4. 加载InvokeCounter，调用`FUC_AlogGMACWrap()`
  5. 递增InvokeCounter
  6. 若请求有签名(`D_general_signing`)，调用`FUC_AlogDisitalSign()`添加响应签名

---

### 四、COSEM对象处理函数 (5个)

#### 9. FV_CosemInfoInit (L1348-L1351)
- **原型**: `void FV_CosemInfoInit(void)`
- **功能**: COSEM信息初始化 — 清零所有端口的COSEM结构体
- **调用时机**: 系统初始化或通信会话开始时

#### 10. FUC_GetCosemInfo (L1354-L1785)
- **原型**: `unsigned char FUC_GetCosemInfo(unsigned char CommType)`
- **功能**: 解析APDU中的COSEM对象描述 — 从通信缓冲区提取操作参数
- **参数**: `CommType` — 通信端口类型
- **返回值**: `0xFF`=成功, `0x00`=解析失败
- **解析的命令类型**:
  | 命令 | 控制类型 | 解析内容 |
  |------|----------|----------|
  | `Comm_CMD_READ` | `Comm_READ_NOM` | 单个读取: ClassID + OBIS + Attribute + AccessSelector |
  | | `Comm_READ_BLK` | 块读取: 续传块号 |
  | | `Comm_READ_LIST` | 列表读取: 多组ClassID + OBIS + Attribute |
  | `Comm_CMD_WRITE` | `Comm_WRITE_NOM` | 单个写入: ClassID + OBIS + Attribute |
  | | `Comm_WRITE_FSTBLK` | 写入首块: ClassID + OBIS + BlockCnt + DataLen |
  | | `Comm_WRITE_BLK` | 写入后续块: LastBlockFlag + BlockCnt + DataLen |
  | | `Comm_WRITE_LIST` | 列表写入: 多组ClassID + OBIS + Attribute |
  | | `Comm_WRITE_BLK_LIST` | 块列表写入 |
  | `Comm_CMD_ACTION` | `Comm_ACTION_NOM` | 单个执行: ClassID + OBIS + Method + AccessSelector |
  | `Comm_CMD_ACCESS` | `Comm_CMD_ACCESS` | 访问请求: InvokeID(4B) + 多组Type + ClassID + OBIS + Attr |
- **解析结果存入**: `gst_Cosem[CommType]`结构体

#### 11. FV_Comm_OperateProcess (L1787-L2013)
- **原型**: `void FV_Comm_OperateProcess(unsigned char CommType, unsigned char Operate)`
- **功能**: 通信操作分发 — 根据ClassID调用对应的COSEM类处理函数
- **参数**:
  - `CommType`: 通信端口类型
  - `Operate`: 操作类型(`Comm_READ`/`Comm_WRITE`/`Comm_ACTION`)
- **核心逻辑**:
  1. 从`gst_Cosem`获取当前COSEM描述符的ClassID、OBIS、Attribute
  2. 调用`Comm_GetClassIDPtr()`验证ClassID有效性
  3. 写入/执行操作时检查设备状态(是否上电完成、是否允许写入)
  4. 按ClassID switch分发到对应处理函数:
     - Class 1(数据), 3(寄存器), 4(扩展寄存器), 5(需求响应), 6(时钟)
     - Class 7(配置文件), 8(脚本表), 9(脚本), 11(特殊日期表)
     - Class 15(安全套件), 17-23, 27-29, 40-47, 64, 70-71, 90-92
     - Class 111-112, 113(STS), 115-116, 122
  5. 将操作结果(0=成功, 1=OBIS错误, 2=权限错误, 3=属性错误, 4=方法错误)写入`gst_Cosem.useResult`
  6. 写入/执行成功时记录参数变更事件

#### 12. FUC_WriteDataFram (L2199-L2311)
- **原型**: `unsigned char FUC_WriteDataFram(unsigned char CommType, unsigned short usNeedDataLn)`
- **功能**: 写入数据帧 — 从通信缓冲区提取写入数据到COSEM数据缓存
- **参数**:
  - `CommType`: 通信端口类型
  - `usNeedDataLn`: 需要的数据长度(`0xFF`=OCTETSTRING类型自动解析长度)
- **返回值**: `0x00`=数据未收完(块传输中), `0x01`=数据收完
- **核心逻辑**:
  1. 首次读取时设置`D_CosemFirstRead`和`F_Comm_Block`标志
  2. 块写入处理: 将跨帧数据拼接到`stCommData.ucCommDataBuf`
  3. 非块写入: 直接从`g_Comm.Buff`复制指定长度数据
  4. `usNeedDataLn=0xFF`时自动解析OCTETSTRING长度(首字节为长度)
  5. 数据收完后清除`F_Comm_Block`标志

#### 13. FUC_CompositeDataFram (L2319-L2956)
- **原型**: `unsigned char FUC_CompositeDataFram(unsigned char* ucDataBuf, unsigned char * Src, unsigned short usDataLn, unsigned char CommType, unsigned short* pusAllDataPtr)`
- **功能**: 复合数据帧构建 — 根据操作类型构建完整的响应APDU
- **参数**:
  - `ucDataBuf`: 输出缓冲区
  - `Src`: 源数据(各ClassID处理函数的输出)
  - `usDataLn`: 源数据长度
  - `CommType`: 通信端口类型
  - `pusAllDataPtr`: 已写入总长度(输入/输出，用于多对象累加)
- **返回值**:
  - `0`: 正常完成
  - `1`: 需要块传输(数据超出单帧)
  - `2`: 写入数据已用完(需等待下一块)
  - `3`: 需要GBT传输(Access大数据)
- **构建的响应类型**:
  | 操作 | 控制类型 | 响应结构 |
  |------|----------|----------|
  | READ | NOM | 响应标签 + 控制 + InvokeID + 结果(0/错误码) + 数据 |
  | READ | BLK | 响应标签 + 控制 + InvokeID + LastFlag + BlockCnt + 长度 + 数据 |
  | READ | LIST | 响应标签 + 控制 + InvokeID + 数量 + [结果+数据]×N |
  | WRITE | NOM | 响应标签 + 控制 + InvokeID + 结果 |
  | WRITE | FSTBLK | 响应标签 + 控制 + InvokeID + BlockCnt |
  | WRITE | BLK | 响应标签 + 控制 + InvokeID + LastFlag + BlockCnt + 结果 |
  | WRITE | LIST | 响应标签 + 控制 + InvokeID + 数量 + [结果]×N |
  | ACTION | NOM | 响应标签 + 控制 + InvokeID + 结果 + 返回值 |
  | ACCESS | - | 响应标签 + InvokeID(4B) + datetime + 描述 + [数据]×N + [结果]×N |

#### 14. FUC_CosemProcess (L2961-L3108)
- **原型**: `unsigned char FUC_CosemProcess(unsigned char CommType)`
- **功能**: COSEM处理主循环 — 遍历所有COSEM对象，执行操作并构建响应
- **参数**: `CommType` — 通信端口类型
- **返回值**: `0`=成功, `1`=权限不足
- **核心逻辑**:
  1. 设置不活动超时(120s/300s)
  2. 备份通信缓冲区到`gucUserBuff`
  3. 调用`FUC_GetCosemInfo()`解析请求
  4. 遍历所有COSEM对象(`useCosemPtr`从0到`ucCosemNum-1`):
     - 检查客户端权限(APPLink状态、ClientSap类型)
     - 按操作类型调用`FV_Comm_OperateProcess()`
     - 调用`FUC_CompositeDataFram()`构建响应
     - 处理GBT标志(大数据需要GBT传输)
  5. 将响应数据从`gucGBTTestBuf`复制回`g_Comm.Buff`
  6. 解析失败时返回通用错误响应(0xD8 0x01 0x01)

---

### 五、参数管理函数 (3个)

#### 15. FV_LoadServerPara (L181-L222)
- **原型**: `void FV_LoadServerPara(unsigned char flag)`
- **功能**: 加载服务器安全参数 — 从EEPROM/FLASH读取所有密钥和证书
- **参数**: `flag` — 强制加载标志(`TRUE`=无条件加载)
- **加载内容**:
  | 存储地址 | 内容 | 长度 |
  |----------|------|------|
  | `COMM_EK1` | 加密密钥(EK) | 32字节(含unicast+broadcast) |
  | `COMM_AK1` | 认证密钥(AK) | 32字节 |
  | `COMM_SEC_POLICY1` | 安全策略 | 1字节 |
  | `COMM_SEC_SUITE1` | 安全套件 | 1字节 |
  | `D_SERVERSIGNPRIVEKEY` | 服务器签名私钥 | 32字节 |
  | `MXF_IUT_S_CERTIFICATE` | 服务器签名证书(含公钥) | - |
  | `D_SERVERKAPRIVEKEY` | 服务器KA私钥 | 32字节 |
  | `MXF_IUT_KA_CERTIFICATE` | 服务器KA证书(含公钥) | - |
  | `MXF_CLIENT_SIGN_CERTIFICATE` | 客户端签名证书(含公钥) | - |
- **校验机制**: 使用`CheckNum()`计算校验和，不匹配时重新加载
- **条件编译**: Suite1相关密钥仅在`COMM_SUIT >= CS_SUIT1`时加载

#### 16. FV_LoadDedicatedKey (L231-L239)
- **原型**: `void FV_LoadDedicatedKey(unsigned char* Key, unsigned char CommType)`
- **功能**: 加载专用密钥 — 为指定通信端口设置专用加密密钥
- **参数**:
  - `Key`: 16字节专用密钥数据
  - `CommType`: 通信端口类型(需<COMM_PORTCNT)
- **操作**: 设置`ucDedicatedKeySta=1`，复制16字节密钥

#### 17. FV_CloseDedicatedKey (L247-L251)
- **原型**: `void FV_CloseDedicatedKey(unsigned char CommType)`
- **功能**: 关闭专用密钥 — 清除指定端口的专用密钥
- **操作**: 清零整个`ST_DLMS_DedKey`结构(含状态和密钥数据)

---

### 六、工具和辅助函数 (2个)

#### 18. FV_ErrorJudge (L2015-L2049)
- **原型**: `void FV_ErrorJudge(unsigned char* ucDataBuf, unsigned char CommType)`
- **功能**: 错误码映射 — 将内部错误标志转换为DLMS标准错误码
- **参数**:
  - `ucDataBuf`: 输出缓冲区(写入1字节错误码)
  - `CommType`: 通信端口类型
- **错误码映射**:
  | 内部标志 | DLMS错误码 | 含义 |
  |----------|------------|------|
  | `Comm_CMD_ERR_NOM` | 0x0C | 其他错误(未指定) |
  | `Comm_CMD_ERR_DATA + ERR_CLASSID + ERR_OBIS` | 0x04 | 对象不可用 |
  | `Comm_CMD_ERR_RANK` | 0x0B | 权限不足 |
  | `Comm_CMD_ERR_ATTR` | 0x03 | 属性不可用 |
  | `Comm_CMD_ERR_METHOD` | 0x02 | 方法不可用 |
  | `Comm_CMD_ERR_OTHER` | COMM_OTHER_REASON | 其他原因(含紧急状态码) |

#### 19. DlmsAdjustLen (L3110-L3130)
- **原型**: `unsigned char DlmsAdjustLen(unsigned char *out, unsigned short num)`
- **功能**: DLMS长度编码 — 将数值编码为BER长度格式
- **参数**:
  - `out`: 输出缓冲区
  - `num`: 待编码数值
- **返回值**: 编码占用的字节数
- **编码规则**:
  | 数值范围 | 编码方式 | 字节数 |
  |----------|----------|--------|
  | 0-127 | 直接编码 | 1 |
  | 128-255 | 0x81 + 值 | 2 |
  | 256-65535 | 0x82 + 高字节 + 低字节 | 3 |

---

## 关键全局变量

### 加密相关
| 变量 | 类型 | 说明 |
|------|------|------|
| `st_Dlms_Digital_Para` | `ST_DLMS_DIGITAL_PARA` | 数字签名参数(签名类型/系统标题/transaction-id) |
| `st_Dlms_ServerPara` | `ST_DLMS_SERVER_PARA` | 服务器参数(所有密钥/策略/证书) |
| `st_AsymmetricKeyPara[3]` | `ST_ASYMMETRIC_KEY_PARA` | 非对称密钥参数(3组) |
| `st_Dlms_DedKeyPara[COMM_PORTCNT]` | `ST_DLMS_DedKey` | 专用密钥参数(按端口) |
| `st_Dlms_EncryptPara` | `ST_DLMS_ENCRYPT_PARA` | 当前加密参数(SC/密钥/计数器/类型) |

### GBT相关
| 变量 | 类型 | 说明 |
|------|------|------|
| `gucGBTTestBuf[10000]` | `unsigned char[]` | GBT数据缓冲区(10KB) |
| `gusGBTTestPtr` | `unsigned short` | GBT缓冲区写入指针 |
| `gucAccessGBTFlg` | `unsigned char` | Access响应需GBT标志 |

### COSEM相关
| 变量 | 类型 | 说明 |
|------|------|------|
| `gst_Cosem[COMM_PORTCNT]` | `ST_COSEM` | 各端口COSEM处理状态 |
| `sucOldCosemPtr` | `unsigned char` | 上一个处理的COSEM索引(列表去重) |
| `gucUserBuff[DSU_COMMBUF_SIZE]` | `unsigned char[]` | 用户缓冲区(通信缓冲区备份) |

### 常量表
| 变量 | 说明 |
|------|------|
| `cucAlgorithmID[4][7]` | AES-GCM-128/256, AES-WRAP-128/256的OID |
| `cucAccessSeleHead[20]` | Access请求选择器头部模板 |
| `cucReadGBTList[2][6]` | 需要GBT传输的OBIS列表 |

---

## 数据流完整流程

### 接收方向
```
HDLC帧 → APDU数据
    │
    ▼
FUC_DlmsApduAnalyse()
    ├── [有签名?] → FUC_AlogDigitalSignVerify() → 验证系统标题
    ├── FUC_AlogGMACUnwrap()
    │       ├── 判断加密类型(glo/ded/general)
    │       ├── 加载密钥(unicast/broadcast/dedicated/wrapped/agreed)
    │       ├── 验证安全策略 + 套件 + 调用计数器
    │       └── Decrypt_ByteData() → 明文APDU
    └── [策略要求签名?] → 二次签名验证
    │
    ▼
FUC_GetCosemInfo()
    ├── 解析命令类型(READ/WRITE/ACTION/ACCESS)
    ├── 解析控制类型(NOM/BLK/LIST)
    └── 提取ClassID + OBIS + Attribute + AccessSelector
    │
    ▼
FUC_CosemProcess()
    └── for each COSEM object:
        ├── 权限检查(ClientSap/APPLink状态)
        ├── FV_Comm_OperateProcess() → Comm_ClassIDxx()
        ├── FUC_WriteDataFram() [写入时]
        └── FUC_CompositeDataFram() → 响应APDU
```

### 发送方向
```
响应APDU数据
    │
    ▼
FUC_DlmsApduWrap()
    ├── [需加密?] → FUC_AlogGMACWrap()
    │       ├── 构建AAD(SC+AK+数据/头)
    │       ├── [agreed-key?] → ECDH + KDF
    │       ├── [wrapped-key?] → 随机EK + AES-WRAP
    │       ├── Encrypt_ByteData() → 密文+Tag
    │       └── 构建帧头(标签+SysTitle+密钥信息+长度+SC+IC)
    ├── 递增InvokeCounter
    └── [需签名?] → FUC_AlogDisitalSign() → ECDSA签名
    │
    ▼
加密APDU → HDLC帧
```

---

## 支持的COSEM Class ID一览

| Class ID | 类名 | 支持的操作 |
|----------|------|------------|
| 0x0001 | Data | 读/写/执行 |
| 0x0003 | Register | 读 |
| 0x0004 | Extended Register | 读 |
| 0x0005 | Demand Register | 读 |
| 0x0006 | Clock | 读/写 |
| 0x0007 | Profile Generic | 读(含范围选择器) |
| 0x0008 | Script Table | 读/执行 |
| 0x0009 | Schedule | 读/写/执行 |
| 0x000B | Special Days Table | 读/写 |
| 0x000F | Security Setup | 读/写/执行 |
| 0x0011-0x0013 | Image Transfer/Association/Disconnect | 读/写/执行 |
| 0x0014 | Activity Calendar | 读/写 |
| 0x0015-0x0017 | Association LN/Push/Modem | 读/写/执行 |
| 0x001B-0x001C | Data Protection/Disconnector | 读/写/执行 |
| 0x001D | Push Setup | 读/写 |
| 0x0028 | Auto Answer | 读/写 |
| 0x0029-0x002D | PPP/GSM/IEC/HDLCLCP/MACAddress/IP4Setup | 读/写 |
| 0x002F | IPv6 Setup | 读/写 |
| 0x0040 | G3-PLC MAC Setup | 读/写 |
| 0x0046 | G3-PLC6LoWPAN | 读/写 |
| 0x0047 | Limiter | 读/写/执行 |
| 0x005A-0x005C | G3-PLC相关 | 读/写 |
| 0x006F-0x0070 | MBus Client/Port | 读/写 |
| 0x0071 | STS预付费 | 读/写/执行 |
| 0x0073-0x0074 | Token相关 | 读/写/执行 |
| 0x007A | 数据集合 | 读/写 |
