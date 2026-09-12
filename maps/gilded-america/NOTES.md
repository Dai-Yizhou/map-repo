# 镀金年代·美国大城（Gilded America）— 配置项说明

> 本文件是该地图目录的**注释文档**：既记录**总体规划初稿**（设计说明），也是配置的进度台账。
> **约束**：任何修改 `*.map.json` / `*.map-meta.json` / `behaviors/*.behavior.json` 的改动，都必须同步在本文件更新对应说明，保持"配置 ↔ 说明"一致。

---

## 一、总体规划初稿（Design Draft）

**主题定位**：1900s 镀金时代 / 进步时代的美国大城——买地、办厂、投铁路、修地标，在繁荣与危机间经营。

### 分区 ×5（对应 map-meta `regions`）

| 区域 id | 主题 | 主要格子 | 气质侧重 |
|---|---|---|---|
| r1 市中心·金融区 | 摩天楼、银行、证交所 | 高价值 `property` + `investment` + `monument` | `money` / `credit` / `reputation` |
| r2 工业区 | 炼钢厂、纺织厂、码头 | `supply`(原料补给) + `property`(厂址) + 劳工 `event` | `pros`(生产繁荣) |
| r3 铁路与港埠 | 火车站、渡轮、电报局 | `transport`(跨区传送核心) | 连通各区 |
| r4 移民聚居街区 | 唐人街 / 意大利 / 爱尔兰街区 | 中低 `property` + 密集 `event` + 生活 `supply` | 烟火气 |
| r5 郊野与都会边缘 | 公园、农场、港口雕像 | `empty` + `monument` + 隐藏彩蛋格 | 松弛感 |

### 格子类型配比（约 80 格）

```
property   28    各区地产/厂址/楼宇（核心经济）
event      15    历史事件（暴击/危机加权随机）
supply     10    补给（经过/踩中）
transport   8    铁路/码头传送枢纽
investment  6    铁路/石油/钢铁公司股票（条件触发分红）
monument    6    自由女神、钢铁桥等地标（修缮→加信用/声望/繁荣）
jail        2     警察局·禁酒执法（入狱）
empty       5     公园/空地过渡
合计       80
```

### 签名机制（建议，待逐步落格）

1. **铁路传送网**：靠 `transport.teleportDestinations` 付费跨区，体现铁路时代。
2. **经济大事件表**：`event` 里配置权重化的「股市崩盘 / 大罢工 / 淘金热 / 大火」等暴击/危机。
3. **昼夜型产销**：用 `investmentTriggers` 的 `day-started` / `night-started` 做昼夜差异（仅预定义事件，不臆造字段）。

---

## 二、地图概览（Current Fact）

- 字段全集（UCT）：player `money/credit/reputation`，region `pros`（来自 map-meta `uct`）。
- 起点格：`startCellId=0`（类型 `supply`），出发点「市政厅广场·出发」。
- 昼夜：`dayNightCycle=30` 分钟。
- 开局：money 1000 / credit 50 / reputation 0。

---

## 三、`gilded-america.map-meta.json`（元数据 / 全局配置）

| 配置项 | 含义 |
|---|---|
| `valueFieldDefinitions` | 声明本图允许的数值字段：money / credit / reputation（player），pros（region，max 100）。 |
| `uct` | player `[money, credit, reputation]`，region `[pros]`。 |
| `playerInitial` | 开局初值：money 1000 / credit 50 / reputation 0。 |
| `startCellId` | `0`，起点格为 `supply`。 |
| `regions` | r1 金融区 / r2 工业区 / r3 铁路港埠 / r4 街区 / r5 郊野，各带 `pros` 初值。 |
| `dayNightCycle` | `30` 分钟一个昼夜循环。 |
| `dice` | cooldownMs 3000，步数 min 1 / max 3。 |
| `tax` | baseTax：财产 2%、信用 1%，财产低于 1000 免征；shareTax：每股 10；计税周期 15 分钟。 |

---

## 四、`gilded-america.map.json`（格子表）

| id | 类型 | 说明 | 关键配置 |
|---|---|---|---|
| 0 | `supply` | 起点「市政厅广场·出发」 | 经过/踩中行为 `start-bonus`（+10 现金）；连线待协作者在编辑器排布。 |

### 方向约定
- **玩家支付/被扣 = 负，获得 = 正**；代码永远 `+=`。

---

## 五、`behaviors/`（行为 JSON，被 `behaviorPass`/`behaviorLand` 按 id 引用）

| 行为 id | 触发场景 | 效果 |
|---|---|---|
| `start-bonus` | 经过/踩中格 0 | +10 现金（single） |

行为内 `ops.type` 三种：`value` / `ownership` / `position`。

---

## 六、维护规则（必须遵守）

1. 修改任何 `map` / `map-meta` / `behaviors` 配置，**必须**同步更新本文件对应说明。
2. 不要使用 `map-meta.uct` 全集之外的字段。
3. 不要改变"玩家支付为负、获得为正"的方向约定。
4. 新增区域须在 `map-meta.regions` 登记，格子才能引用。
5. 本图变更后，入库前必须通过主仓库 schema 校验（快速失败，不静默兜底）。