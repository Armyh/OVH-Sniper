# OVH Sniper - OVH 服务器自动抢购工具

🎯 纯 Go 语言编写的 OVH 独立服务器自动监控和抢购工具，支持多账户、多机房、多配置并发监控。

## ✨ 主要特性

- 🚀 **纯净抢购** - 无 Web 界面，纯命令行运行，资源占用极低
- 👥 **多账户支持** - 同时监控多个 OVH 账户，互不干扰
- 🌍 **多地区监控** - 支持监控多个数据中心（RBX、GRA、SBG 等）
- ⚙️ **灵活配置** - 支持自定义硬件配置选项（带宽、内存、存储）
- 🔥 **热重载** - 修改配置文件后自动生效，无需重启
- 💨 **高并发抢单** - 发现有货时使用多个并发快速下单
- 🔍 **智能监控** - 并发检查 + 随机延迟，避免 API 限流
- 📱 **Telegram 通知** - 支持 Telegram Bot 实时通知
- 🛡️ **线程安全** - 使用 RWMutex 保证多账户并发安全

## 📦 快速开始

### 1. 下载

```bash
# 克隆项目
git clone <repository-url>
cd ovh-sniper-go

# 运行
./ovh-sniper
```

### 2. 配置文件

编辑 `config.json`，填入你的 OVH API 凭据。

## ⚙️ 配置说明

### 监控配置 (monitor)

| 参数 | 说明 | 默认值 | 推荐值 |
|------|------|--------|--------|
| `check_interval` | 每隔几秒检查一次库存 | 5 | 3-10 秒 |
| `check_concurrency` | 每个账户同时检查几个目标（避免请求过多） | 3 | 3-5 |
| `order_concurrency` | 发现有货后同时下几个订单 | 10 | 5-20 |
| `random_delay_min` | 随机延迟最小值，避免被限流（毫秒） | 100 | 50-200ms |
| `random_delay_max` | 随机延迟最大值（毫秒） | 500 | 300-1000ms |

**示例配置：**

```json
{
  "monitor": {
    "check_interval": 5,
    "check_concurrency": 3,
    "order_concurrency": 10,
    "random_delay_min": 100,
    "random_delay_max": 500
  }
}
```

### Telegram 通知配置

```json
{
  "telegram": {
    "enabled": false,
    "bot_token": "",
    "chat_id": ""
  }
}
```

**如何获取 Telegram 配置：**

1. **创建 Bot**：与 [@BotFather](https://t.me/BotFather) 对话，发送 `/newbot` 创建机器人
2. **获取 Token**：BotFather 会给你一个 `bot_token`
3. **获取 Chat ID**：
   - 与你的 Bot 发送任意消息
   - 访问：`https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates`
   - 在返回的 JSON 中找到 `chat.id`

### 账户配置 (accounts)

**API 端点说明：**

| 端点 | 地区 | API 地址 |
|------|------|----------|
| `ovh-eu` | 欧洲 | https://eu.api.ovh.com/createToken/ |
| `ovh-ca` | 加拿大 | https://ca.api.ovh.com/createToken/ |
| `ovh-us` | 美国 | https://us.api.ovh.com/createToken/ |

**如何获取 API 凭据：**

1. 访问对应地区的 API Token 创建页面
2. 设置权限：
   - `GET /dedicated/server/*`
   - `POST /order/*`
   - `PUT /order/*`
3. 点击创建，保存 `Application Key`、`Application Secret`、`Consumer Key`

**示例配置：**

```json
{
  "accounts": [
    {
      "name": "主账户",
      "endpoint": "ovh-eu",
      "app_key": "YOUR_APP_KEY",
      "app_secret": "YOUR_APP_SECRET",
      "consumer_key": "YOUR_CONSUMER_KEY",
      "ovh_subsidiary": "IE",
      "enabled": true,
      "targets": [...]
    }
  ]
}
```

### OVH Subsidiary 说明

`ovh_subsidiary` 字段决定账户的地区和税率，不同地区的价格和服务条款可能不同。

**常用代码：**

| 代码 | 国家/地区 | 说明 |
|------|---------|------|
| `IE` | 爱尔兰 | 欧洲默认地区 |
| `FR` | 法国 | 法国本土 |
| `DE` | 德国 | 德国本土 |
| `ES` | 西班牙 | 西班牙本土 |
| `GB` | 英国 | 英国本土 |
| `IT` | 意大利 | 意大利本土 |
| `NL` | 荷兰 | 荷兰本土 |
| `PL` | 波兰 | 波兰本土 |
| `PT` | 葡萄牙 | 葡萄牙本土 |
| `CA` | 加拿大 | 加拿大地区 |
| `US` | 美国 | 美国地区 |

**注意事项：**
- 如果不设置，默认使用 `IE`（爱尔兰）
- `ovh_subsidiary` 必须与你的账户地区匹配
- 不同地区的价格、税率、服务条款可能不同

### 监控目标配置 (targets)

**参数详解：**

| 参数 | 说明 | 示例 |
|------|------|------|
| `name` | 目标名称，用于日志识别 | `"KS-LE-B NVMe 高配"` |
| `plan_code` | 产品代码，在 OVH 官网查看 | `"26skleb01-v1"` |
| `datacenters` | 机房列表<br>• 空数组 `[]` = 监控所有机房<br>• 指定机房 `["rbx", "gra"]` = 只监控这些机房 | `["rbx", "gra", "sbg"]` |
| `auto_order` | • `true` = 有货自动下单<br>• `false` = 只监控通知 | `true` |
| `quantity` | 有货时抢购几台 | `1` 或 `2` |
| `options` | 硬件升级选项，空数组表示默认配置 | 见下方 |

**常用机房代码：**

| 代码 | 机房名称 | 位置 |
|------|---------|------|
| `rbx` | Roubaix | 法国鲁贝 |
| `gra` | Gravelines | 法国格拉夫林 |
| `sbg` | Strasbourg | 法国斯特拉斯堡 |
| `bhs` | Beauharnois | 加拿大博阿尔努瓦 |

**硬件配置选项示例：**

```json
{
  "options": [
    "bandwidth-500-26skle",
    "ram-32g-ecc-2400-26skleb01-v1",
    "softraid-2x450nvme-26skleb01-v1"
  ]
}
```

**如何查找配置选项代码：**

1. 访问 OVH API 文档：`https://eu.api.ovh.com/console/`
2. 使用接口：`GET /order/cart/{cartId}/eco/options?planCode={planCode}`
3. 查看返回的可用选项列表

**完整示例：**

```json
{
  "targets": [
    {
      "name": "KS-LE-B 默认配置",
      "plan_code": "26skleb01-v1",
      "datacenters": ["rbx", "gra"],
      "auto_order": true,
      "quantity": 1,
      "options": []
    },
    {
      "name": "KS-LE-B NVMe 高配",
      "plan_code": "26skleb01-v1",
      "datacenters": [],
      "auto_order": true,
      "quantity": 2,
      "options": [
        "bandwidth-500-26skle",
        "ram-32g-ecc-2400-26skleb01-v1",
        "softraid-2x450nvme-26skleb01-v1"
      ]
    },
    {
      "name": "只监控不下单",
      "plan_code": "26skleb01-v1",
      "datacenters": ["rbx"],
      "auto_order": false,
      "quantity": 1,
      "options": []
    }
  ]
}
```

## 📋 完整配置示例

```json
{
  "monitor": {
    "check_interval": 5,
    "check_concurrency": 3,
    "order_concurrency": 10,
    "random_delay_min": 100,
    "random_delay_max": 500
  },

  "telegram": {
    "enabled": true,
    "bot_token": "123456789:ABCdefGHIjklMNOpqrsTUVwxyz",
    "chat_id": "123456789"
  },

  "accounts": [
    {
      "name": "主账户",
      "endpoint": "ovh-eu",
      "app_key": "your_app_key",
      "app_secret": "your_app_secret",
      "consumer_key": "your_consumer_key",
      "ovh_subsidiary": "IE",
      "enabled": true,
      "targets": [
        {
          "name": "KS-LE-B 默认配置",
          "plan_code": "26skleb01-v1",
          "datacenters": ["rbx", "gra"],
          "auto_order": true,
          "quantity": 1,
          "options": []
        }
      ]
    },

    {
      "name": "备用账户",
      "endpoint": "ovh-eu",
      "app_key": "another_app_key",
      "app_secret": "another_app_secret",
      "consumer_key": "another_consumer_key",
      "ovh_subsidiary": "FR",
      "enabled": false,
      "targets": [
        {
          "name": "备用目标",
          "plan_code": "26skleb01-v1",
          "datacenters": ["gra"],
          "auto_order": true,
          "quantity": 1,
          "options": []
        }
      ]
    }
  ]
}
```

## 🚀 运行逻辑

### 监控架构

```
多账户并发监控：
├── 账户1 (独立协程)
│   ├── 并发检查目标1 ──┐
│   ├── 并发检查目标2   │ 最多 3 个并发 (check_concurrency)
│   ├── 并发检查目标3   │ 每次检查前随机延迟 100-500ms
│   └── 并发检查目标4 ──┘
│   └── 发现有货 ──> 开启 10 个并发抢单 (order_concurrency)
│
├── 账户2 (独立协程)
│   ├── 并发检查目标1 ──┐
│   └── 并发检查目标2   │ 独立运行，互不干扰
│   └── 发现有货 ──> 开启 10 个并发抢单
│
└── 账户3 (独立协程)
    └── ...
```

### 工作流程

1. **初始化**
   - 加载配置文件 `config.json`
   - 为每个启用的账户创建 OVH API 客户端
   - 为每个账户启动独立的监控协程

2. **监控阶段**
   - 每个账户独立运行，互不干扰
   - 使用并发检查该账户的所有目标
   - 每次检查前随机延迟，避免 API 限流
   - 检测到库存状态变化时触发通知

3. **下单阶段**
   - 检测到 `unavailable` → `available` 时触发
   - 如果 `auto_order: true`，立即开启高并发抢单
   - 使用信号量控制并发数，避免请求过多
   - 实时记录成功/失败数量

4. **热重载**
   - 监控 `config.json` 文件变化
   - 检测到修改后自动重新加载配置
   - 清空旧的监控状态
   - 重新初始化所有客户端

## 📊 日志说明

运行时的日志输出示例：

```
🎯 OVH Sniper - 纯净抢购程序 (支持热重载)
========================================
✅ 从 config.json 加载了 2 个账户
========================================
👥 账户数量: 2 个
📊 监控目标: 4 个
⏱️  检查间隔: 5 秒
🔍 检查并发: 3
💨 下单并发: 10
⏲️  随机延迟: 100-500 毫秒
📱 Telegram 通知: 已启用

📋 监控目标列表:

  账户: 主账户 (ovh-eu)
    1. 26skleb01-v1 | 机房: [rbx gra] | 配置: 默认配置 | 自动下单: ✅ | 数量: 1
    2. 26skleb01-v1 | 机房: 所有机房 | 配置: [...] | 自动下单: ✅ | 数量: 2

  账户: 备用账户 (ovh-eu)
    3. 26skleb01-v1 | 机房: [gra] | 配置: 默认配置 | 自动下单: ✅ | 数量: 1

========================================
🔥 配置热重载已启用，修改 config.json 后自动生效
🚀 监控已启动
🔍 开始监控账户: 主账户
🔍 [主账户] 检查 26skleb01-v1 的可用性...
⏳ [主账户] 26skleb01-v1@rbx 当前无货
✅ [主账户] 26skleb01-v1@gra 当前有货
🛒 [主账户] 开始下单: 26skleb01-v1@gra (配置: 默认配置, 数量: 1)
💨 [主账户] 使用 10 个并发下单
✅ [主账户] 下单成功 (1/1): 26skleb01-v1@gra
📊 [主账户] 下单完成: 26skleb01-v1@gra - 成功 1 台, 失败 0 台
```

## 🛡️ 安全建议

1. **保护配置文件**
   - `config.json` 包含 API 凭据，请勿泄露
   - 已添加到 `.gitignore`，不会被提交到 Git

2. **API 权限控制**
   - 建议只给必要的 API 权限
   - 定期更换 API 密钥

3. **并发控制**
   - 不要设置过高的并发数，避免触发 OVH API 限流
   - 推荐：`check_concurrency: 3`, `order_concurrency: 10`

4. **随机延迟**
   - 必须启用随机延迟，避免被识别为机器人
   - 推荐：`100-500ms`

## 🔧 常见问题

### Q1: 如何查看可用的产品代码？

访问 OVH 官网的 Kimsufi/SoYouStart 页面，在产品详情页 URL 中可以找到 `plan_code`。

### Q2: 如何查找硬件配置选项代码？

使用 OVH API Console：
```
GET /order/cart/{cartId}/eco/options?planCode={planCode}
```

### Q3: 热重载不生效？

检查文件监控是否启动成功，如果看到 `⚠️ 启动配置监控失败`，需要手动重启程序。

### Q4: 提示 API 限流怎么办？

- 降低 `check_concurrency` 和 `order_concurrency`
- 增加 `check_interval`
- 增大随机延迟范围


## 📄 许可证

MIT License

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

## 🙏 致谢

基于原项目 [OVH-BUY](https://github.com/coolci/OVH-BUY) 的核心逻辑重写。

---

**注意：** 本工具仅供学习和个人使用，请遵守 OVH 服务条款，合理使用 API。
