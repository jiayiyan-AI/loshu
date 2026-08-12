# 飞书多维表格数据结构调研

> 状态：调研记录
> 日期：2026-08-12
> 方法：Chrome DevTools 接管真实登录会话，抓取 `xcnpxpdgobfj.feishu.cn` 上一个真实 Base 的全部接口响应，解压后逐字段分析。**本文所有结论都来自抓到的报文，不来自文档或猜测**；无法从报文确认的部分单列在第 10 节。

被观察的 Base：`G8Nbb0OW8aUwUCsHAZech1hFnnd`（"✅任务管理"，由官方任务管理模板复制而来），调研期间人为添加了一个单向关联字段、一个双向关联字段，以便观察关联的下发形态。

---

## 1. 一句话结论

飞书的线上格式**不是**「一个 schema JSON + 一批记录」。它是**按加载单元切分的多份报文**：Base 骨架、单张表的 schema+首屏行、关联表的 schema+行、仪表盘的 widget 树、每个图表的聚合结果，各自独立请求、独立编码、独立版本号。

其中最值得注意的三点，都在后文有报文佐证：

1. **Base 的成员是同构的 block，不是「表 + 别的东西」**（第 2 节）。
2. **视图不携带数据，只携带「怎么看」的参数；行序和分组结果由服务端单独下发**（第 5 节）。
3. **算出来的值一律不在行里**：关联的显示文案被服务端 join 进单元格，公式结果被彻底剥离到独立通道（第 6、7 节）。

---

## 2. 一个"项目"是怎么定义的：Base 与 block

`clientvars` 响应体里 `data.base` 字段解开后是 Base 本身：

```json
{
  "id": "7673052492816452544",
  "token": "G8Nbb0OW8aUwUCsHAZech1hFnnd",
  "name": "✅任务管理",
  "rev": 4,
  "owner": "7637435848350321600",
  "objType": 8,
  "subType": 0,
  "schemaVersion": 5,
  "timezone": "Asia/Shanghai",
  "calcMode": 3,
  "enableFmlDataOptimization": 1,
  "baseEngineEnabled": false,
  "isProBase": false,
  "baseRecordsNum": 23,
  "rankInfo": { "initRank": "i00000000", "rankStep": 1048576 },
  "blocks": [ /* 见下 */ ],
  "blockInfos": { /* 见下 */ }
}
```

### 2.1 成员是一个有序的 block 列表

```json
"blocks": [
  "tbl3iNXJ8JDbEHG9",     // ✅任务管理（表）
  "blkc2e5swqxonarH",     // 任务统计看板（仪表盘）
  "ldx5RuegwRd2Exou",     // 💡使用说明文档（文档）
  "wkf2P1mAdYVd13VX",     // 重要任务每日定时提醒（自动化）
  "tblddtXJIo2katnD",     // 单向关联原始表（表）
  "tblLBe1aYHhczZw6"      // 双向关联原始表（表）
]
```

**表、仪表盘、文档、自动化在这里是平级的。** 数组顺序就是左侧边栏顺序 —— 注意后加的两张表排在仪表盘和文档之后，与界面呈现一致。

`blockInfos` 给出每个成员的类型：

```json
{
  "tbl3iNXJ8JDbEHG9": { "id": "tbl3iNXJ8JDbEHG9", "name": "✅任务管理" },
  "tblddtXJIo2katnD": { "id": "tblddtXJIo2katnD", "name": "单向关联原始表" },
  "tblLBe1aYHhczZw6": { "id": "tblLBe1aYHhczZw6", "name": "双向关联原始表" },
  "blkc2e5swqxonarH": { "id": "blkc2e5swqxonarH", "name": "任务统计看板",
                        "blockToken": "dbdcn5Tmus5KHfdsRLsyXpMw2fe", "blockType": 36 },
  "ldx5RuegwRd2Exou": { "id": "ldx5RuegwRd2Exou", "name": "💡使用说明文档",
                        "blockToken": "NoKed6pzooQ12QxXhNXcqWgonBd", "blockType": 22 },
  "wkf2P1mAdYVd13VX": { "id": "wkf2P1mAdYVd13VX", "name": "重要任务每日定时提醒",
                        "blockToken": "7673052305370156247", "blockType": 86 }
}
```

三条可直接读出的规律：

| 观察 | 报文依据 |
| --- | --- |
| **表没有 `blockType`，非表才有** | 三个 `tbl*` 条目只有 `id` + `name`；仪表盘 22/36/86 都带 `blockType` |
| **非表 block 的内容不在 Base 里，只留一个 `blockToken` 指针** | 仪表盘的 `blockToken` 是 `dbdcn5...`，与 Base token 不同源，要另发请求 |
| **表虽然没写 `blockType`，服务端内部仍给它编了号** | `GET /block_info?table=tbl3iNXJ8JDbEHG9` 返回 `"blockType": 83`；`view_type` 接口也返回 `"blockType": 83` |

也就是说：**表在数据模型上是一种 block（type=83），只是因为它是默认成员而在 `blockInfos` 里省略了类型标记**，且它的内容直接内联在 Base 的加载流程里，其余 block 一律外挂。

### 2.2 权限挂在 block 粒度上

```json
"permissionMap": { "blocks": {
  "tbl3iNXJ8JDbEHG9": { "visible": true },
  "blkc2e5swqxonarH": { "visible": true },
  "ldx5RuegwRd2Exou": { "visible": true },
  "wkf2P1mAdYVd13VX": { "visible": true },
  "tblddtXJIo2katnD": { "visible": true },
  "tblLBe1aYHhczZw6": { "visible": true }
} }
```

可见性以 block 为单位声明，不区分类型 —— 进一步印证 block 是统一的权限主体。

### 2.3 ID 体系

抓到的所有 ID 都带类型前缀，且长度固定：

| 前缀 | 指向 | 样例 | 长度 |
| --- | --- | --- | --- |
| `tbl` | 表 | `tbl3iNXJ8JDbEHG9` | 16 |
| `vew` | 视图 | `vewXxBNTOK` | 10 |
| `fld` | 字段 | `fldaxqIJ1m` | 10 |
| `rec` | 记录 | `rec0oA6okm` / `recvs4vBWaak1t` | 10 或 14 |
| `opt` | 选项 | `opt0dhXwYV` | 10 |
| `blk` | 仪表盘 | `blkc2e5swqxonarH` | 16 |
| `wkf` | 自动化 | `wkf2P1mAdYVd13VX` | 16 |
| `ldx` | 文档 | `ldx5RuegwRd2Exou` | 16 |
| `cht` | 图表 | `chtcnz5ODmUvjx87se0Gb0ShUKd` | 27 |
| `dbdcn` | 仪表盘文档 token | `dbdcn5Tmus5KHfdsRLsyXpMw2fe` | 27 |

值得注意的是 **`rec` 出现了两种长度**：模板自带的旧记录是 10 位（`rec0oA6okm`），调研期间新建的记录是 14 位（`recvs4vBWaak1t`、`recv3FLSCcnnU3`）。同一张表里两种长度共存，说明 ID 生成规则改过版，且**消费方不能假设 ID 定长**。

另外 Base 同时有 `id`（雪花号 `7673052492816452544`）和 `token`（`G8Nbb0OW8aUwUCsHAZech1hFnnd`）。所有 URL 用 token，`owner` / `createdUser` 等用户标识用雪花号。

---

## 3. 每张表是怎么定义的

`clientvars` 响应体的 `data.table` 解开后是单张表的完整定义。顶层键（主表实测）：

```
meta · views · viewMap · fieldMap · primaryKey · recordMap · recordMeta
rankInfo · groupList · formulaInfo · userMap · mentionMap · resourceMap
commentMap · milestoneMap · cascadeProxyFieldMap · fieldGroups
currentView · recordCount · viewRecordNum · viewGroupNum · viewTopLevelRecordNum
recordPage · cs · latestCSRev · deniedRecords · tablePerm · schemaVersion
exInfo · useNewGroup · weakErrCode · tableLimit · enableViewCache
```

**注意：一次 `clientvars` 只返回一张表。** Base 里有 3 张表，就要发 3 次（第 8 节）。

### 3.1 `meta`：表级版本号

```json
{ "id": "tbl3iNXJ8JDbEHG9", "rev": 10, "schemaVersion": 5,
  "recordsNum": 13, "level": 0, "depRev": "", "jointRev": 26,
  "exType": 0, "archiveEnabled": false, "archiveRecordsNum": 0 }
```

三个版本号并存，实测值不同（`rev=10` / `jointRev=26` / `schemaVersion=5`），说明它们计数的是不同的东西。可佐证的一点：**`jointRev` 与关联相关** —— 三张表的实测值是

| 表 | 角色 | `rev` | `jointRev` |
| --- | --- | --- | --- |
| `tbl3iNXJ8JDbEHG9` 主表 | 持有两个关联字段 | 10 | 26 |
| `tblLBe1aYHhczZw6` 双向对端 | 被双向关联 | 8 | 29 |
| `tblddtXJIo2katnD` 单向对端 | 被单向关联 | 4 | **-1** |

**单向关联的目标表 `jointRev = -1`，另外两张都是正数。** 单向的对端不需要参与联动（它自己不知道谁引用了它），双向的两侧都要。这是一处很强的信号：单向与双向在服务端是**不同的机制**，不只是"少建一个反向字段"。

### 3.2 `fieldMap`：字段定义

```json
"fldaxqIJ1m": { "name": "任务描述", "type": 1, "property": null }
```

抓到的字段一律是 `{ name, type, property }` 三件套（`isPrimary` 只在个别字段上出现，主键的权威来源是顶层 `primaryKey`）。

**字段类型是数字枚举。** 本 Base 实测到的：

| type | 含义 | `property` 实测内容 |
| --- | --- | --- |
| 1 | 文本 | `null` |
| 3 | 单选 | `{ options: [{id, name, color}], optionsType: 0 }` |
| 5 | 日期 | `{ dateFormat: "yyyy/MM/dd", timeFormat: "", autoFill: bool }` |
| 11 | 人员 | `{ multiple: false }` |
| 18 | **单向关联** | `{ baseId, tableId, viewId, multiple }` |
| 20 | 公式 | `{ formula, formatter, currencyCode }` |
| 21 | **双向关联** | `{ baseId, tableId, viewId, backFieldId, backFieldName, multiple }` |

两处可直接读出的设计取舍：

- **格式参数下沉到 `property`，不上浮到字段顶层。** 日期的 `dateFormat`、公式的 `currencyCode`、选项集、`multiple` 全在 `property` 里，字段顶层只有 `name`/`type`。
- **公式字段的 `property` 里混着 `currencyCode` 和 `formatter`，即使公式返回的是文本。** 实测该字段返回 `"🚨 已延期"`，但 `currencyCode: ""`、`formatter: ""` 依然存在 —— `property` 是按 kind 分支的松散袋子，不是紧凑联合类型。

### 3.3 公式的引用是全限定的

```
IF(OR(AND(TODAY()>bitable::$table[tbl3iNXJ8JDbEHG9].$field[flddOQSEwM],
          bitable::$table[tbl3iNXJ8JDbEHG9].$field[fldJjwpiC3]!="已完成"),
      bitable::$table[tbl3iNXJ8JDbEHG9].$field[fldXq5jUcN]
        >bitable::$table[tbl3iNXJ8JDbEHG9].$field[flddOQSEwM]),
   "🚨 已延期","✅ 正常")
```

**引用同一张表的字段也写全限定路径** `bitable::$table[tbl].$field[fld]`，而不是裸 `fldXXX`。这让公式文本在跨表引用时不必换语法，代价是可读性。注意引用用的是 ID 不是列名 —— 改列名不需要改公式。

同时可以看到：**公式里的字面量是渲染后的展示文本**（`!="已完成"` 比的是选项名而非 `optRz8Igse`；返回值直接是带 emoji 的字符串）。这说明公式引擎工作在"显示值"层面，不在存储值层面。

---

## 4. 一张表的多个视图是怎么定义的

主表 `views` 数组（有序）：

```json
["vewXxBNTOK", "vew0Tu0VCB", "vewi5IHsPO", "vewabHZr1x", "vewJRDhHGr"]
```

`viewMap` 给出每个视图的定义，结构统一：

```json
{
  "id": "vewXxBNTOK",
  "name": "任务管理表",
  "type": 1,
  "isPrivate": false,
  "publicLevel": 0,
  "owner": "7637435848350321600",
  "privateViewOwner": "",
  "bizType": 2,          // 只在部分视图上出现
  "property": { /* 类型相关 */ }
}
```

实测到的 `type`：`1` = 表格，`2` = 看板。

### 4.1 视图的公共部分与类型专属部分

比较表格视图与看板视图的 `property`，可以清楚看到分层：

**公共（两种视图都有）**

```json
"fields":     ["fldaxqIJ1m", "fldfbCy326", ...],   // 列顺序
"sortInfo":   [],
"group":      [{ "fieldId": "fld9cvGzic", "desc": false }],
"filterInfo": null,
"records":    []
```

**表格视图专属**

```json
"colInfos":       { "fldaxqIJ1m": { "width": 321, "hidden": false }, ... },
"rowHeightLevel": 1,
"frozenColCount": 1,
"cardViewSetting": null,
"hierarchyConfig": null,
"colorInfo":      null
```

**看板视图专属**

```json
"sectionInfos": { "fldXq5jUcN": { "hidden": true }, ... },
"cardConfig":   { "showFieldName": true, "coverFieldId": "",
                  "coverFitType": "", "showCover": false, "showMode": "" }
```

关键观察：**"哪些列可见"这件事在两种视图里用了不同的键** —— 表格是 `colInfos[fld].hidden`（与列宽同住），看板是 `sectionInfos[fld].hidden`（与卡片区块同住）。这不是同一个概念的两种写法：表格的隐藏是"不画这一列"，看板的隐藏是"不在卡片上列出这个字段"。合并成一个 `hidden` 会丢掉这层差别，但代价是消费方必须按视图类型分支取值。

另一处：`fields` 数组在所有视图里**都包含全部 12 个字段**，包括标记为 hidden 的。所以 `fields` 是**顺序**声明而非**可见集**声明，可见性完全由 `colInfos`/`sectionInfos` 决定。

### 4.2 分组在视图上，不在数据上

三个业务视图的差别只有 `group` 一项：

| 视图 | `type` | `group.fieldId` | 语义 |
| --- | --- | --- | --- |
| 任务管理表 | 1 表格 | `fld9cvGzic` 重要紧急程度 | 四象限 |
| 进度看板 | 2 看板 | `fldJjwpiC3` 进展 | 状态流转 |
| 人员任务分配看板 | 2 看板 | `fldp8FSfnw` 任务执行人 | 人员负载 |

同一份数据、同一套字段，靠切换 `group.fieldId` 得到三个管理视角。这是这个模板的核心设计，也是"视图 = 一组呈现参数"的最好例证。

### 4.3 双向关联会自动建一个视图

`vewJRDhHGr`（"表格 2"）在界面上不显示，但它在 `views` 里，且有两个特征：

```json
{ "id": "vewJRDhHGr", "name": "表格 2", "type": 1, "bizType": 2,
  "property": { "fields": [ /* 全部 12 个 */ ], "colInfos": {}, "group": [] } }
```

- `bizType: 2` —— 业务视图 `bizType` 为 `undefined`，只有它和同样疑似废弃的 `vewabHZr1x` 有这个标记。
- `colInfos` 是**空对象**，说明从没有人调过列宽 —— 不是人建的。

它被谁引用？在对端表的双向关联字段里：

```json
// tblLBe1aYHhczZw6 的字段 fldzGBZXWW
{ "name": "✅任务管理", "type": 21,
  "property": { "tableId": "tbl3iNXJ8JDbEHG9",
                "viewId": "vewJRDhHGr",        // ← 指向这个自动建的视图
                "backFieldId": "fldxu4KVBI",
                "backFieldName": "双向关联原始表" } }
```

**建立双向关联时，服务端在被关联表上自动创建了一个视图，供反向字段做"可选记录范围"的锚点。** 对照主表侧的正向字段 `fldxu4KVBI`，它的 `viewId` 是空串 —— 只有自动建的那一侧填了。

这说明关联字段的 `viewId` 是**候选集限定器**（"只能从这个视图里挑记录"），默认空表示全表；双向关联在自动建反向字段时，顺手固化了一个全量视图作为锚点。

---

## 5. 表里的"行"是怎么定义的

行数据不是一个数组，而是**拆成四份平行的 map**，用 recordId 关联：

| 结构 | 键 | 存什么 |
| --- | --- | --- |
| `recordMap` | recordId → fieldId → 单元格 | 用户填的值 |
| `recordMeta` | recordId → `recMeta` | 创建/修改的人与时间、行版本 |
| `rankInfo.rankMap` | recordId → rank 串 | 行序 |
| `groupList` | 数组 | 分组结果与组内行序 |

### 5.1 `recordMap`：值 + 每格自带的修改元信息

```json
"rec0oA6okm": {
  "fldaxqIJ1m": {
    "modifiedUser": "7637435848350321600",
    "modifiedTime": 1786521754,
    "value": [ { "text": "多维表格应用模式员工培训：", "type": "text" },
               { "text": "应用模板中心 ", "type": "mention",
                 "token": "YXgxbYlJSatOnvsQzDNcW7JQnng",
                 "link": "https://www.feishu.cn/app/YXgx...",
                 "mentionType": 8, "realMentionType": 8, "mentionNotify": false },
               { "text": "", "type": "text" } ]
  },
  "fldt93MWer": { "modifiedUser": "...", "modifiedTime": 1786521754,
                  "value": 1764086400000 },
  "fldJjwpiC3": { "...": "...", "value": "opth4KnfEy" }
}
```

三点：

1. **修改元信息下沉到单元格。** `modifiedUser` / `modifiedTime` 每一格都有一份，而不是只在行上。这是协同编辑必需的粒度（谁改了哪一格），代价是报文里这两个键重复了 `行数 × 列数` 次。
2. **文本值是富文本片段数组，不是字符串。** 即使是纯文本也包成 `[{type:"text", text:"..."}]`。这样 mention 才能作为同一数组里的另一种片段共存，不需要另立字段类型。
3. **各类型的值形状不同**：文本是数组，日期是毫秒时间戳（number），单选是 optionId（string），人员是 `{users:[{userId, name, enName, avatarUrl, mentionId, notify}]}`（对象）。

### 5.2 人员字段的值把用户信息冗余进了单元格

```json
"fldp8FSfnw": { "value": { "users": [ {
  "userId":   "6897851847462109187",
  "name":     "刘贝拉",
  "enName":   "刘贝拉",
  "avatarUrl": "https://s1-fs.pstatp.com/static-resource/v1/9beebed7-...",
  "mentionId": "4379eca4-0d2a-4a12-9627-6233fa3f46c9",
  "notify":   true
} ] } }
```

同时顶层还有一份 `userMap`：

```json
"userMap": { "7637435848350321600": { "name": "谭颢", "enName": "谭颢", "avatarUrl": "..." },
             "6897851847462109187": { "name": "刘贝拉", ... }, ... }
```

**同样的用户信息存了两遍**：单元格里一份（自包含），`userMap` 里一份（供 `modifiedUser`、`createdUser` 这类只有 ID 的地方查表）。前者让单元格能独立渲染，后者服务于元信息 —— 两种消费场景，所以没有合并。

### 5.3 `recordMeta`：行级审计与行版本

```json
"rec0oA6okm": { "recMeta": {
  "rev": 3,
  "createdTime": 1786521754, "createdUser": "7637435848350321600",
  "modifiedTime": 1786525798, "modifiedUser": "7637435848350321600"
} }
```

实测 `rev` 有 0 / 2 / 3 三种值，且**恰好等于该行被改过的次数** —— 模板导入时创建的行是 0，我加了关联值后变成 2 或 3。行有自己的版本号，与表的 `meta.rev` 独立。

### 5.4 `rankInfo`：行序是稀疏字符串，不是整数序号

```json
"rankInfo": {
  "nextRank": "i0008464g",
  "rankMap": {
    "rec0oA6okm":     "i00000000",
    "recHwz6PGh":     "i0000mh34",
    "recFYZDsHt":     "i00018y68",
    "rect4NklCs":     "i0001vf9c",
    ...
    "recvs4vcjo2KpG": "i0007hp1c"
  },
  "viewRankMap": { "vewXxBNTOK": {} }
}
```

对照 Base 里的 `rankInfo: { initRank: "i00000000", rankStep: 1048576 }`：首行 rank 就是 `initRank`，后续按 `rankStep` 递增，编码成 36 进制字符串。这是**分数索引（fractional indexing）**：两行之间插入新行只需生成一个介于两者之间的串，不必重排后面所有行 —— 协同场景下这能让"插入行"变成一次单点写入而非全表重排。

`viewRankMap` 为每个视图留了位置（实测为空），说明**行序可以按视图覆盖** —— 全局一份默认序，某个视图手工拖动后在这里存差异。

### 5.5 `groupList`：分组结果由服务端算好

```json
"groupList": [ {
  "by": null,
  "recordIDList": ["rec0oA6okm", "recHwz6PGh", ..., "recvs4vcjo2KpG"],
  "firstRecordOffset": 0,
  "groupRecordNum": 0
} ]
```

**它只有 recordId，没有值** —— 是一份"排好序的行 ID 清单"。前端拿它决定渲染顺序，再去 `recordMap` 取值。

有意思的是：视图 `vewXxBNTOK` 明明配了 `group: [{fieldId: "fld9cvGzic"}]`，但这里只有一个 `by: null` 的扁平组。而界面上确实是按四象限分组显示的。合理的解释是首屏走了 `loadingType: "rank_full"`（全量按 rank 下发，前端自行分组），分组结果的服务端下发形态是另一种 loadingType —— 但**本次没有抓到那种响应，不下断言**（见第 10 节）。

---

## 6. 关联数据是怎么来的

### 6.1 单元格里同时有外键和显示文案

```json
// 单向关联 (type=18)
"fldQewJDeO": {
  "modifiedUser": "7637435848350321600", "modifiedTime": 1786525798,
  "value": ["recvs4vigo65ZY"],
  "displayValue": { "recvs4vigo65ZY": "张三" }
}

// 双向关联 (type=21)
"fldxu4KVBI": {
  "value": ["recvs4vBWaak1t", "recvs4vBWah98p"],
  "displayValue": { "recvs4vBWaak1t": "工程师", "recvs4vBWah98p": "设计师" }
}
```

`value` 是目标行 ID 数组，`displayValue` 是**服务端 join 好的目标行主键字段文本**。前端渲染这一列不需要访问关联表。

注意 `value` 是**数组**（`multiple: true`），不是单值外键 —— 一格可以指向多行。这与关系型数据库的外键列不同，更接近把关联表内联进了单元格。

### 6.2 双向关联的两侧各存一份

对端表 `tblLBe1aYHhczZw6` 的同一条关联：

```json
// recvs4vBWah98p（"设计师"）的反向字段
"fldzGBZXWW": {
  "value": ["rec0oA6okm", "recHwz6PGh"],
  "displayValue": { "rec0oA6okm": "完成年度财务报告",
                    "recHwz6PGh": "组织年度员工团建活动" }
}
```

主表侧：`rec0oA6okm → [工程师, 设计师]`，`recHwz6PGh → [设计师]`
对端侧：`设计师 → [完成年度财务报告, 组织年度员工团建活动]`，`工程师 → [完成年度财务报告]`

**两侧数据完全一致，各存一份物化的多对多关系。** 不是一侧存、另一侧查出来 —— 反向字段 `fldzGBZXWW` 是对端表 `fieldMap` 里一个真实的 type=21 字段，有自己的 `modifiedTime`。

两侧字段互相指认：

```
主表  fldxu4KVBI.property.backFieldId = "fldzGBZXWW"
对端  fldzGBZXWW.property.backFieldId = "fldxu4KVBI"
```

这解释了 5.1 节看到的 `jointRev`：双向的两侧都要参与联动（改一边要同步另一边），所以都有正的 `jointRev`；单向对端 `jointRev = -1`，因为它压根不知道自己被引用了。

### 6.3 关联表被整表预加载

页面加载时，除主表外还发了两个 `clientvars`：

```
GET /clientvars?tableID=tblddtXJIo2katnD&viewID=&recordLimit=200&...
GET /clientvars?tableID=tblLBe1aYHhczZw6&viewID=&recordLimit=200&...
```

`viewID` 为空 —— 不针对任何视图，是把整张关联表（schema + 前 200 行）拉进前端。

这不是为了渲染关联列（`displayValue` 已经够了），而是为了交互：点单元格弹出的"选择关联记录"面板要能搜索、hover 卡片要展开完整行。**展示走服务端预 join，交互走前端整表副本**，两条路径。

代价是 `recordLimit=200` 写死：关联到几万行的表时前端只有前 200 行，交互层必然还要走别的接口兜底（本次未触发，未确认）。

---

## 7. 公式结果走独立通道

公式字段 `fld90fZ6tK` 在 `recordMap` 里的值是 `null`，且**该字段根本不在行的键列表里**：

```python
recordMap["rec0oA6okm"].keys()
# ['fldxu4KVBI','fldp8FSfnw','fldaxqIJ1m','fldq2mc2Al','fldXq5jUcN',
#  'fld9cvGzic','fldfbCy326','fldJjwpiC3','flddOQSEwM','fldt93MWer','fldQewJDeO']
# ← 没有 fld90fZ6tK
```

真正的值在顶层 `formulaInfo`：

```json
"formulaInfo": {
  "code": 0,
  "calcCompletionTime": 1786525829,
  "fieldBusTypeMap": { "fld90fZ6tK": 201 },
  "fieldMap": {
    "fld90fZ6tK": {
      "rec0oA6okm": { "value": "{\"bus_type\":[201],\"data\":[\"🚨 已延期\"]}", "ext": "" },
      "rec5CX9BZX": { "value": "{\"bus_type\":[201],\"data\":[\"✅ 正常\"]}",  "ext": "" }
    }
  }
}
```

几点：

- **索引顺序是 `字段 → 记录`，与 `recordMap` 的 `记录 → 字段` 相反。** 公式是按列重算的，这个方向让"重算一列"变成替换一个子树。
- **值是二次序列化的 JSON 字符串**，内层带 `bus_type` 标记结果类型（201 = 文本，与 `fieldBusTypeMap` 一致）。公式返回类型是运行时确定的，所以要随值带类型。
- **`calcCompletionTime` 说明计算是异步的**，配套接口：

```
POST /space/api/bitable/fetch_calc_status/
{"baseToken":"G8Nbb0OW8aUwUCsHAZech1hFnnd","tableID":"tbl3iNXJ8JDbEHG9"}
```

—— 前端要轮询"这张表算完了没"。

计算模式的开关在 Base 上：

```json
"calcMode": 3,
"enableFmlDataOptimization": 1,
"baseEngineEnabled": false
```

`downgradeConfig` 里有一条规则直接引用了它们：

```json
{ "level": "tableNotSupport", "entity": "table",
  "match": { "conjunction": 0, "conditions": [
    { "property": "clientVars.baseJson.enableFmlDataOptimization", "method": "eq", "target": 1 },
    { "property": "clientVars.baseJson.calcMode", "method": "eq", "target": 3 } ] },
  "notify": { "type": "page",
              "config": { "content": "LarkCCM_Bitable_UnableToOpenUpgradeToLatestVersion_Desc_Mob" } } }
```

**满足这两个条件时，旧版客户端直接被拒绝打开，提示升级。** 因为老前端的本地公式求值器在这个模式下拿不到数据（`recordMap` 里是 null）。这是一次不向后兼容的架构切换，且切换开关下发给客户端自行判断。

### 7.1 三类字段的下发路径对比

| 字段类型 | 值在哪 | 谁算 |
| --- | --- | --- |
| 普通（文本/日期/单选/人员） | `recordMap[rec][fld].value` | 用户填，原样存取 |
| 关联（18 / 21） | `recordMap[rec][fld]` 的 `value` + `displayValue` | 服务端 join 出 `displayValue` |
| 公式（20） | `formulaInfo.fieldMap[fld][rec]`，`recordMap` 里为 null | 服务端异步引擎，带完成时间 + 轮询接口 |

递进很清楚：关联还是"值在原位、顺手补个展示名"；公式则彻底剥离成独立产物，有自己的生命周期（异步、可降级、要查状态）。

---

## 8. 数据是怎么传输的

### 8.1 首屏：一次请求换一张表的全部

```
GET /space/api/v1/bitable/{baseToken}/clientvars
      ?tableID=tbl3iNXJ8JDbEHG9
      &viewID=vewXxBNTOK
      &recordLimit=200        // 首屏行数
      &ondemandLimit=200
      &needBase=true          // 是否带 Base 骨架
      &viewLazyLoad=true
      &ondemandVer=2
      &noMissCS=true
      &optimizationFlag=1
      &removeFmlExtra=true    // 裁掉公式的 ext 字段
```

响应外层信封：

```json
{ "code": 0, "msg": "success", "data": {
  "type": "CLIENT_VARS",
  "loadingType": "rank_full",
  "encoding": 0,
  "base":  "H4sIAAAA...",     // base64(gzip(JSON))
  "table": "H4sIAAAA...",     // base64(gzip(JSON))
  "currentView": "vewXxBNTOK",
  "timeZone": "Asia/Shanghai",
  "timestamp": 1786526086,
  "permissions": [4, 1],
  "users": {}, "roomMembers": [], "syncTableIds": [],
  "costInfo": { ... }, "downgradeConfig": [ ... ],
  "calendar": null, "fallback": "", "calcDowngrade": "", "oldSchema": null,
  "extraInfo": "{\"weakErrCode\":0}"
} }
```

要点：

- **`base` 与 `table` 是两个独立的 base64(gzip(JSON)) 字符串**，不是内联对象。实测 21KB 的响应解开后是 2.1KB base + 29KB table —— HTTP 层已经有 `content-encoding: br`，这里是**第二层压缩**。这么做的收益是这两块可以整块缓存、整块比对、按需解压（`needBase=false` 时不下发 base）。
- **`encoding: 0` 是编码标记**，说明还有别的编码方案，客户端要按值分支解码。
- **`loadingType: "rank_full"`** 说明首屏策略是可切换的（按 rank 全量）。
- `extraInfo` 是**字符串化的 JSON**，与 `formulaInfo` 里的二次序列化是同一种做法 —— 这些块在服务端可能来自不同的服务，透传而不解析。

### 8.2 翻页：另一个接口，同一套结构

```
GET /space/api/v1/bitable/{baseToken}/records
      ?tableId=tbl3iNXJ8JDbEHG9&viewId=vewXxBNTOK
      &tableRev=10&depRev={}
      &offset=0&limit=5
      &viewLazyLoad=true&removeFmlExtra=true
```

响应（同样是 base64+gzip）：

```json
{ "tableID": "tbl3iNXJ8JDbEHG9", "tableRev": 10, "depRev": "{}", "jointRev": 26,
  "groupList": [ { "recordIDList": ["rec0oA6okm", ... 5 个 ...], "firstRecordOffset": 0 } ],
  "tableRecordNum": 13, "viewRecordNum": 0,
  "recordMap": { ... }, "recordMeta": { ... }, "rankInfo": { ... },
  "formulaInfo": { ... }, "resourceMap": { ... }, "fallBack": ... }
```

**翻页返回的是首屏结构的子集**：只有行相关的部分（`recordMap`/`recordMeta`/`rankInfo`/`groupList`/`formulaInfo`），没有 `fieldMap`/`viewMap`。schema 只下发一次。

请求里带 `tableRev=10` —— **客户端把自己知道的表版本回传**，服务端据此判断能否安全增量下发（若期间 schema 变了，可以返回 fallback）。响应里也回带 `tableRev`/`jointRev`/`depRev` 供校验。

### 8.3 增量同步：长轮询 + 房间

抓到的持续性请求：

```
POST /space/api/room/watch?member_id=20695740228913     // 长轮询，页面存活期间反复发起
POST /space/api/rce/heartbeat?member_id=...             // 心跳
POST /space/api/rce/messages?member_id=...              // 拉消息
GET  /space/api/rce/member_list?member_id=...&token=...&obj_type=8&protocol_version=2
POST /space/api/pandora_ws/ws_ticket/                   // WebSocket 票据
```

`member_id` 每次刷新都变（实测 `57667922919827` → `26248517276484` → `20695740228913`），是**会话级临时 ID**，不是用户 ID。

表数据结构里对应的锚点：`cs`（实测 `[]`）与 `latestCSRev`（实测 10，等于 `meta.rev`）。`cs` 应是 changeset 队列，`latestCSRev` 是客户端已消费到的版本 —— 但**本次没有抓到实际的 changeset 报文**（调研期间没有第二个用户在编辑），其具体形状不下断言（第 10 节）。

### 8.4 一次页面加载的完整请求序列

按抓到的时序（只列 Base 相关，剔除埋点/合规探测）：

```
1. clientvars?tableID=tbl3iNXJ8JDbEHG9&viewID=vewXxBNTOK&needBase=true   ← 主表 + Base
2. permission/user_role                                                  ← 权限
3. ssr_cache_info / ssr_data / base/ssr/header / base/csr/config          ← SSR 与前端配置
4. block_info?table=... / view_type?table=...                            ← block 类型
5. clientvars?tableID=tblddtXJIo2katnD&viewID=      ← 单向关联表（整表）
6. clientvars?tableID=tblLBe1aYHhczZw6&viewID=      ← 双向关联表（整表）
7. fetch_calc_status/                               ← 公式算完了没
8. records?...&offset=12&limit=3000                 ← 首屏 200 行之后的剩余行
9. datasource/list · connector/plugins              ← 外部数据源（实测为空）
10. room/watch · rce/heartbeat · ws_ticket          ← 协同通道
```

**加载单元是"表"而不是"Base"**：Base 有 3 张表就发 3 次 clientvars。仪表盘、文档、自动化在这个序列里完全没出现 —— 它们只在用户切过去时才加载。

---

## 9. 仪表盘是完全独立的一套

切到仪表盘 block 时，请求路径变成了另一个家族 —— 不是 `/bitable/*` 而是 `/sheet/blocks/*`：

```
GET  /space/api/v1/bitable/{baseToken}/clientvars?tableID=blkc2e5swqxonarH&...  ← 仍走 clientvars 拿骨架
GET  /space/api/v2/sheet/blocks/dashboard/web_ssr/charts?token={baseToken}&tableID=blkc2e5swqxonarH
GET  /space/api/v2/sheet/blocks/dashboard/global_filter/datasource?dashboardToken=dbdcn5Tmus5KHfdsRLsyXpMw2fe
POST /space/api/v3/sheet/blocks/chart/batch_get_data
POST /space/api/v2/sheet/blocks/chart/snapshot
```

**`/sheet/` 这个前缀说明仪表盘复用的是电子表格的图表基建**，不是多维表原生的东西。

### 9.1 仪表盘是一棵 widget 树

`web_ssr/charts` 返回的 `dashboard_clientVars`（又是字符串化的 JSON，且嵌套了三层 `data`）解开后：

```json
{ "rootWidgetId": "rootWidget",
  "filterInfos": [],
  "theme": { "themeStyle": "energetic",
             "backgroundSettings": { "selectType": "color", "color": "", "imageToken": "" } },
  "map": { /* 14 个 widget */ },
  "reaction": { /* 联动关系 */ } }
```

`map` 里 14 个 widget，扁平存储 + `parentId` 建树：

| type | typeV2 | chartType | name |
| --- | --- | --- | --- |
| CHART | — | `statistic` | 任务总数 |
| CHART | — | `statistic` | 延期任务数 |
| CHART | — | `aggregate_bar` | 任务负责人看板 |
| CHART | — | `aggregate_column` | 任务紧急程度分布 |
| CHART | — | `funnel` | 任务进展看板 |
| CHART | CHART | `ranking` | 任务数排行 |
| CHART | CHART | `countdown` | （无名） |
| CHART | AI_CHART | `ai_chart` | 任务流向分析桑基图 |
| CHART | AI_CHART | `ai_chart` | 进展与紧急程度分布热力图 |
| CHART | TABLE_VIEW | — | 关键任务看板 |
| CHART | SLICER | `slicer` | 切片器 ×3 |
| LAYOUT | — | — | （容器） |

可以直接读出的几件事：

- **`type` / `typeV2` / `chartType` 三个类型字段并存且不正交。** 切片器和表格视图既是 `type: CHART` 又有 `typeV2: SLICER` / `TABLE_VIEW`；有些 widget 有 `typeV2` 有些没有。这是**类型系统演进留下的地层**：早期只有 `type`（CHART/LAYOUT）+ `chartType`，后来加了 `typeV2` 来容纳"不是图表的图表"（切片器是筛选控件、TABLE_VIEW 是嵌入的表格）。
- **每个 widget 只存 `token`（`cht...`）**，图表定义本身不在这里，要再发 `batch_get_data`。三层：仪表盘 → widget → 图表定义。
- **`size` 只在部分 widget 上有**（`{h: 78, w: 653.625}` / `{h: 2, w: 3}`），且单位明显不同（一个像素一个格）。布局信息不完整/不统一，多数 widget 靠别处（未抓到）决定位置。

### 9.2 图表定义：数据条件 + 视觉模型分离

`POST /v3/sheet/blocks/chart/batch_get_data`，请求体极简：

```json
{"tokens":["chtcnz5ODmUvjx87se0Gb0ShUKd"]}
```

响应里 `snapshot.snapshot` 又是字符串化 JSON，解开后结构清晰：

```json
{
  "dataSources": [ { "rangeId": "_nj2q6su2y", "rangeDefinition": "<又一层字符串化 JSON>" } ],
  "viewModel": { "type": "biViewModel", "chartKind": 1049857, "rules": { ... } },
  "overrideViewModel": { ... },
  "tableView": null
}
```

`rangeDefinition` 解开后是**查询定义**：

```json
{ "refMap": { "#ref1": "tbl3iNXJ8JDbEHG9" },
  "dataCondition": {
    "tableId": "#ref1",
    "seriesArray": "COUNTA",
    "group": [
      { "fieldId": "fld9cvGzic", "mode": "integrated", "sort": { "sortType": "GROUP" } },
      { "fieldId": "fld90fZ6tK", "mode": "integrated", "sort": { "order": 1, "sortType": "GROUP" } }
    ],
    "source": { "type": "ALL", "filterInfo": null },
    "includeRecordIds": false,
    "includeArchiveTable": false,
    "extraConfig": { "ranking": { "limitSize": 10 } }
  } }
```

这是一个标准的 `GROUP BY 重要紧急程度, 是否延期` + `COUNTA` 聚合查询。几个观察：

- **表 ID 走 `refMap` 间接引用**（`#ref1`），不是直接写 `tbl3iNXJ8JDbEHG9`。这为多数据源图表（`isMultiDataSource`）留了位置。
- **`group` 里可以放公式字段**（`fld90fZ6tK` 是否延期）—— 说明聚合发生在公式求值之后。
- **`source.type: "ALL"` vs `"CUSTOM"`**：实测切片器用的是 `CUSTOM`，普通图表用 `ALL`。这是"取全表还是取某个视图/筛选结果"的开关。
- **`chartKind` 是位标记式的大整数**（`1049857`、`4194304`、`2147483948`、`1073741825`），不是小枚举。

### 9.3 聚合结果是二维表格，不是对象数组

```json
"data": { "rangeData": "[
  [ {\"value\":\"重要紧急程度\",\"text\":\"重要紧急程度\",\"groupKey\":null},
    {\"value\":\"✅ 正常\",\"text\":\"✅ 正常\",\"groupKey\":\"✅ 正常\"},
    {\"value\":\"🚨 已延期\",\"text\":\"🚨 已延期\",\"groupKey\":\"🚨 已延期\"} ],
  [ {\"value\":\"重要紧急\",\"text\":\"重要紧急\",\"groupKey\":\"opt0dhXwYV\"},
    {\"value\":3,\"text\":\"3\",\"formatString\":\"\",\"groupKey\":null},
    {\"value\":2,\"text\":\"2\",\"formatString\":\"\",\"groupKey\":null} ],
  [ {\"value\":\"紧急不重要\",...,\"groupKey\":\"optZhOuZUw\"}, {\"value\":1,...}, {\"value\":1,...} ],
  ...
]" }
```

**行列交叉的二维数组**，第一行是列头，第一列是行头 —— 就是一张透视表。每个格子是 `{value, text, groupKey}`：

- `value` 是原始值（数字 `3`），`text` 是格式化后的显示串（`"3"`）；
- `groupKey` 是分组的**稳定标识**：行头的 groupKey 是 optionId（`opt0dhXwYV`），列头是公式的输出文本（`"✅ 正常"`，公式没有 ID 可用）。

这个设计让图表库拿到的是可直接绘制的矩阵，同时保留了「点击某一格能反查回是哪个选项」的能力。

统计类图表更简单：

```json
// 任务总数
"[[{\"value\":\"Bitable_Dashboard_Count\",\"i18nKey\":\"Bitable_Dashboard_Count\",...}],
  [{\"value\":13,\"text\":\"13\",\"groupKey\":null}]]"
```

同样是二维数组（1 列 × 2 行），列头带 `i18nKey` 走客户端多语言。**统计值和柱状图用的是同一种载荷格式**，只是形状退化。

切片器返回的不是聚合而是候选值集合：

```json
"{\"valueList\":[
   {\"text\":\"已停滞\",\"value\":\"opt4yJuV43\",\"attribute\":{\"color\":32,\"name\":\"已停滞\"}},
   {\"text\":\"待开始\",\"value\":\"opth4KnfEy\",\"attribute\":{\"color\":11,\"name\":\"待开始\"}}, ...],
 \"limit\":{\"exceeded\":false},
 \"extra\":{\"fieldType\":3,\"fieldUIType\":3}}"
```

—— 走同一个 `batch_get_data` 接口，但载荷是另一种形状（`valueList` 而非 `rangeData`）。**接口统一、载荷按 widget 类型分支。**

### 9.4 联动关系单独一张表

```json
"reaction": {
  "chtcnNq4j9A9DRD59b4C4nQuQUd": {
    "type": "SLICER",
    "scope": "SERVER_CALC",
    "applyOn": "ALL",
    "applyList": [ { "id": "lVBGdmIHa1fSUeZTmo2c", "token": "chtcnJbOEgGGcphel70wb6tqmLe" },
                   { "id": "5KbrgBahsfMdZOURWzIf", "token": "chtcnz5ODmUvjx87se0Gb0ShUKd" }, ... ],
    "applySlicerList": [],
    "filterInfos": [ { "tableId": "tbl3iNXJ8JDbEHG9", "filterInfo": null } ]
  }
}
```

**"这个切片器影响哪些图表"是一份显式的邻接表**，不是靠 widget 嵌套关系推导的。`scope: "SERVER_CALC"` 表明联动后的重算在服务端做 —— 与第 7 节的公式一致，仪表盘也是"前端只收结果"。

---

## 10. 未能从报文确认的部分

以下问题在本次抓包中没有得到直接证据，列出以免与已确认结论混淆：

| # | 问题 | 现状 |
| --- | --- | --- |
| 1 | 服务端下发的分组结果长什么样 | 首屏走 `loadingType: "rank_full"`，`groupList` 只有一个 `by: null` 的扁平组。存在按组下发的 loadingType（`by` 字段的存在暗示了），但未触发 |
| 2 | changeset（`cs` / `latestCSRev`）的实际形状 | 调研期间无并发编辑，`cs` 始终为 `[]` |
| 3 | 关联表超过 200 行时交互层怎么兜底 | 关联表只有 5 行，未触发分页/搜索路径 |
| 4 | 写入路径（改单元格 / 建字段 / 建视图）的报文 | 本次只观察了读路径 |
| 5 | 仪表盘 widget 的完整布局信息 | `map` 里只有部分 widget 有 `size`，位置信息应在别处 |
| 6 | `permissions: [4, 1]` 的具体含义 | 是数字数组，未见解码依据 |
| 7 | `chartKind` 大整数的位定义 | 观察到 4 个值，不足以反推编码规则 |
| 8 | 更多字段类型（附件、地理位置、汇总、查找引用等） | 本 Base 未使用，未抓到 |

---

## 附：本次调研涉及的接口清单

| 接口 | 方法 | 用途 |
| --- | --- | --- |
| `/space/api/v1/bitable/{base}/clientvars` | GET | Base 骨架 + 单表 schema + 首屏行（核心） |
| `/space/api/v1/bitable/{base}/records` | GET | 行数据翻页 |
| `/space/api/bitable/fetch_calc_status/` | POST | 查公式是否算完 |
| `/space/api/bitable/{base}/block_info` | GET | block 类型 |
| `/space/api/bitable/{base}/view_type` | GET | 视图类型 |
| `/space/api/bitable/{base}/permission/user_role` | GET | 角色 |
| `/space/api/bitable/views/` | POST | 视图增量（返回 `gzipViews`） |
| `/space/api/bitable/datasource/list` | POST | 外部数据源（本 Base 为空） |
| `/space/api/v2/sheet/blocks/dashboard/web_ssr/charts` | GET | 仪表盘 widget 树 |
| `/space/api/v3/sheet/blocks/chart/batch_get_data` | POST | 图表定义 + 聚合结果 |
| `/space/api/v2/sheet/blocks/dashboard/global_filter/datasource` | GET | 全局筛选数据源 |
| `/space/api/room/watch` · `/rce/heartbeat` · `/pandora_ws/ws_ticket/` | POST | 协同通道 |
