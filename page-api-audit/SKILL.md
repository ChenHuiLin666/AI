---
name: page-api-audit
description: 扫描已拉到本地的页面元数据及 recruit-paas 组件源码，整理静态、动态接口、请求参数及直接依赖链路。用于页面爬接口、数据源分析和接口清单整理，不负责拉取页面。
---

# 页面接口扫描

## 前提与范围

页面由用户自行拉到本地。等用户通知拉取完成并要求扫描后再执行；只讨论规则或维护本 skill 时不要扫描。确定页面目录、本地 recruit-paas 仓库及本次迭代分支；优先从已有上下文和本地信息确定，无法确定再询问，不猜测分支。不修改业务代码，不擅自切换用户的工作分支，优先只读查看目标分支源码。

## 组件项目定位

- 已知 recruit-paas 仓库根目录：`/Users/chenhuilin/beisen/20264/recruit-paas`。
- 已知组件入口文件：`/Users/chenhuilin/beisen/20264/recruit-paas/src/index.tsx`。从该入口的导入、导出和映射追踪对应组件实现，不将入口本身视作全部组件源码。
- 仍需确认本次迭代分支；路径中的 `20264` 不等于分支名。
- 遇到无法确定属于哪个项目的组件时，立即暂停扫描，向用户说明组件包名、所属页面以及已检查的位置，询问该组件的项目或源码路径。用户答复前不要继续扫描其他页面、猜测所属项目或将该组件跳过后宣称完成。

- `@recruit-common/desensitize` 和 `@recruit-common/navigate` 的源码项目为 `/Users/chenhuilin/beisen/recruit-common`，分别位于 `src/components/desensitize` 和 `src/components/navigate`；项目根 `package.json` 为用户指定入口。
- 用户说明 `@recruit-paas/feedback-fetch` 在 recruit-paas 项目内。检查源码包映射及安装依赖，不能仅因 src/components 没有同名目录就判断未知项目。本次在 recruit-paas 的 `node_modules/@recruit-paas/feedback-fetch` 定位到实际安装包7.0.4；若使用该发布产物而非迭代源码，必须明确记录来源与版本，不能冒充当前分支源码。

- 所有 `@recruit-base/*` 组件统一到 `/Users/chenhuilin/beisen/recruit-base` 定位，按实际 package.json 名称和导出解析组件目录，不因包名与文件夹名不同而判断未知项目。

## 扫描流程

一个页面完整处理后再处理下一页面。pagecode 是当前页面文件夹名，不是 page-scripts 等子目录名。

每一步开始和结束都要在控制台输出进度：pagecode、步骤、当前文件或组件、接口数量及缺失项。文件缺失或无法解析应明确输出，不能当成没有接口。

1. 扫描页面下 `page-scripts` 中所有 `.ts` 文件（包括子目录），不限制脚本文件名；结合页面绑定关系分析接口调用、调用时机、条件和参数来源。
2. 扫描 `datasources.ts`，提取各数据源的 `id`、`url`、`method`、`params`、`header`，同时检查 `beforeFetch`、`transformer`、`autoLoad` 等对请求和数据流的影响。
   - 将第一步中 `getDatasource(id)` 的使用与数据源 id 关联，追踪后续触发请求、返回值使用及依赖。
   - 获取数据源对象不等于发起请求，不能仅凭配置存在或 autoLoad 字段判定分类。
   - 前两步发现的接口属于页面类型接口。
3. 扫描 `dependencies.ts` 中的组件。明确排除 `@beisen/layout-container`、`@beisen-platform/related-objects-container`、`@beisen/flex-layout`，以及 `@beisen-platform/ux-standard-form`、`@beisen-platform/ux-standard-fields-common`、`@beisen-platform/ux-standard-fields-helper`、`@beisen-platform/nusion-field-label`、`@beisen-phoenix/field-phone-input`；以及 `@beisen-phoenix/common-constant-selector`；上述九个组件无需整理，其内部接口和依赖不纳入扫描；在扫描记录中注明排除范围，不因此暂停询问。其他组件例如 `@recruit-paas/task-execution`。到本地 recruit-paas 项目的本次迭代分支找到对应组件源码，分析组件中的静态、动态接口及依赖关系。
   - 保留实际组件包名、仓库和分支证据。源码不可用或分支不明时标记待确认，不以其他版本代替。
   - 组件源码分析可以复用，但动态接口必须保留当前页面上下文。

所有 `@recruit-jobad/*`、`@recruit-rpa/*` 和 `@beisen-platform/*` 包均按用户要求排除，不进入其源码，不为这些包询问路径；在扫描记录标明这三类前缀排除范围。

另外明确排除 `@beisen-phoenix/icon`、`@beisen-phoenix/modal-layer`、`@beisen-phoenix/rc-trigger`、`@beisen-phoenix/rc-virtual-list`、`@beisen-phoenix/tips`，不扫描其内部接口及依赖。

## 递归依赖扫描（必须完成）

- 对 dependencies.ts 的每个未排除组件，递归追踪其运行时代码中的 import、export/re-export、require、动态 import、React.lazy，以及内部组件、hooks、服务和请求封装引用，直到各分支均有明确结论。不能只查直接依赖或少数显眼请求。
- 记录完整组件引用路径，例如 resume-form-module → resume-form → resume-common-form → meta-constant-selector → common-constant-selector，同时记录发起接口的实际包名；不能一律把底层接口归到页面顶层组件包名。接口依赖边与组件引用链分开记录，避免将组件引用直接当作接口依赖。
- 按实际包名解析目录，查 package.json 或项目组件映射，不假定目录名与包名相同。追踪同项目基础包时去重并防止循环；纯类型导入无需作为运行时请求分支。
- 只读、打印模式、未触发事件、缓存命中等不能成为省略底层源码扫描的理由。先完整盘点源码接口，再单独分析本页调用条件；未触发或不可达接口也记录于扫描记录，不能误写为打开即请求或伪造上游接口。
- 已获用户明确排除的九个包仍为边界，不因递归引用重新纳入；引用到其他未知所属项目的包时，立即暂停询问路径，不擅自排除。
- 建立递归覆盖清单，每个分支记录已检查/无接口/明确排除/待定位及源码证据。存在待定位或尚未检查分支时，不宣称整页扫描完成，也不以目前发现的接口数量代表完整总量。

必须追踪引用的辅助函数、请求封装和关联源码，不能只靠 URL 字符串搜索就宣称完成扫描。

## 分类和标识

静态接口：打开当前页面就会请求的接口。
动态接口：依赖前置接口返回值的接口，包括返回值控制是否请求、作为下游请求参数、或两者兼有。

每条接口必须包含以下标识：

| 来源 | 静态接口标识 | 动态接口标识 |
| --- | --- | --- |
| 页面脚本或数据源 | pagecode | pagecode |
| 组件内部 | 当前组件包名 | pagecode + 组件包名 |

例如组件动态接口标识为 `setting + @recruit-paas/task-execution`。组件静态接口仍保留所属页面上下文，但标识是组件包名。

不能根据 GET/POST 判断分类。同一接口有不同调用场景时分别记录。证据不足或不符合已定义分类的调用标记待确认，不强行归类。

## 接口记录

每条接口至少包含：标识、pagecode、来源类型（页面/组件）、组件包名（如有）、静态/动态分类、URL、请求方式、URL query、params、header、调用时机或条件、源码文件及行号或代码符号。数据源接口还需记录 datasource id。

query、params 和 body 如存在，按实际请求位置分别记录。不擅自把所有 params 当成 URL query，也不因为 POST 就认定参数在 body 中。保留值、表达式及来源；未知运行时值记录表达式和不确定原因，不编造。区分空对象、未配置和无法确定。

## 动态依赖链路

每条直接依赖边需关联上下游接口记录及各自标识，并能查到下游的请求方式、query 及其他请求参数。明确标注依赖用途：

- 控制请求：上游接口及返回字段、实际判断条件、下游接口，说明该值如何决定是否请求。
- 传递参数：上游返回字段到下游 query、params、body 或 header 字段的映射，包含中间转换表达式。
- 两者兼有：同时记录判断条件和参数映射。

例如 A 返回 a，仅在 a === true 时请求 B，记录 A → B 的控制请求依赖；若 A 的 id 同时传入 B 的 params.recordId，还要记录 id → params.recordId 参数映射。

多层依赖 A → B → C 必须拆成 A → B、B → C 两条直接链路，不用一条长链替代，不推断不存在的 A → C。代码中的先后调用顺序不代表依赖，必须有控制流或数据流证据。

## 输出与核对

默认输出 Excel 工作簿，包含以下三个工作表。当前已确认的桌面模板为 `/Users/chenhuilin/Desktop/页面接口扫描模板.xlsx`；后续用户指定其他路径或格式时遵从新要求。更新已有文件前先读取其内容，保留已有数据和无关工作表。只维护本 skill 时不要执行接口扫描或改写工作簿。

### 静态注册

列顺序固定为：`类型 | 名称 | 编码 | 请求方式 | 请求接口 | 所有query参数`。

- 类型仅为页面或组件；名称填写对应页面或组件名称。
- 编码使用已确认的静态接口标识：页面用 pagecode，组件用组件包名。
- 请求接口填写 URL，请求方式记录 GET、POST 等。

### 动态依赖

列顺序固定为：`应用 | 页面编码 | 数据来源接口 | 返回字段 | 操作接口 | 接收字段 | 操作接口所有query参数 | 组件包 | 备注`。不添加用户表格末尾的空白列。

- 应用固定填写 `招聘管理系统`。
- 页面编码填写当前 pagecode，组件依赖也保留所属页面编码。
- 数据来源接口填写上游接口 URL；返回字段填写上游提供依赖值的字段路径。
- 操作接口填写下游接口 URL；接收字段填写下游接收参数的路径，例如 `query.recordId`、`params.recordId`。纯控制请求且不传参时填写 `不涉及传参`。
- 操作接口所有query参数使用字段路径格式，例如 `query.recordId、query.status、query.type`，不使用 JSON。列出全部 query 字段，而不只列依赖字段；值、来源和转换表达式记入备注。没有 query 时明确填写 `无`，无法确定时标注待确认。
- 组件包填写实际完整包名；页面代码中的依赖留空。页面编码与组件包组合表达动态接口的标识。
- 备注填写依赖用途（控制请求、传递参数、两者兼有）、判断条件、参数值及转换，以及上下游请求方式、其他请求参数、header 和源码证据等九列无法单独承载的必要信息。
- 每条直接依赖单独记录；有多个字段映射时逐个字段记录，不能丢失控制请求依赖。

### 扫描记录

记录页面名称、pagecode、页面目录、扫描步骤、扫描文件或组件、组件仓库、迭代分支、扫描状态、静态接口数量、动态接口数量、缺失项及待确认问题、源码位置。

不得为了适配表头而丢失前面要求分析的请求信息；没有独立列的信息保存在相应备注或扫描记录中。最终提供工作簿链接。

可用唯一记录编号关联接口清单和依赖表，但编号不能代替用户要求的 pagecode/组件标识。不要因 URL 相同而合并掉不同页面、组件、参数或触发条件。

交付前检查每页面的三个扫描步骤、每条接口的标识和请求信息、每条动态边的用途和参数映射。缺失源码、未解析的动态 URL 和无法确定的运行时条件如实记入待确认项。
