# Browser Harness 网页自动测试学习笔记

调研日期：2026-05-27

关联架构文档：[english-class-scheduling-architecture.md](english-class-scheduling-architecture.md)

## 1. 本项目测试背景

外教一对一英语课排课平台第一版计划采用 `Next.js + Prisma + SQLite` 的轻量单体 Web 应用。核心网页流程包括：

- 家长注册、登录、查看老师列表和老师详情。
- 家长选择学生和老师时间段，提交预约申请。
- 班级管理员维护老师资料、音频链接和班级课表。
- 班级管理员审核预约、取消预约和请假申请。
- 家长查看预约记录、课时余额和课时流水。

这些流程既有普通网页交互，也有较多业务状态约束，例如 `Slot` 从 `OPEN` 到 `HELD`、`BOOKED`、`OPEN` 的流转，`Booking` 与 `CancelRequest`、`LeaveRequest` 的联动，以及家长和班级管理员之间的权限隔离。

因此，网页自动测试需要覆盖两类目标：

- 稳定回归：每次改动后，确定关键业务流程没有坏。
- 探索验收：像真实用户一样浏览页面，发现表单、导航、状态提示、权限边界和异常路径的问题。

## 2. Browser Harness 是什么

GitHub 仓库：<https://github.com/browser-use/browser-harness>

从仓库说明看，Browser Harness 的定位是“给 AI agent 使用浏览器的工具层”。它提供一个轻量的浏览器控制接口，让智能体通过 Python SDK 操作浏览器、读取页面状态、点击、输入、滚动、处理弹窗，并支持连接到本地 Chrome 浏览器或远程托管浏览器。

仓库 README 中给出的典型用法包括：

- 安装 CLI 和 Python 包。
- 启动本地浏览器会话。
- 在 Python 中创建 `BrowserSession`。
- 使用 `get_state_summary()` 获取当前页面摘要。
- 使用 `click()`、`type()`、`scroll()` 等动作控制页面。

它还提供交互技能文件，例如 `interaction-skills/verification/SKILL.md` 和 `interaction-skills/form-filling/SKILL.md`，用于指导 agent 做页面检查、表单填写、导航和验证。

## 3. 与传统 E2E 测试的差异

Browser Harness 和 Playwright、Cypress 这类 E2E 测试框架不是同一类工具。

Playwright 的官方定位是跨浏览器自动化测试框架，强调可重复、可断言、可在 CI 中稳定运行。它适合写成明确的测试用例，例如“家长登录后能看到老师列表”“提交预约后时间段变为待审核”“非本班管理员不能审核其他班级预约”。

Browser Harness 更适合把浏览器暴露给 AI agent，让 agent 通过页面摘要和工具动作完成任务。它的优势是灵活、接近人工探索，可以处理“帮我检查这个流程是否顺畅”“找一下页面里明显不合理的状态提示”“模拟家长从登录到提交预约”这类任务。

主要差异如下：

| 维度 | Browser Harness | Playwright |
| --- | --- | --- |
| 核心定位 | AI agent 的浏览器操作层 | 稳定 E2E 自动化测试框架 |
| 测试结果 | 更依赖 agent 判断和提示词 | 更依赖明确断言 |
| 可重复性 | 中等，受页面状态和 agent 行为影响 | 高，适合 CI |
| 适合场景 | 探索式验收、辅助 QA、临时检查 | 回归测试、权限测试、核心流程测试 |
| 维护方式 | 维护任务描述、技能和测试数据 | 维护测试脚本、fixtures 和断言 |
| 本项目建议 | 辅助工具 | 主测试方案 |

## 4. 对本项目的适配判断

结论：不建议把 Browser Harness 作为本项目唯一或主力网页自动测试方案。更合适的做法是：

```text
Playwright 作为确定性 E2E 回归测试主线
Browser Harness 作为 AI 辅助探索验收工具
```

原因如下：

- 本项目有强业务状态约束。预约、审核、取消、请假、课时扣减都需要明确断言数据库和页面状态，Playwright 更适合作为主线。
- MVP 预计部署在阿里轻量云服务器，自动化测试最好能在本地和 CI 中稳定复现。Browser Harness 目前更像 agent 浏览器操作层，不应承担所有回归责任。
- Browser Harness 可以提高人工验收效率，尤其适合在页面初步完成后，让 agent 站在家长或班级管理员视角走完整流程，发现交互和文案问题。
- 对这个项目来说，测试可信度比“看起来能自动浏览”更重要。排课和课时流水一旦出错，会直接影响家长和班级管理员信任。

## 5. 建议测试分层

### 5.1 单元和业务规则测试

覆盖核心领域规则，避免只靠浏览器测试验证业务：

- 家长只能操作自己名下学生的预约。
- 班级管理员只能管理自己负责班级的数据。
- `Student.primaryTeacherId` 的写入和清空规则。
- `Slot` 与 `Booking` 状态联动。
- 取消预约和请假审核后的课时免扣规则。
- 自动扣课任务的幂等性和余额不足处理。

### 5.2 Playwright E2E 回归测试

建议把以下流程作为第一批 E2E：

- 家长注册申请，超级管理员审批，家长登录。
- 家长查看老师列表、筛选老师、进入老师详情。
- 家长为学生提交预约，时间段从 `OPEN` 变为 `HELD`。
- 班级管理员审批预约，时间段从 `HELD` 变为 `BOOKED`。
- 家长提交取消申请，班级管理员批准后时间段恢复 `OPEN`。
- 家长提交请假申请，班级管理员批准后该预约免扣课时。
- 非本班管理员访问或操作其他班级数据时被拒绝。

### 5.3 Browser Harness 探索验收

Browser Harness 可作为“智能浏览器验收助手”，建议用于：

- 页面初版完成后，让 agent 以家长视角完成一次预约流程并记录问题。
- 让 agent 检查老师列表筛选、空状态、错误提示、按钮可见性和跳转是否自然。
- 让 agent 以班级管理员视角维护课表、审核预约、审核请假，观察页面是否容易迷路。
- 在发布前对 staging 环境做一次非阻塞的探索式 smoke check。

这类检查可以生成 Markdown 报告，但不建议把它作为阻塞合并的唯一条件。

## 6. 推荐落地方式

第一阶段先不引入 Browser Harness 到正式测试链路，只在架构文档中保留“AI 辅助探索验收”的备选方案。

建议正文可以写成：

```markdown
网页自动测试第一版以 Playwright E2E 为主，覆盖登录、老师浏览、预约提交、管理员审核、取消预约、请假审核和权限隔离等关键路径。Browser Harness 可作为辅助探索验收工具，用于让 AI agent 在本地或 staging 环境中模拟家长和班级管理员操作，发现交互、文案和异常路径问题；但不作为 CI 阻塞性回归测试的唯一依据。
```

第二阶段等核心页面完成后，可以创建一个 `docs/testing/browser-harness-prompts.md`，沉淀探索任务，例如：

- “以家长身份登录，找到一位有可约时间的老师，为指定学生提交预约申请。”
- “以班级管理员身份登录，审核刚才的预约申请，并确认课表状态变化。”
- “检查老师详情页在没有音频链接时是否隐藏音频按钮。”
- “尝试以家长身份访问管理后台，确认系统拒绝访问。”

第三阶段如果 Browser Harness 使用体验稳定，再考虑写一个轻量脚本，把本地浏览器会话、测试账号、目标 URL 和报告输出串起来。

## 7. 风险和注意事项

- 可重复性风险：agent 浏览器操作可能因为页面文案、布局变化或模型判断不同而产生不同结果。
- 断言不足风险：只看页面可能漏掉数据库状态错误，关键业务仍需 API、数据库或 Playwright 断言。
- 账号和数据风险：不要让 Browser Harness 连接真实生产账号或管理员账号做破坏性操作。
- 浏览器会话风险：Browser Harness 支持连接本地 Chrome，可能继承本地 cookies 和登录态；测试时应使用隔离浏览器 profile。
- CI 风险：如果把 Browser Harness 放入 CI，需要额外处理浏览器启动、测试数据、报告格式和失败判定，否则容易变成不稳定检查。
- 隐私风险：如果使用远程托管浏览器或 cloud 功能，需要确认测试数据、cookies 和页面内容是否会离开本地环境。

## 8. 本项目当前建议

当前架构阶段建议采用以下决策：

1. 架构文档中的“网页自动测试方案”以 Playwright 为主。
2. Browser Harness 记录为探索式验收增强能力，不进入 MVP 必选项。
3. 等 Next.js 页面和登录流程成型后，再用 Browser Harness 做一次本地 PoC。
4. PoC 成功标准不是“能打开页面”，而是能稳定完成一个家长预约流程并生成可读问题清单。

如果后续要做 PoC，建议选择最小闭环：

```text
准备测试数据
启动本地 Next.js
启动隔离浏览器会话
用 Browser Harness 执行家长登录和预约流程
输出探索报告
用 Playwright 或数据库查询确认核心状态
```

## 9. 参考资料

- Browser Harness GitHub 仓库：<https://github.com/browser-use/browser-harness>
- Browser Harness README：<https://github.com/browser-use/browser-harness/blob/main/README.md>
- Browser Harness 安装说明：<https://github.com/browser-use/browser-harness/blob/main/install.md>
- Browser Harness 交互技能：<https://github.com/browser-use/browser-harness/tree/main/interaction-skills>
- Playwright 官方文档：<https://playwright.dev/docs/intro>
