# SchemaDefinition 设计

> 状态：草案
> 日期：2026-08-12

`SchemaDefinition` 是一个多维表应用的完整声明：有哪些表、每张表有哪些列、数据怎么被查看、怎么被汇总成图表。它是唯一的结构真相 —— 运行时、渲染层、查询层都从它派生，没有第二份可编辑的表示。

## 1. 范围

| 进 | 不进 |
| --- | --- |
| tables（表与字段） | 自动化规则 |
| views（视图） | 权限 / 鉴权 |
| dashboards（仪表盘） | 持久化与编辑协议 |
| 值模型（与结构**分开声明**） | 记录本身 |

三种定义各自声明什么：

| 定义 | 声明的是 | 运行时对应什么 |
| --- | --- | --- |
| `Table` | 一张表有哪些列、每列什么类型、什么约束 | 可写的记录集合，一行 = 一个业务实体 |
| `View` | 一个查询：取哪张表、怎么筛选排序、附带哪些算出来的列、怎么展示 | 一批行；来自表的列可写，算出来的列只读 |
| `Dashboard` | 一组聚合查询，以及它们绑定到哪种图表 | 统计结果，纯读 |

**`SchemaDefinition` 里只有中间那一列。** 记录本身不在其中 —— 右边一列只是为了说清每种定义在跑起来之后是什么，不属于本文档描述的对象。

---

## 2. 总原则：与关系型数据库一一对应

> **关系型数据库在 `CREATE TABLE` 里定义的，就放进 `Table`；不在那里定义的，就不放。**

| `CREATE TABLE` 里能写的 | 定义里的对应物 |
| --- | --- |
| 列名、类型 | `kind` + `format` |
| `NOT NULL` | `required` |
| `UNIQUE` | `unique` |
| `DEFAULT` | `default` |
| `CHECK` | `minimum` / `maximum` / 格式约束 |
| `PRIMARY KEY` | `primaryField` |
| `FOREIGN KEY … ON DELETE` | `relation` + `onDelete` |
| `GENERATED ALWAYS AS (表达式)` | `formula` |

`JOIN` 和聚合**不在** `CREATE TABLE` 里 —— 它们属于 `SELECT` / `CREATE VIEW`，因此归视图。

这条原则直接定下了一件本来会反复扯皮的事：

- **沿关联取对端的列、沿关联汇总子行，都归视图**（第 8 节）。

### 一一对应对应到的是关系型模型，不必是单个列

关联是这条原则唯一要点破的地方：一个关联列对应的**不是**一个 `CREATE TABLE` 里的外键列，而是一张 `(本行 id, 目标行 id)` 的关联表 —— 它拥有的链接边集。这不违反一一对应，只是对应物从「一列」换成「一张关联表」。

正因为对应到的是边集，「参与人这一列可以多选」这种业务用户期待的直接交互就**原生成立**：关联列声明 `multiple`，链接作为边存在数据层（见第 5 节），一格连多行只是边集上多几条边，不是要绕开的形态。

**代价写在明处：** 一个关联字段不再对应单个列，而对应一张关联表 —— 校验、查询、迁移都得按边集来处理。换来的是多值关联在定义+数据层就地成立，不必让工具去合成一张用户可见的中间表，也不必为迁就单值外键而扭曲交互。后者的补丁会摊到每一处，只在字段背后藏一张关联表则只花一次。

---

## 3. 结构与值分两层

`SchemaDefinition` 只描述结构，记录不在其中。值模型独立声明，两者在类型上关联（字段 kind → 值的形状），存储上分离。

```ts
interface SchemaDefinition {
  specFormat: 'schema/v1'
  meta:       Meta
  tables:     Record<Id, Table>
  views:      Record<Id, View>
  dashboards: Record<Id, Dashboard>
}

interface Meta {
  name: string                    // 应用名，如「洛书」
  description?: string
}

interface Table {
  name: string
  primaryField: Id                // 主字段：一行的展示标题，见第 2 节
  fields: Record<Id, Field>
}
```

四个顶级 map 的 key 都是 `Id`：一处声明用它，别处引用也用它（视图的 `source.table`、关联的 `target`、图表的通道，全是这些 id）。`Table` 只包字段；视图、仪表盘各自成表，见 §8.2 / 第 9 / 10 节。

### 3.1 一个单元格能存什么

```ts
type GeoPoint  = { lat: number; lng: number; address?: string }
type CellValue = string | number | boolean | null | readonly string[] | GeoPoint
```

多值列（多选、多人、附件、多值关联）统一用 `readonly string[]`，元素是 id。单值关联存一个目标行 id（`Id | null`），多值关联存一组（`readonly Id[]`）—— 关联的链接是数据层的边，格子里放的就是这些边指向的目标行 id，见第 5 节。

`GeoPoint` 是唯一的结构形状 —— 坐标不序列化成字符串塞进文本列，因为值模型本来就要单独定义，为一个字段类型多加一种形状是划算的。

### 3.2 字段类型与值的对应

```ts
type ValueOf<F extends Field> =
  F extends TextField     ? string | null :
  F extends NumberField   ? number | null :
  F extends SelectField   ? (F['multiple'] extends true ? readonly Id[] : Id | null) :
  F extends RelationField ? (F['multiple'] extends true ? readonly Id[] : Id | null) :
  F extends GeoField      ? GeoPoint | null :
  /* … */ never
```

---

## 4. 字段

### 4.1 九个存储 kind

`text · number · boolean · date · select · user · attachment · geo · relation`

刻意保持窄。「电话」「邮箱」「条码」「评分」「进度」「货币」这些**不是独立的类型**，它们是 text 或 number 加一层格式约束 —— 让它们各占一个顶级类型，会把类型数量推到二十开外，而每个新类型都要在校验、渲染、查询、值定义四处各写一遍。

### 4.2 格式约束往下放一层

格式各自的参数跟着格式走，不上浮到字段顶层：

```ts
type NumberFormat =
  | { kind: 'plain';    precision?: number }
  | { kind: 'percent';  precision?: number }
  | { kind: 'currency'; currencyCode: string; precision?: number }   // ISO 4217
  | { kind: 'rating';   max: number }
  | { kind: 'progress' }                                             // 0–100

interface NumberField extends WritableFieldBase {
  kind: 'number'
  format?: NumberFormat        // 省略 = plain
  minimum?: number
  maximum?: number
  default?: number
}

type TextFormat =
  | { kind: 'plain' }
  | { kind: 'richText' }
  | { kind: 'email' }
  | { kind: 'phone' }
  | { kind: 'barcode' }
  | { kind: 'url'; allowedSchemes?: readonly ('https' | 'http' | 'mailto')[] }
```

`currencyCode` 只有货币格式才有，`max` 只有评分才有 —— 放进各自的分支，就不会出现「一个数字列同时挂着货币代码和评分上限」这种配得出来但没意义的组合。

同样的收敛用在另外两处：单选与多选合并为 `select` + `multiple?`；人员与群组合并为 `user` + `subject: 'member' | 'group'` —— 两者结构相同，区别只是查的哪本目录。

### 4.3 每种 format 约束了什么

按第 2 节的原则，`format` 对应 SQL 的 **DOMAIN** —— 带名字的类型加 `CHECK`：

```sql
CREATE DOMAIN email AS text CHECK (VALUE ~ '…');
```

所以 `format` 是**选一个内置 DOMAIN**，不是写一条规则。规则由引擎内置，不逐列携带正则 —— 否则每个邮箱列各存一份正则，写错一个就有一列的校验和别处不一样，正则方言还有可移植性问题。

| format | 约束 | 对应 SQL |
| --- | --- | --- |
| `text / plain` | 无 | `text` |
| `text / richText` | 值是富文本标记而非纯文本 | 语义差别，无 `CHECK` |
| `text / email` | 必须是合法邮箱 | `CHECK` |
| `text / phone` | 必须是合法电话号 | `CHECK`（宽松） |
| `text / url` | scheme 必须在 `allowedSchemes` 内 | `CHECK` |
| `text / barcode` | 必须是所选码制的合法编码 | `CHECK` |
| `number / plain` | 无 | `numeric` |
| `number / percent` | 值是比例（`1` = 100%） | 单位约定，无 `CHECK` |
| `number / currency` | 金额，单位是 `currencyCode` | `numeric(_, precision)` |
| `number / rating` | 整数且 `0 ≤ 值 ≤ max` | `CHECK` |
| `number / progress` | `0 ≤ 值 ≤ 100` | `CHECK` |

字段顶层的 `minimum` / `maximum` 同样是 `CHECK`，区别只是它跨所有数字格式通用。

**`precision` 在两处都出现，含义不同：**

| 位置 | 含义 | 影响 |
| --- | --- | --- |
| `Table` 的 `format.precision` | **存储精度** —— 小数存几位 | `numeric(_, n)` 的 scale，会截断真实数据 |
| `View` 的 `presentation` | **显示位数** —— 界面上呈现几位 | 只影响呈现，不动存储 |

同一张表的两个视图可以显示不同位数，但存储精度只有一份。

**暂不支持自定义格式。** 「工号必须是 `EMP-` 加六位数字」这类需求就是 `CREATE DOMAIN`，加起来不难（`SchemaDefinition` 里开一个 `domains` map 供各列引用），也不违反第 2 节的原则，但它是个新概念，等真有需要再说。

### 4.4 系统列

值由引擎在写入时生成，用户不可编辑，无配置项：

```ts
interface SystemField extends FieldBase {
  kind: 'createdAt' | 'updatedAt' | 'createdBy' | 'updatedBy' | 'autoNumber'
}
```

它们与算出来的列不同：系统列的值来自**写入这个动作本身**，写的那一刻定死，丢了不可重建；算出来的列的值来自**别的数据**，随时可以重算。

### 4.5 基接口分层

基类只放**跨类型通用**的约束；格式类的约束是各 kind 自己的事，挂在各自的 `format` 上（见 4.3）。

`required`（必填）和 `unique`（不重复）是**对用户输入的约束** —— 前提是这一格的值由人填。它们因此不属于所有字段的公共基类，而属于「可写」这一层：

```ts
interface FieldBase         { name: string; description?: string; deletedAt?: number }
interface WritableFieldBase extends FieldBase { required?: true; unique?: true }

interface TextField     extends WritableFieldBase { kind: 'text';     /* … */ }
interface RelationField extends WritableFieldBase { kind: 'relation'; /* … */ }
interface FormulaField  extends FieldBase         { kind: 'formula';  /* … */ }
interface SystemField   extends FieldBase         { kind: 'createdAt' | /* … */ }
```

基类**窄，按需往上加**。加法可以嵌套复用，减法不行 —— 一旦某个子类型需要「去掉」基类的属性，就说明基类设宽了，而且减错或漏减编译器不会提醒。

### 4.6 「能不能写」这条线由编译器守着

这个判断会被下游反复用到（写入校验、表单渲染、API 入参）。用一张必须写全的登记表来记，而不是散落各处的类型判断：

```ts
const WRITABLE_KINDS: Record<WritableField['kind'], true> = {
  text: true, number: true, boolean: true, date: true, select: true,
  user: true, attachment: true, geo: true, relation: true,
}

export function isWritable(f: Field): f is WritableField {
  return f.kind in WRITABLE_KINDS
}
```

`Record<WritableField['kind'], true>` 要求把联合类型的每个成员都列出来：新增一个可写 kind 却忘了登记 → 编译报错；把不可写的 kind 误登记进来 → 编译报错。判断只此一处，其余地方一律调 `isWritable`。

> **为什么不把字段按可写性拆成多张 map。** 分表能让「不可写的列出现在可写位置」从「能检查出来」变成「根本写不出来」，但换不到任何类型推导 —— schema 是运行时从 JSON 载入的，`Table` 不是字面量类型，`Record<Id, Field>` 里哪个 key 对应哪个变体，在类型层本来就不存在。代价却是实打实的：编辑协议要先判断列属于哪张 map，改列类型如果跨 map 就得表达成「从一张搬到另一张」而不是原地改，还要额外校验两张 map 之间 id 不重复。上面那张必须写全的登记表拿到了绝大部分收益，且不动协议。

---

## 5. 关联

关联指的是**指向本应用另一张表的行**（一行或多行）。`select` / `user` / `attachment` 的格子里也存 id，但它们指向的是选项集、人员目录、附件，不是表的行，因此不属于本节。

```ts
interface RelationField extends WritableFieldBase {
  kind: 'relation'
  target: Id                                      // 目标表；允许自引用
  multiple?: true                                 // 一格可连多行；省略 = 至多一行
  onDelete: 'restrict' | 'cascade' | 'setNull'    // 目标行被删时，指向它的边怎么办
}
```

**关联列只声明「这一列连向哪张表」（`target`）；具体哪一行连哪一行是数据，不是结构。** 列定义里不含任何一格的链接 —— 那些 `(本行 → 目标行)` 的链接作为一组边存在数据层，由这一列拥有。

于是「一格连一行」和「一格连多行」不是两种结构，只是这组边上的一条基数约束：`multiple` 控制一个本行能不能连多个目标，`unique`（继承自 `WritableFieldBase`）控制一个目标能不能被多个本行共享。两个开关正交，四种基数全覆盖（见 5.1）—— 就像 select 用 `multiple?` 区分单选多选（§4.2），不为多值另开一种字段。

`onDelete` 作用在边上：目标行被删时，`restrict` 禁止删除、`cascade` 连本行一并删、`setNull` 只删掉这条边（多值即从那一组 id 里去掉一个，单值即变 `null`）。

这仍然一一对应到关系型模型：边集就是那张 `(本行 id, 目标行 id)` 关联表，`multiple` / `unique` 对应它两列上的唯一约束。区别只是这张关联表由关联列拥有、隐式存在，不占用户界面上的一张表。

### 5.1 四种基数怎么表达

两个正交开关就够了：`multiple` 管**本行这端能连几个目标**，`unique` 管**一个目标能被几个本行连**。

| 想要的关系 | multiple | unique | 一个本行 → | 一个目标 ← |
| --- | --- | --- | --- | --- |
| 多对一 | — | — | 至多一个目标 | 可被多个本行共享 |
| 一对一 | — | ✓ | 至多一个目标 | 至多一个本行 |
| 一对多 | ✓ | ✓ | 可连多个目标 | 只属一个本行 |
| 多对多 | ✓ | — | 可连多个目标 | 可被多个本行共享 |

一对一、一对多复用已有的 `unique`，不新开开关 —— 这正是关系型数据库在关联表两列上加唯一约束的做法。

一对多现在**从任意一侧都能声明**：边集是对称的，从「多」那侧声明一个多对一（默认），或从「一」那侧声明一个 `multiple + unique`，落到的是同一张边集。旧的「一对多只能从『多』那侧」是内联外键才有的限制 —— 链接进了数据层，这条限制没了。多对多也不再要用户手建一张中间表，声明一个 `multiple` 关联即可，中间表由这一列隐式拥有。

### 5.2 关系只声明一次

一条关联只在声明这一列的那张表上声明，目标表上不为它另立一个反向列。边集本来就是对称的，从哪端读都是同一组边 —— **逆向方向不是一个需要声明的对象**，它是这条关联白拿的另一半（见 5.3），由查询层按需走出来：视图汇总子行时，`via` 引用的就是这条 relation 的 id（见 8.2），方向由「站在目标表一侧回看」自然确定。

给反向另立一条声明换不到任何东西，却要付代价：「建一个双向关联」就得拆成两侧各一条，两次提交之间会出现「反向端指向一个不存在的列」这种不合法的中间状态。**一条关联只声明一次。**

### 5.3 走哪个方向，决定是一行还是一堆行

沿一条边从任一端走过去，可能落到一行，也可能落到一堆行，由那一端的基数约束定：

- **从本行走向目标**：`multiple` → 一堆行；默认 → 至多一行。
- **从目标回看本行**：`unique` → 至多一行；否则 → 一堆行。

走过去只有一行的方向称 **to-one**，是一堆行的称 **to-many**。以「任务 → 所属项目」（默认单值）为例：正向 任务 → 项目 是 to-one，反向 项目 → 任务 是 to-many。

于是每条关联都同时给出一个正向和一个反向，各自是 to-one 或 to-many。to-many 不是边角情况，它往往正是业务想看的那半：「这个项目有几个任务」「这个客户下了多少单」。**注意 `multiple` 关联的正向本身就是 to-many** —— 把一格里的多个链接铺开，走的就是 5.5 说的那条汇总 / 列举路径。

### 5.4 走一跳，行数会不会变

- **to-one 走一跳，行数不变。** 可以任意叠加，也可以连着走多跳（任务 → 项目 → 客户 → 客户经理），每跳都不改变行数。
- **to-many 走一跳，行数会变。** 一个有 3 个任务的项目会变成 3 行。

### 5.5 来自 to-many 的值必须先汇总

沿 to-many 走一跳得到的是一堆行。把它接回原来的行集有两种接法，它们回答的是**两个不同的问题**。

**join —— 行数变多。** 项目表的行集 join 任务表，一个有 3 个任务的项目变成 3 行，得到的是「任务列表，每行带着它所属项目的信息」。这正是要 join 的人想要的东西。

**汇总 —— 行数不变，对每一行分别汇总它自己的子行。** SQL 里写成相关子查询，不是 join：

```sql
SELECT p.*, (SELECT COUNT(*) FROM task t WHERE t.project_id = p.id) FROM project p
```

3 行进、3 行出，得到的是「项目列表，每行带着它的任务数」。

关键在于**后者不能由前者加工得到**：join 之后，一行代表的已经是一个任务而不是一个项目，项目自己的每一列在那 3 行里各出现一次；想变回「一行一个项目」，只能再汇总一次 —— 那不如一开始就汇总。

所以「在项目这一行上显示一个来自任务表的值」，本身就是一次汇总，不是 join 的后处理。视图因此要把 join 和汇总分开声明，见 8.2。

---

## 6. 查询语言

一套写法，视图、仪表盘、汇总列共用。

### 6.1 Filter

```ts
interface FilterOperators {
  $eq?: FilterLiteral;  $ne?: FilterLiteral
  $in?: readonly FilterLiteral[];  $nin?: readonly FilterLiteral[]
  $gt?: FilterLiteral;  $gte?: FilterLiteral
  $lt?: FilterLiteral;  $lte?: FilterLiteral
  $exists?: boolean
  $contains?: FilterLiteral;  $startsWith?: string
}

type FilterCondition = FilterOperators | FilterLiteral   // 直接给值是简写：{ 状态: '已完成' }

type Filter =
  | { $and: readonly Filter[] }
  | { $or:  readonly Filter[] }
  | { $not: Filter }
  | { [fieldId: string]: FilterCondition }
```

### 6.2 「当前用户」「最近 N 天」是值，不是运算符

```ts
type FilterLiteral =
  | JsonPrimitive
  | { $me: true }                                              // 当前用户
  | { $now: true }
  | { $daysAgo: number }
  | { $periodStart: 'week' | 'month' | 'quarter' | 'year' }
```

另一种做法是给它开一个专用运算符，摆在 `$gt` / `$lt` 旁边：`{ createdAt: { $inLastDays: 7 } }`。不这么做，是因为那样只表达得出「最近 N 天」这一种比较；做成值则可以配任何运算符：

```ts
{ createdAt: { $gte: { $daysAgo: 7 } } }                        // 最近 7 天
{ createdAt: { $lt:  { $daysAgo: 30 } } }                       // 30 天没动过的
{ createdAt: { $gte: { $daysAgo: 30 }, $lt: { $daysAgo: 7 } } } // 7～30 天前
{ 截止日:    { $lt:  { $now: true } } }                          // 已经过期的
```

后三行若走运算符那条路，得各加一个新运算符。`{ $me: true }` 同理 —— 它是值，所以 `$eq` / `$ne` / `$in` 都能拿它比较。

### 6.3 汇总函数

```ts
type Aggregation = 'count' | 'distinctCount' | 'sum' | 'average' | 'min' | 'max' | 'list'
```

`list` 不合成一个数，而是把子行的值列出来（「这个项目下所有任务名」）。数值汇总和列举放在同一个枚举里 —— 它们是同一个动作（沿关联走到一堆行、收进一个格子）的不同收法，不该做成两套机制。

### 6.4 Query

```ts
interface Query {                      // 行级：视图用
  table: Id
  filter?: Filter
  sort?: readonly Sort[]
  limit?: number
}

interface Sort {
  field: Id                            // 存储列 / join / aggregate / computed 的 id 都行
  direction: 'asc' | 'desc'
}

interface AggregateQuery extends Query {   // 聚合级：仪表盘用
  dimensions?: readonly Dimension[]    // GROUP BY
  measures:    readonly Measure[]      // SELECT 里的聚合，至少一个
  having?:     readonly Having[]        // 对聚合结果再筛
}

interface Dimension {                  // 分组的一项
  id: Id                               // 供图表通道引用
  field: Id
  bucket?: 'day' | 'week' | 'month' | 'quarter' | 'year'   // 日期分桶；非日期列省略
}

interface Measure {                    // 一个聚合出来的数
  id: Id                               // 供图表通道 / having 引用
  aggregation: Aggregation             // 见 6.3
  field?: Id                           // count / distinctCount 可省略，其余必填
  filter?: Filter                      // 只统计满足条件的子集，如「已完成任务数」
}

interface Having {                     // SQL 的 HAVING：拿聚合结果比较
  measure: Id                          // 引用某个 measure 的 id
  condition: FilterOperators           // 复用 6.1 的运算符
}
```

`Dimension` / `Measure` 都带 `id`，因为仪表盘的图表要按 id 把它们接到坐标轴、系列、数值这些视觉通道上（第 10 节）。`having` 对着 `measure` 的 id 筛，正如 SQL 的 `HAVING` 只能碰聚合结果、不碰原始行。

---

## 7. 表 · 视图 · 仪表盘的分界

这一节比较的是**用户最终看到的东西**，不是定义本身 —— 但它决定一个算出来的值该做成表或视图的一列，还是做成仪表盘的一个指标。

| | 表（含其视图） | 仪表盘 |
| --- | --- | --- |
| 一行是什么 | 一个你要**动它**的业务实体 | 一个统计结果 |
| 用户在干嘛 | 逐行处理：改状态、指派、跟进 | 看对比、趋势、异常 |
| 可写 | 是 | 否 |
| 用多久 | 边看边改 | 看一眼就走 |

判断方法一句话：**有没有人对着这一行做动作。**

据此：「项目的任务数 / 完成率」是操作型 —— 项目负责人打开项目列表逐个跟进，要能排序、筛选、点进去，这是工作清单不是图表。「各客户的平均项目完成率」是分析型 —— 横向对比，看完不对某一行做动作。

跨多层的汇总（客户 ← 项目 ← 任务）由仪表盘的查询层承担，不作为表列设计的依据。

---

## 8. 算出来的列住在哪一层

按第 2 节的原则，这条线画在 `CREATE TABLE` 的边界上：

| 算什么 | 归属 | RDBMS 里是什么 |
| --- | --- | --- |
| 同一行内的表达式 | **表** | `GENERATED ALWAYS AS (expr)` |
| 取对端的一列 | **视图** | `JOIN` |
| 汇总子行 | **视图** | 相关子查询 |

### 8.1 表：generated column

```ts
interface FormulaField extends FieldBase {
  kind: 'formula'
  expression: string          // 只能引用同一行的存储列
}
```

SQL 的 generated column 也只能引用同一行 —— 作用范围天然一致，不必额外规定。它不可写，因此继承 `FieldBase` 而不是 `WritableFieldBase`（见 4.5），也就不会带上 `required` / `unique`。

### 8.2 视图：join / 汇总 / 表达式

```ts
interface View {
  name: string
  source: { table: Id } | { view: Id }                       // ← 见第 12 节 #1
  joins?:      Record<Id, { via: readonly Id[]; field: Id }>  // ① to-one，可多跳，行数不变
  aggregates?: Record<Id, { via: Id; field?: Id
                            aggregation: Aggregation; filter?: Filter }>  // ② to-many，逐行汇总，行数不变
  computed?:   Record<Id, { expression: string }>            // ③ 在 ①② 结果上算
  filter?: Filter
  sort?: readonly Sort[]
  presentation: Presentation                                 // 怎么展示这批行，见第 9 节
}
```

求值顺序固定为 **① join → ② 汇总 → ③ 表达式**，全程行数不变，始终一行一条源表记录。

**视图这一层也有表达式，它和表里的 generated column 不是一回事**：generated column 只能碰同一行的存储列；视图的表达式可以用 ① ② 的结果。「完成率 = 已完成数 ÷ 任务数」属于后者 —— 两个数都是汇总出来的，表那一层碰不到。

这个分工和 SQL 一致：表有 generated column，视图的 `SELECT` 列表里也可以写表达式。

### 8.3 这么切的代价

**同一个定义要写很多遍。** 一张表若有 5 个视图，前 4 个都要「完成率」，就各声明一遍 join + 汇总 + 表达式。定义变更要改多处，漏一处即两个页面显示不同的数。机器生成 spec 时这个成本更明显 —— 生成便宜，保持一致贵。

缓解手段是允许**视图以视图为源**：一张表配一个基础视图把定义写清楚，其余视图以它为源，只叠 filter / sort / 展示。待定，见第 12 节 #1。

**读接口以视图为单位。** join 和汇总出来的值只存在于视图输出里，直接查表拿不到「完成率」（表里能拿到的只有存储列、系统列和 generated column）。跨表引用这类值时，join 的目标也是视图而不是表。

### 8.4 已考虑并排除的形态

| 形态 | 排除理由 |
| --- | --- |
| 把取值与汇总也做成表的列 | `CREATE TABLE` 里没有 `JOIN` 和聚合，违反第 2 节的原则 |
| 表达式全部下沉到视图（表里不留 generated column） | `GENERATED ALWAYS AS` 在 `CREATE TABLE` 里，同样违反原则；而且同一行内的计算本来就不需要查询层 |
| 只提供 join + 表达式，不单设汇总声明 | join 到 to-many 之后，一行代表的是子行而不是源表的行；「一行一个项目」的汇总没法由它加工得到，见 5.5 |
| 一律用 `GROUP BY` 重建「一行一个项目」 | 语义上可行，但项目表本来就一行一个项目，从任务表分组把它重建一遍是绕路：项目自己的每一列都要额外声明怎么合并，而逐行汇总直接表达同一意图 |
| 字段按可写性拆成多张 map | 换不到类型推导，却明显复杂化编辑协议，见 4.6 |

---

## 9. 视图：查询之外，还要声明怎么展示

§8.2 的 `View` 把「取哪些行」说清楚了；剩下的是「这批行怎么摆在屏幕上」—— 这就是 `presentation`。它只管呈现，不改一行数据，也不改行数。

```ts
type Presentation =
  | GridPresentation
  | KanbanPresentation
  | CalendarPresentation
  | GalleryPresentation
  | FormPresentation

interface ColumnStyle {           // 单列的呈现覆盖；key = 字段 / join / aggregate / computed 的 id
  hidden?: true
  width?: number                  // 像素
  precision?: number              // 显示位数，见 4.3：只影响呈现，不动存储精度
}

interface GridPresentation {
  kind: 'grid'
  order?: readonly Id[]           // 列从左到右的顺序；省略 = 定义顺序
  columns?: Record<Id, ColumnStyle>
  frozen?: number                 // 冻结左侧几列
  rowHeight?: 'short' | 'medium' | 'tall'
}

interface KanbanPresentation {
  kind: 'kanban'
  groupBy: Id                     // 按哪一列分栏，通常是 select 或 relation
  order?: readonly Id[]           // 卡面显示哪些列、按什么顺序
  columns?: Record<Id, ColumnStyle>
}

interface CalendarPresentation {
  kind: 'calendar'
  startField: Id                  // 事件落在日历上用哪个 date 列
  endField?: Id                   // 有跨度的事件
  titleField?: Id
}

interface GalleryPresentation {
  kind: 'gallery'
  coverField?: Id                 // 用哪个 attachment 列做封面
  order?: readonly Id[]
  columns?: Record<Id, ColumnStyle>
}

interface FormPresentation {
  kind: 'form'
  items: readonly FormItem[]      // 只列可写字段；顺序即渲染顺序
  submitLabel?: string
}
interface FormItem {
  field: Id
  label?: string                  // 覆盖字段名
  help?: string                   // 填写说明
  required?: true                 // 表单级必填，叠加在字段级 required 之上
}
```

**呈现里凡是指列，都用同一套 id。** grid 的 `columns`、kanban 的 `groupBy`、日历的 `startField`，key 都是 §8.2 那些 id：存储列、join、aggregate、computed 一视同仁。算出来的列在呈现层和存储列没有区别 —— 这正是 §8.2 把它们都收进视图、给统一 id 的回报。

### 9.1 form 是视图的一种呈现，不是第四种顶级定义

原本悬而未决（旧「视图是否拆出录入面」）：form 没有 filter / sort，跟 grid / kanban 结构不同，要不要拆成单独的东西。结论是**不拆** —— form 仍然绑在一个源表上、仍然是「把这张表的字段摆出来」，只是摆法是一张录入卡而非一个行网格。`filter` / `sort` 在 `View` 上本来就是可选的，form 不设就是了。

代价写在明处：一张纯新建用的 form 不消费查询的行 —— 它的 `source` 只圈定往哪张表插、能编辑哪些既有行。为这一点差异另开一个 `forms` 顶级 map、多一个概念，换来的分离不值当；`presentation` 这个联合本来就在表达「同一个源，几种摆法」，form 就是其中一种。

---

## 10. 仪表盘

视图 = 查询 + 怎么摆行；**仪表盘的一个 widget = 聚合查询 + 怎么把聚合结果画成图**。结构上完全对称：把 §9 的「呈现」换成「图表通道绑定」。

```ts
interface Dashboard {
  name: string
  widgets: Record<Id, Widget>
  layout?: Record<Id, Box>        // 每个 widget 在 12 列网格里的位置；省略 = 按序堆叠
}
interface Box { x: number; y: number; w: number; h: number }

interface Widget {
  title?: string
  query: AggregateQuery           // 见 6.4：dimensions + measures (+ having)
  chart: Chart
}

type Chart =
  | { kind: 'metric'; value: Id }                                    // 0 维 1 值：一个大数字
  | { kind: 'bar' | 'line' | 'area'; x: Id; y: readonly Id[]; series?: Id }
  | { kind: 'pie' | 'donut'; category: Id; value: Id }
  | { kind: 'table' }                                                // 维度做行、度量做列，直接铺开
```

### 10.1 图表通道接的是 query 里的 id

`chart` 上的 `value` / `x` / `y` / `category` / `series` 全是 `id`，引用同一个 widget 的 `query` 里声明的 dimension 或 measure。哪个 id 接哪个通道，就定了这张图怎么画：

| kind | query 要求 | 通道 |
| --- | --- | --- |
| `metric` | 0 个 dimension，≥1 个 measure | `value` = 一个 measure |
| `bar` / `line` / `area` | ≥1 个 dimension | `x` = dimension；`series?` = 第二个 dimension（分系列）；`y` = 一到多个 measure |
| `pie` / `donut` | 1 个 dimension，1 个 measure | `category` = dimension；`value` = measure |
| `table` | 任意 | 无需绑定：dimension 顺次做前导列，measure 做后续列 |

这跟视图用同一套办法：**查询只负责算出 dimension 和 measure，怎么摆是另一层的事。** measure 换个聚合函数、dimension 换个分桶，图表通道不动；反过来 bar 改成 line 也不碰查询。二者不搅在一起，正是 §6.4 给 dimension / measure 各配一个 id 的用意。

### 10.2 为什么 widget 不直接引用视图

widget 自带一个 `AggregateQuery`，而不是指向某个已有 `View`。因为视图产出的是**行**（一行一条源记录），仪表盘要的是**聚合结果**（一行一个统计），两者查询形状不同（`AggregateQuery` 比 `Query` 多了 dimensions / measures / having）。让 widget 复用视图，就得允许「对视图再 GROUP BY」，那是又长一截的查询能力；直接给 widget 一个聚合查询一步到位，也和 §7「操作型 / 分析型」的分界一致：分析型的东西不必挂在操作型的视图上。

---

## 11. 一个完整的例子

[`examples/project-tracker.json`](examples/project-tracker.json) 是一份能跑通上面所有构件的 `SchemaDefinition`：客户 / 项目 / 任务三张表，四个视图（grid / kanban / grid / form），一个总览仪表盘。挑三段看它长什么样。

**表：任务，带一个多值自关联和一个 generated column**（§4 / §5）

```jsonc
"fld_task_project": { "kind": "relation", "name": "所属项目", "target": "tbl_project", "onDelete": "cascade" },
"fld_task_deps":    { "kind": "relation", "name": "前置任务", "target": "tbl_task", "multiple": true, "onDelete": "setNull" },
"fld_task_weight":  { "kind": "formula",  "name": "加权分", "expression": "fld_task_priority * (100 - fld_task_progress)" }
```

`所属项目` 是单值关联（一个任务一个项目）；`前置任务` 是 `multiple` 自关联（一个任务可依赖多个任务）—— 正是第 5 节那套「列声明端点、链接是数据层的边」。`加权分` 只碰同一行的列，是表级的 generated column。

**视图：项目清单，逐行汇总子任务再算完成率**（§8.2）

```jsonc
"aggregates": {
  "agg_task_count": { "via": "fld_task_project", "aggregation": "count" },
  "agg_task_done":  { "via": "fld_task_project", "aggregation": "count",
                      "filter": { "fld_task_status": "opt_done" } }
},
"computed": {
  "c_done_rate": { "expression": "agg_task_count > 0 ? agg_task_done / agg_task_count : 0" }
}
```

`via` 指向住在任务表上、指向项目的那条关联（`fld_task_project`）—— 从项目这侧逆着走，就是「这个项目的任务们」，逐行 count（§5.5）。`c_done_rate` 在两个汇总数之上再算，行数始终不变。

**仪表盘：一个 widget = 聚合查询 + 图表通道**（第 10 节）

```jsonc
"query": {
  "table": "tbl_project",
  "dimensions": [{ "id": "d_status", "field": "fld_proj_status" }],
  "measures":   [{ "id": "m_count", "aggregation": "count" }]
},
"chart": { "kind": "bar", "x": "d_status", "y": ["m_count"] }
```

查询按状态分组、数每组项目数；图表把 `d_status` 接到 x 轴、`m_count` 接到高度。换成 `{ "kind": "donut", "category": "d_status", "value": "m_count" }` 就是同一份数据的环形图 —— 查询一个字都不用改。

---

## 12. 未决问题

| # | 问题 | 状态 |
| --- | --- | --- |
| 1 | 视图能否以视图为源 | 倾向支持：这是 8.3 那个重复问题唯一干净的解法 |
| 2 | to-one join 是否支持多跳（`via: Id[]`） | 倾向支持：每跳行数不变，是白拿的表达力 |
| 3 | 读接口以视图为单位，对外部接口的影响面 | 需评估 |
| 4 | 编辑协议与持久化 | 本文不涉及 |
| 5 | 值模型（记录长什么样、null 语义、id 引用指向谁） | 另开一份文档 |

> 旧 #4（仪表盘 widget 模型）已在第 10 节展开，旧 #5（form 是否拆成独立定义）已在 §9.1 定为「不拆」。

---

## 附：本文用到的两个说法

| 说法 | 含义 |
| --- | --- |
| to-one / to-many | 沿一条关联边走过去落到一行还是一堆行，由那一端的 `multiple` / `unique` 定，见 5.3 |
| 操作型 / 分析型 | 见第 7 节，判断方法是「有没有人对着这一行做动作」 |
