# 制图规范详解（SPEC）

> 供 mapcraft（制图助手）Skill 驱动的字段级 schema。**唯一事实源在主仓库**；本文件只描述协作方需要知道的内容，不复制主仓库解析代码。
> 若发现本文件与主仓库 `shared/map-parser` + `shared/types` 冲突，以主仓库为准并在合并时重跑校验。
> 主仓库位于```https://github.com/Dai-Yizhou/tycoon-game```，**制图者及其agent对主仓库仅有只读权限**。

---

## 0. 解析顺序（必须先 map-meta 后 map.json）

1. 先解析 `*.map-meta.json` → 得到 **UCT 字段全集**、玩家/区域初始值、区域框架、昼夜/骰子/税配置。
2. 再按这份约定解析 `*.map.json` 的格子字段。**与某格子无关的 UCT 键运行时不会读取。**

---

## 1. 通用费用类型（UCT）

形状：

```jsonc
{
  "player": { "money": -10, "credit": 2, "env": 0 },   // 作用于目标"玩家"
  "region": { "pros": 1 }                              // 作用于目标"区域"
}
```

**自定义权限（协作者）**：UCT 的字段集合由协作者在 `map-meta.json` 的 `uct` 中声明，**可自由新增/删减** player/region 字段（如加作 `reputation` 声望），无需开发者介入。新字段须同步补进 `valueFieldDefinitions` 名目（同名 id、作用域、min/max）。反过来，凡是本图未声明的字段，运行时一律当成错误。因此"能设计什么、不能写超"的边界一目了然：**声明了就能用，没声明就是配置错误**。

规则：

- **字段全集**：只能出现本图 `map-meta.uct` 声明的键（`uct.player[]` 与 `uct.region[]`）。未声明字段 = 配置错误。
- **稀疏省略（合法缺省）**：与当前格子/操作无关的键可省略，缺省按 **0** 处理（UCT 零元，不是行为回退）。
- **符号与方向（统一 `+=`，符号写在数据里）**：
  - 代码一律 `field += uct.field`；方向由数值符号承载。
  - **允许同一 UCT 混合符号**且是预期用法。例：`repairCost = { "player": { "money": -20, "credit": 5 }, "region": { "pros": 10 } }` ⇒ 玩家扣 20 钱 + 加 5 信用、区域加 10 繁荣。schema **不拒绝**混合符号。
- **股份分配**：分红/扣款时，每个 `player` 字段都按同一股比分配 `owner.field += floor(delta.field × share)`；`floor` 保留 `delta.field` 的符号。`region` 字段不按玩家股比拆分。

---

## 2. map-meta.json

```jsonc
{
  "id": "example-map",
  "version": "2.0.0",
  "name": { "zh-CN": "示例地图", "en-US": "Example Map" },

  // —— 数值字段架构（UCT 全集的"名称/作用域/上下限"来源，不含 current）——
  // 注意：min 是"破产阈值"而非扣减下限（负债式破产，见要点）；max 是数值上界
  "valueFieldDefinitions": [
    { "id": "money",  "name": { "zh-CN": "财产", "en-US": "Money" },      "scope": "player", "min": 0 },
    { "id": "credit", "name": { "zh-CN": "信用", "en-US": "Credit" },     "scope": "player", "min": 0 },
    { "id": "pros",   "name": { "zh-CN": "繁荣", "en-US": "Prosperity" }, "scope": "region", "min": 0, "max": 100 }
  ],
  "uct": { "player": ["money", "credit"], "region": ["pros"] },

  // —— 玩家初始状态（UCT，仅 player 键）+ 初始格 ——
  "playerInitial": { "player": { "money": 1000, "credit": 50 } },
  "startCellId": 0,

  // —— 区域表：每行自带初始状态（UCT，仅 region 键）——
  "regions": [
    { "id": "r1", "name": { "zh-CN": "东区", "en-US": "East" }, "initial": { "region": { "pros": 50 } } }
  ],

  // —— 昼夜周期（分钟，整数，缺省 15）与白昼占比（可选，缺省 0.5 即白昼 06:00-18:00）——
  "dayNightCycle": 24,
  "dayNightRatio": 0.5,

  // —— 昼夜 UCT 增量（可选）：切到白天/夜晚时对区域字段各施加一次增量，符号写在数据里 ——
  "dayNight": { "day": { "region": { "pros": 20 } }, "night": { "region": { "pros": -20 } } },

  // —— 骰子 ——
  "dice": { "cooldownMs": 3000, "min": 1, "max": 3 },

  // —— 实时榜单（可选）——
  "ranking": { "enabled": true, "topN": 5, "refreshMs": 1000,
               "score": { "constant": 0, "player": { "money": 1, "credit": 0.5 }, "region": { "pros": 0 } } },

  // —— 计税 ——
  "tax": {
    "baseTax": {
      "rates":       { "player": { "money": 0.02 } },   // 逐字段税率 [0,1]；未列字段不征
      "exemptBelow": { "player": { "money": 1000 } },   // 可选；字段低于该阈值免
      "taxInterval": 900000
    },
    "shareTax": {
      "rates":       { "player": { "money": 10 } },     // 每股应税额（按持股数量）
      "exemptBelow": 0,                                 // 总持股量低于该值免（0=不免）
      "taxInterval": 900000
    }
  },

  // —— 数值联动（D8，已实现，可选）：让数值随状态变量联动，详见 §4 ——
  "valueModifiers": [
    { "scope": { "cellType": "property", "base": "price" },
      "calc": { "player": { "money": { "$op": "sub", "args": [ { "$ref": "base.player.money" }, { "$op": "mul", "args": [ { "$ref": "region.uct.pros" }, 50 ] } ] } } } }
  ],

  // —— 长期离线清理（可选，未配置则关闭）：回收离线玩家占用的股东位 ——
  "inactivityCleanup": { "thresholdMs": 604800000, "sweepIntervalMs": 60000 }
}
```

要点：

**i18n 格式合规（中文工作流）**：所有 `name`（含 `valueFieldDefinitions[].name`、`regions[].name`）、`description` 必须保持 i18n 对象结构 `{"zh-CN": "…", "en-US": ""}`。**中文工作流不强制填全**：`zh-CN` 必填，`en-US` 可留空（`""`）供多语言后续填充。**禁止**写成裸字符串（如 `"name": "起点"`）——那是格式错误。AI 生成时自动补全 `{zh-CN, en-US}` 外壳。

- `valueFieldDefinitions` 只描述字段架构；初值由 `playerInitial`（玩家）与 `regions[].initial`（区域）提供，**不重复**。
- **min = 破产阈值（负债式破产，主仓库 2026-10 新口径）**：`min` **不是扣减下限**——数值允许被扣成**负数**；任一 `current < min` 即触发破产。判定推迟到当前操作完整结算之后（"先结算、再破产"）。**主动操作**（购买/升级/传送/保释等）余额不足时服务端先给 `would_bankrupt` 预览、玩家确认后才执行（负债式二次确认）；**被动扣款**（租金、税收）无预检、按整数下限全额结算，扣成负数即破产。设计 `min` 与扣款量级时按"可负债"口径思考，勿再按"钳在 min 触底"理解。
- **租金全额结算**：付款方余额不足时不再拒付，实扣至负数并触发破产；股东照常收到**全额**租金（不受付款方 shortfall 影响）。
- `regions[]` **不**记录所辖格子（由各格 `regionId` 聚合）、**无颜色概念**。
- **昼夜（`dayNightCycle` 与可选 `dayNight`）**：`dayNightCycle` 决定周期分钟数（缺省 15）；**可选 `dayNightRatio`**（0,1 开区间）决定白昼占比，缺省 0.5（白昼 06:00-18:00，12:00 为白昼中点）。区域繁荣初值来自 `regions[].initial.pros`。**可选的 `dayNight`** 在切换白天/夜晚时对区域字段各施加一次 UCT 增量（`day:{region:{...}}` / `night:{region:{...}}`，符号写在数据里），**属于地图字段，可直接配置**。除此之外的"衰减/恢复系数、更新频率、繁荣度对租金/事件概率的影响系数"属于**服务端配置**；协作者**不得臆造** `nightDecayFactor` 之类的字段，若有需要上报开发者。
- **长期离线清理（可选 `inactivityCleanup`）**：`{ thresholdMs, sweepIntervalMs? }`。离线超过 `thresholdMs` 的玩家释放全部股份并置为破产态（不删号），用于回收被离线玩家永久占用的股东位；未配置则关闭。设计股东位上限（`maxOwnerCount`）时可把本机制纳入饱和考量。
- **实时榜单（可选 `ranking`）**：`{ enabled, topN, refreshMs, score:{ constant, player:{<字段>→权重}, region:{<字段>→权重} } }`。未配置则不启用榜单。
- **数值联动（可选 `valueModifiers`）**：见 §4。让 `price`/`rent`/`upgradeCost`/`teleportDestinations[].cost`/`investmentTriggers[].delta`/`jailCooldown`/`jailCost`/`repairCost` 随 `region.uct.*`/`region.time`/`player.uct.*`/`team.*`/`curCell.*` 联动。**同 `base` 只允许一条规则**（多条会因歧义被 lint 拒绝）。

---

## 3. map.json（格子）

### 3.1 公共字段（所有格子必填）

```
id, x, y,
type(= empty|supply|monument|property|investment|jail|transport|event),
name({zh-CN,en-US}), description({zh-CN,en-US}),
destinations(有向边 number[]),
regionId(必须存在于 map-meta.regions), timezone(UTC 偏移分钟,字面量),
theme(设计令牌 id，string 必填)
```

> **`theme`**：必填 string，指定该格使用的主题令牌 id（如 `northeast` / `south` / `midwest` / `west`，见主仓库 `packages/shared/design-tokens/themes/*.json`）。主仓库 `Cell` 类型将其声明为必填；空白会导致客户端主题渲染缺失，务必填写。

### 3.1.1 字段责任归属（哪些**不适合 agent 手填**）

不是所有字段都该由 agent（制图助手）逐字手填。下表字段可以**由协作者可视化/编辑器产出**，agent 不做精确手填，只表达意图：

| 字段 | 生产方式 | 说明 |
|---|---|---|
| `id` | 编辑器自动连续编号 | agent 如需编号需符合唯一连续 |
| `x` / `y` | **协作者在可视化编辑器**（`map-editor-v02/`）拖拽/点击画布生成 | agent **不手填精确坐标**。agent 产出的格子坐标只作初值/意图占位（如隐藏格 `900,900`），**正式排布以协作者在编辑器拖拽后的导出结果为准** |
| `destinations` | 编辑器连线生成 | agent 只说明"谁连到谁"的连通意图，最终连线由协作者在编辑器里拉出，除非协作者明确需要单向边 |

**文本字段（`name` / `description` / `msg`）——体现地图气质，归协作者掌控**：
- **协作者明确给出的文本，agent 原样采用、不润色**。文本是地图气质所在，agent 不得擅自改写措辞、风格或语气，也不为"统一/美观"代笔。
- **仅当协作者未给明确文本时**（只说效果、没定文案），agent 才协助起草中文文案并补全 `{zh-CN,en-US}` i18n 外壳，供协作者确认/调整。
- `en-US` 仍可留空待多语言后续填充；格式必须保持 i18n 对象结构（禁止裸字符串）。

### 3.2 按类型出现的字段

| 类型 | 专属字段 | 备注 |
|---|---|---|
| `empty` | — | 可有 description，零效果 |
| `supply` | `behaviorPass`、`behaviorLand`（均必填） | 经过=A；踩中=A→B |
| `monument` | `repairCost`(UCT) | 踩中可选修缮一次；符号内嵌 |
| `property` | `price`、`rent[]`、`upgradeCost[]`、`maxLevel`、`maxOwnerCount` | 踩中可购/升；不持股者踩中按 `rent[level]` 扣款→股东按股比分 |
| `investment` | `price`、`maxOwnerCount`、`investmentTriggers[]` | 条件触发，不限位置 |
| `jail` | `jailCooldown`、`jailCost`(UCT) | 入狱：冷却+、扣 credit、禁用收款 |
| `transport` | `teleportDestinations[]` | 停靠可选传送 |
| `event` | `behaviorLand` | 踩中按行为 effects 触发 |

> **合租（持股比例）无乘数字段**：地产/投资的合租持股比例 = 该玩家本次实际支付金额 ÷ 该项目累计投入金额（见主仓库 `Ownership.addOwnership`）。**不存在 `buyInMultiplier` 字段**——旧版该字段已被主仓库从 schema 移除，解析时若出现直接报错，禁止填写。
> 起点**不是** `start` 类型，由 `map-meta.startCellId` 指定。

### 3.3 property / investment / transport / monument 示例字段

```jsonc
{
  "id": 1, "x": 2, "y": 0, "type": "property",
  "name": { "zh-CN": "东区地皮", "en-US": "East Plot" },
  "description": { "zh-CN": "经典地产", "en-US": "Classic property" },
  "destinations": [2], "regionId": "r1", "timezone": 0,
  "maxOwnerCount": 5,
  "price":  { "player": { "money": -100 } },
  "maxLevel": 3,
  "rent": [ { "player": { "money": -10 } }, { "player": { "money": -20 } },
            { "player": { "money": -30 } }, { "player": { "money": -40 } } ],
  "upgradeCost": [ { "player": { "money": -50 } }, { "player": { "money": -100 } },
                   { "player": { "money": -150 } } ]
}
```

```jsonc
{
  "id": 9, "type": "monument",
  "name": { "zh-CN": "英雄碑", "en-US": "Hero Monument" },
  "destinations": [], "regionId": "r1", "timezone": 0,
  "repairCost": { "player": { "money": -20, "credit": 5 }, "region": { "pros": 10 } }
}
```

```jsonc
{
  "id": 6, "type": "transport",
  "destinations": [7], "regionId": "r1", "timezone": 0,
  "teleportDestinations": [
    { "cellid": 3, "cost": { "player": { "money": -10, "credit": -3 } } },
    { "cellid": 6, "cost": { "player": { "money": -20 }, "region": { "pros": -1 } } }
  ]
}
```

### 3.4 隐藏格（彩蛋）——用现有字段实现，零新字段

**目的**：做一张"正常骰子到不了"的奖励格，把玩家从入口送进去。**不新增字段、不加插件位**；超出下面模式的想法上报开发者。

**建格（关键纪律）**：
- 不把本格 `id` 写进任何格的 `destinations`（这样骰子永远到不了）。
- 坐标放到主图之外（如 `x:900,y:900`），视觉上不干扰。
- 类型常为 `event`（`behaviorLand` 触发奖励）；`destinations` 留空 `[]`（合法）。
- **仍必须有合法 `regionId`**：`map-parser` 会校验非空，所以必须挂在一个已存在区域，不能建"无区域格"。
- `name`/`description` 需保持 i18n 格式（`zh-CN` 必填，`en-US` 可留空）。

```jsonc
{ "id": 42, "x": 900, "y": 900, "type": "event",
  "name": { "zh-CN": "秘境", "en-US": "Hidden Realm" },
  "description": { "zh-CN": "隐藏奖励格", "en-US": "Hidden reward cell" },
  "destinations": [], "regionId": "r1", "timezone": 0,
  "behaviorLand": "hidden-reward" }
```

**入口 A：传送**（transport 格的 `teleportDestinations` 加该格；校验只查"id 存在"，不查"可达"，正好满足）：
```jsonc
"teleportDestinations": [ { "cellid": 42, "cost": { "player": { "money": -10 } } } ]
```

**入口 B：低概率事件送进去**（行为 `position.teleport`，`cellId` 指向 42；配 `exclusive:true` + 低 `weight` 当事先藏好的彩蛋事件）：
```jsonc
{ "type": "position", "target": "single", "action": "teleport", "cellId": 42 }
```

### 3.5 investmentTriggers 预定义事件（需申请新增）

`investment` 格的 `investmentTriggers[].on` **只能使用以下预定义事件**，不可自定义：

| 事件标识 | 触发时机 |
|---|---|
| `any-player-lands-event` | 任意玩家踩中任一 `event` 格 |
| `any-player-lands-monument` | 任意玩家踩中任一 `monument` 格 |
| `shareholder-bankrupt` | 任一持股人破产 |
| `day-started` | 昼夜周期进入白天 |
| `night-started` | 昼夜周期进入黑夜 |

> **差异点（重点）**：UCT 字段、behavior（`effects`/`ops`/`weight`/`exclusive` 组合）都由你**自行设计**；但 `investmentTriggers[].on` 的值是**服务端预定义的事件标识**。若想要表中之外的新触发时机（如"玩家升级地产""队伍清空某区域"），**向开发者申请**，等事件上线后才可引用，不得为绕过而臆造事件名。

---

## 4. 数值联动 valueModifiers（D8，已实现）

弥补"只有 behavior 能引用变量"的缺口：让**静态数值字段在结算/展示时随世界状态联动**。定义在 map-meta 的 `valueModifiers`（全局规则表，**不做 per-cell 声明**）。求值两端同构（`@game/shared`，`resolveField`），确定、可 lint。

**协作约束（不暴露语法）**：`calc` 的 `numExpr`/`uctValue` AST（§4.2）只作我内部落地用，**不向协作者展示**。对齐效果时用**自然语言 / 数学式 / 伪代码**（例："夜晚租金上浮 25%，股东越多越贵"）复述，确认后才落成 JSON；其他有复杂语法的配置项（如行为 `$ref` 算术、`investmentTriggers[].on`）同理。`$ref` 引用的合法锚在 §4.3 表中（越界引用由 lint 拒绝，不臆造锚）。

**out of scope（留给行为引擎，不进 modifier）**：随机数、查询函数（最近玩家/距离/排行）、"地产集合/垄断组"等跨格概念——这些仍走 behavior（§5），不写进 `valueModifiers`。

**客户端展示边界**：客户端悬浮展示**服务端权威 final**，`base → final` 仅作摘要提示、**不暴露计算式**；可与服务端同构求值做乐观显示，服务端在结算 ack 校正。

### 4.1 规则结构

```jsonc
{ "id": "可选-id", "scope": { "cellType": "property", "base": "price" }, "calc": <Calc> }
```
- `scope.base` = 作用到的**静态字段名**（标识维度，非取值机制）。
- `calc`：number 字段 → 单个 `numExpr`；UCT 字段 → `uctValue`。
- **同 `base` 只允许一条规则**（重叠歧义 → lint 拒绝）；不同 `base` 各一条。
- 结果 = **覆盖**：`calc` 显式列出的子字段替换为新值，**未列出的子字段保持原 base 值（不归零）**。想要增量效果，在表达式中用自指 `base` 显式 `$add`。（如 `price` 只改 `player.money`，`player.credit`/`region.*` 沿用原值。）

### 4.2 数值表达式 `numExpr`（统一前缀式）

```
node := number                       // 字面量
      | { "$ref": "<锚路径>" }        // 变量引用
      | { "$op": "<op>", "args": [ node, ... ] }
```
运算符（闭合集合）：算术 `add`(≥2) `sub`(≥2) `mul`(≥2) `div`(2) `neg`(1) `abs`(1) `min` `max` `round`(1) `clamp`(3)；条件 `gte` `lte` `eq` `and` `if`(3)。
UCT 写法：`uctValue := { player?: { <fieldId>: numExpr }, region?: { <fieldId>: numExpr } }`。**number 字段 → 单一 `numExpr`**（如 `jailCooldown`）。

### 4.3 可引用变量（refs，固定锚；叶节点字段名来自 `valueFieldDefinitions`）

| 锚 | 相对谁 | 说明 |
|---|---|---|
| `base`（保留字） | 目标字段自身 | 当前静态值（写入前）。number 字段直接整体引用（如 `{ "$ref": "base" }`）；UCT 字段须钻子字段（如 `base.player.money`），不带子字段整体只用于期待 UCT 的位置 |
| `player.uct.<fieldId>` | 付款方玩家 | 其 player 作用域子字段（购买=买家、付租=租客、保释=进监者、修复=所有者） |
| `team.uct.<fieldId>` | 所在团队 | 成员该字段**算术均值**；`memberCount==0` 或字段未声明 → lint 拒绝 |
| `team.memberCount` | 同上 | number |
| `region.uct.<fieldId>` | 目标格所在区域 | region 作用域子字段（如 `pros`/`env`） |
| `region.time` | 同上 | number：昼夜环境时刻，**白天=0 / 夜晚=1**（驱动 §4.4 昼夜联动） |
| `curCell.level` | 目标格 | number |
| `curCell.ownerCount` | 目标格 | number |

> 引用 UCT（`$ref` 或自指 `base`）必须带子字段才得 `number`；不带子字段只能作 UCT 整体（顶层写入）。
> 单调性：对 `region.uct.*`/`curCell.*`/`player.*` 尽量设计为单调增/减（lint 只做符号提示，非强校验）。

### 4.4 允许的 base（按 cellType）

| cellType | base 字段 | 类型     |
|---|---|-------|
| `property` | `price`、`rent[]`、`upgradeCost[]` | UCT     |
| `transport` | `teleportDestinations[].cost` | UCT     |
| `investment` | `price`、`investmentTriggers[].delta` | UCT     |
| `jail` | `jailCooldown`(number)、`jailCost` | numbUCT |
| `monument` | `repairCost` | UCT     |

空/supply/event 无 base（走 behavior）。数组字段（`rent[level]`、`teleportDestinations[i].cost`、`investmentTriggers[j].delta`）= 作用在**被消费的那个元素**，自指 `base` 取当前元素。

### 4.5 生命周期与 behavior 的关系

- **变更类**（实付、租金、升级费、传送费、保释金、修复费、投资 delta）：结算/购买时刻 `resolveField` 一次并固定，随 ack 返回实付。
- **展示类**：按当前上下文求值。
- **与 behavior 不相交**：behavior 只在 `supply/event`（无 base），valueModifiers 只作用于有 base 的格子；两套不共享字段、不叠加、无生效顺序冲突。

### 4.6 已实现的行为引擎 `$ref`（behavior 内变量，v2）

behavior JSON 里同样支持 `$ref` 变量（`$actor.money`、`$region.pros`、`$cell.id`、`$nearestPlayer` 等），可用于**概率或效果**（如税收/事件影响）。behavior 表达式为受限算术（`$ref` + `+ - * / min/max`，白名单），格式见 §5。

---

## 5. 行为 JSON（behaviorLand / behaviorPass）

被 `map.json` 的 `behaviorPass` / `behaviorLand` 按 **id** 引用。同图的行为 JSON 放该图 `behaviors/` 目录。

**自定义权限（协作者）**：行为 JSON 的内容（`effects` 数组、每个 effect 的 `weight`/`exclusive`/`msg`/`ops`、ops 三分支与实际取值）**完全由你设计**，无需开发者介入。只要行为 id 被本图引用即生效。唯一限制是 effects 语义也要遵守本节结构（选中规则、ops 分支、target、`$ref` 白名单），即"结构受约束、内容自由裁量"。

```jsonc
{
  "id": "event-lottery",
  "effects": [
    {
      "weight": 0.4,          // [0,1] 小数 或 {"$ref":"..."}
      "exclusive": false,     // false/缺省=独立（各按 weight 独立判定，可多命中）；true=互斥（组内归一仅命中其一）
      "msg": { "zh-CN": "好运：+20 现金", "en-US": "Lucky: +20 money" },
      "ops": [                // 一个效果可同时携带多个操作
        { "type": "value", "target": "single", "delta": { "player": { "money": 20 } } }
      ]
    },
    {
      "weight": 0.6, "exclusive": true,
      "msg": { "zh-CN": "厄运：-15 现金", "en-US": "Bad luck: -15 money" },
      "ops": [
        { "type": "value", "target": "single", "delta": { "player": { "money": -15 } } }
      ]
    }
  ]
}
```

### 5.1 效果选中（独立/互斥两类**分开触发**）

- 执行时**先处理独立效果**（`exclusive:false`，各按 `weight` 独立判定，可多命中），**再处理互斥组**（所有 `exclusive:true` 归一组，`weight` 归一后仅命中其一）。
- `weight` 可为 `number | {"$ref":"..."}`；互斥组全 0 视为均等。
- `target` 已**下移到每个 op**，必填，无默认掩盖。

### 5.2 ops 三分支（每项必带 `type` 与 `target`）

- **`value`**：`delta` 为 UCT，作用于解析后的目标玩家/区域，逐字段 `+=`。
- **`ownership`**：`action` = `acquireShare`（须明确单个来源格）| `loseShare`（可 `cells[]` 逐格，或用 `scope` 全量）。
  - 定位方式二选一（同时出现=错误）：
    - `cells[]`：`[ { "cellId": 3 }, { "cellId": { "$ref": "$cell.id" } }, ... ]` 逐格失股；
    - `scope`：`"all"`（property+investment）| `"all-property"` | `"all-investment"` —— 失去 target 在该范围内的全部股份（忽略 cells）。
- **`position`**：`action` = `moveTo`（沿有向边走动，会触发途经格的经过钩子）| `teleport`（瞬移，到达后作为新"踩中"结算）；`cellId` = 目标格（数字或 `$ref`）。

### 5.3 目标 target（op 级，必填）

`single`（触发者=`$actor`）| `team`（触发者队伍）| `region`（触发者所在区域全部玩家）| `globe`（全部玩家）| `{ "$ref": "..." }`（运行时变量解析出的玩家）。

### 5.4 运行时引用 `{$ref:"..."}`

- **上下文绑定**：`$actor`、`$target`、`$map`、`$cell`、`$region`；字段读取 `$actor.<fieldId>`（如 `$actor.money`、`$region.pros`）。
- **白名单解析器**（需计算量，仅白名单）：如 `$nearestPlayer`（距 `$actor` 最近的其他玩家）、`$randomShareholder`（任一股东）。
- **受限算术**：`+ - * /`、括号、`min/max`；作用于上下文变量与白名单结果。例：`"weight": { "$ref": "1 - (($actor.credit)/2000)" }`。
- **不支持**：任意函数、循环、随机源、IO。仅服务端求值；求值失败 = 配置错误（快速失败）。标量槽（UCT 数值、`weight`、`share`、`cellId`）允许 `number | {"$ref":"..."}`。

---

## 6. 快速失败（配置错误，不静默回退）

以下任一 = 配置错误，**报错→定位→修正**：

- `supply` 缺 `behaviorPass` / `behaviorLand`。
- `behaviorPass/Land` 指向本图 `behaviors/` 中不存在的行为 id。
- `op` 缺 `type`；`target` 缺失。
- `value.delta` 含有 UCT 未声明字段 / 非 number。
- `investmentTriggers[].on` 用了 3.5 预定义事件之外的值。
- `name`/`description` 写成裸字符串、或缺少 `zh-CN`（i18n 格式错误）。
- `regionId` 不在 `map-meta.regions`。
- `ownership` 同时给出 `cells` 与 `scope`。
- 注释文档 `NOTES.md` 缺失或与配置不一致。

> 需要区分**合法缺省**（不触发钩子、UCT 零元、`exclusive:false`）与**配置错误**（声明了但缺失/失效）。