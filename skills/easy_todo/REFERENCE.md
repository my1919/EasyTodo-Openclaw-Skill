# EasyTodo 公共规则

## 一、用途

这个文件是整个多 skill 系统的公共约束中心。

它只负责：

- 统一数据结构
- 统一读写边界
- 统一失败处理
- 统一展示原则
- 统一任务匹配优先级

它不负责路由，不直接执行任何动作。

---

## 二、全局原则

### 1. 架构原则

- 系统必须保持“1 个主入口 + 多个动作分支”
- 主入口只做意图识别、语义归一、单 skill 路由
- 每次只允许命中一个子 skill

### 2. 语言原则

- 所有 skill 提示词必须是中文
- 所有用户可见输出必须是中文

### 3. 数据原则

- 所有 JSON 顶层必须是对象，不能是数组
- 数据文件只允许放在 `workspace/easy_todo/data/`
- 所有分支都必须先读后写
- 所有成功写库的 skill 都必须同步更新顶层 `updated_at`

### 4. 安全原则

- AI 默认只建议，不默认写库
- 只有被授权的分支才能写库
- 不允许新增复杂状态机

---

## 三、数据文件结构

### 1. tasks.json 顶层结构

`tasks.json` 顶层对象必须包含：

- `version`
- `updated_at`
- `tasks`

其中：

- `version`：数据版本号
- `updated_at`：整个任务库最后更新时间
- `tasks`：任务数组

### 2. tasks.json 单个任务字段

每个任务对象必须保留以下字段：

- `id`
- `title`
- `display_title`
- `theme_tag`
- `project_tags`
- `status`
- `priority`
- `planned_for_date`
- `due_date`
- `next_action`
- `task_time`
- `reminder_at`
- `reminder_lead_minutes`
- `reminder_sent`
- `created_at`
- `completed_at`
- `source`
- `notes`

字段含义：

- `id`：任务唯一标识
- `title`：自然标题，保留用户原意，不强制改格式
- `display_title`：展示标题，允许包含 `【类型】` 等展示优化
- `theme_tag`：主类型标签
- `project_tags`：项目标签数组
- `status`：轻量状态字段
- `priority`：优先级
- `planned_for_date`：计划归属日期
- `due_date`：截止日期
- `next_action`：下一步动作
- `task_time`：任务时段或预估执行时间
- `reminder_at`：提醒时间
- `reminder_lead_minutes`：提前提醒分钟数
- `reminder_sent`：提醒是否已发出
- `created_at`：创建时间
- `completed_at`：完成时间
- `source`：来源
- `notes`：备注

### 3. inspirations.json 顶层结构

`inspirations.json` 顶层对象必须包含：

- `version`
- `updated_at`
- `inspirations`

其中：

- `version`：数据版本号
- `updated_at`：整个灵感库最后更新时间
- `inspirations`：灵感数组

### 4. inspirations.json 单条灵感字段

每条灵感对象优先使用以下字段：

- `id`
- `content`
- `theme_tag`
- `created_at`
- `source`
- `converted_to_task`

如果保留以下字段：

- `comment`
- `suggestion`

它们只能作为附加展示信息，不参与：

- 状态判断
- 任务匹配
- 灵感匹配
- 写回逻辑

---

## 四、轻量状态约束

系统不允许新增复杂状态机。

任务状态只允许使用以下集合：

- `inbox`
- `next`
- `done`
- `dropped`

约束：

- 不新增流程型中间态
- 不做自动状态流转
- 状态变化必须由明确分支触发

---

## 五、统一任务匹配优先级

所有涉及“完成任务”“更新任务”“应用建议”“并入今天”“放弃任务”的分支，都必须遵守以下匹配顺序：

1. 当前上下文编号
2. 精确 `id`
3. 精确 `title`
4. 模糊匹配必须二次确认

补充约束：

- 如果当前上下文编号存在，就优先按当前编号解释
- 如果用户给出了精确 `id`，优先于标题模糊猜测
- 如果用户给出精确 `title`，必须做精确匹配，不要自行模糊扩展
- 只要进入模糊匹配，就不得直接写库，必须先二次确认

---

## 六、各 skill 的读写边界

### 1. easy_todo

- 只路由
- 不读库
- 不写库
- 不生成任务对象

### 2. easy_todo_capture

- 可以读取 `tasks.json`
- 可以新建任务并写回 `tasks.json`
- 只负责新增任务

### 3. easy_todo_view

- 可以读取 `tasks.json`
- 可以读取 `inspirations.json`
- 默认纯读取
- 不承担写回逻辑

### 4. easy_todo_complete

- 可以读取 `tasks.json`
- 可以更新任务完成状态并写回 `tasks.json`
- 只处理显式完成动作

### 5. easy_todo_inspiration_capture

- 可以读取 `inspirations.json`
- 可以新建灵感并写回 `inspirations.json`

### 6. easy_todo_ai_assist

- 默认只建议，不写库
- 只有用户明确说“应用建议 / 写回 / 直接更新”时，才允许有限字段写回 `tasks.json`

允许写回字段白名单：

- `display_title`
- `theme_tag`
- `project_tags`
- `priority`
- `planned_for_date`
- `due_date`
- `next_action`
- `task_time`
- `reminder_at`
- `reminder_lead_minutes`
- `reminder_sent`
- `notes`

禁止写回：

- 不允许新建任务
- 不允许直接完成任务
- 不允许修改灵感库
- 不允许修改不在白名单中的任务字段

---

## 七、核心行为约束

### 1. title 与 display_title

- `title` 始终保留自然标题
- `display_title` 用于展示优化
- 不允许把展示逻辑反写进 `title`

### 2. 今日任务定义

今日任务必须定义为同时满足：

- `planned_for_date = 今天`
- `status = next`

不满足这两个条件的任务，不进入今日任务主列表。

### 3. 昨天未完成定义

昨天未完成必须定义为同时满足：

- `planned_for_date < 今天`
- `status = next`

这类任务可以作为候选项展示，但不等于自动并入今天。

### 4. 完成动作

- 只有显式完成命令才能触发 complete 分支
- 不允许在 view 或 ai_assist 默认模式里自动判定完成

### 5. 灵感处理

- 灵感不自动转任务
- 灵感记录只写灵感库
- `converted_to_task` 只表示是否已被转成任务，不参与匹配逻辑

---

## 八、失败处理逻辑

### 1. 文件读取失败

如果 JSON 不存在、不可解析、缺少顶层关键字段：

- 不允许继续写库
- 不允许伪造读取结果
- 必须明确说明读取失败

### 2. 文件结构损坏

以下任一情况视为结构损坏：

- `tasks.json` 缺少 `version`、`updated_at`、`tasks`
- `inspirations.json` 缺少 `version`、`updated_at`、`inspirations`
- 顶层不是对象

处理方式：

- 立即停止本次写入
- 返回结构损坏提示

### 3. 写入失败

如果准备结果成功但写库失败：

- 先返回整理结果或建议结果
- 再明确说明这次没有成功写入本地数据
- 不允许说“已记录”“已更新”“已完成”

### 4. 编号歧义

如果用户说“完成 1、2”或“把 1 和 3 并入今天”，但编号上下文不明确：

- 不允许写库
- 必须返回澄清提示

### 5. 模糊匹配

如果任务匹配进入模糊匹配阶段：

- 不允许直接写库
- 必须先做二次确认

---

## 九、展示规则

### 1. 通用聊天工具优先

- 输出优先适配普通聊天工具
- 不依赖特定卡片协议
- 通过轻层级、短行、短段落来提升可读性

### 2. 标题与展示

- 主标题可以保留
- 其他模块标题要弱化
- 不要堆叠大标题
- 不要大段粗体

### 3. 任务展示

- 高优先级任务可用完整 `display_title` 加粗
- 普通未完成任务默认不加状态符号
- 已完成任务保留 `✅`
- 每条任务尽量 1-2 行

### 4. 今日视图固定结构

建议固定为：

1. 标题
2. 今日任务
3. 昨天未完成
4. 灵感
5. 今日成就

---

## 十、实现自检要求

每生成一个 skill 文件后，都必须自检：

1. 是否越权写库
2. 是否违反单一职责
3. 是否破坏数据结构

自检标准：

- 主入口不能读写 JSON
- view 默认不能写库
- ai_assist 默认不能写库
- 所有写库 skill 都必须遵守顶层对象结构
- 所有成功写库的 skill 都必须更新顶层 `updated_at`
