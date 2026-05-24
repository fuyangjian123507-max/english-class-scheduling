# 外教一对一英语课排课平台典型工作流

本文档基于 [架构设计](./english-class-scheduling-architecture.md) 梳理第一版的典型业务工作流。每个 workflow 独立成节，并用一条横向流程图表达。

## 1. 注册与登录

Precondition:

- 用户尚未拥有可登录账号，或已有账号但尚未登录。
- 超级管理员具备注册审批权限。
- 每个家长账号最多可以添加 2 个学生。

```mermaid
flowchart LR
  start["用户打开注册页面"] --> register["提交注册信息"]
  register --> pending["账号状态: 待审批"]
  pending --> review["超级管理员审批注册"]
  review -- "通过" --> active["账号状态: 已启用"]
  review -- "拒绝" --> rejected["账号状态: 已拒绝<br/>不能登录"]
  active --> login["家长登录"]
  login --> dashboard["进入家长端"]
  dashboard --> studentCount{"已添加学生数量 < 2"}
  studentCount -- "是" --> addStudent["添加学生"]
  addStudent --> studentSaved["学生保存成功"]
  studentCount -- "否" --> blocked["提示最多添加 2 个学生"]
```

## 2. 家长查看老师并提交预约，班级管理员审核预约

Precondition:

- 家长已登录；未登录用户停留在登录界面，不能进入老师列表。
- 老师资料已创建，并至少存在一个 `OPEN` 的可约时间段。
- 家长账号已存在，且账号下至少绑定一个学生。
- 学生 `primaryTeacherId` 为空时表示新选老师；如果学生已有主老师，本次新增预约只能选择该老师加课。

```mermaid
flowchart LR
  start["家长打开老师列表"] --> filter["搜索/筛选老师"]
  filter --> detail["查看老师详情和可约时间"]
  detail --> select["选择学生和时间段"]
  select --> validate{"学生 primaryTeacherId 为空<br/>或目标老师为 primaryTeacherId"}
  validate -- "是" --> submit["提交预约申请"]
  validate -- "否" --> blocked["提示只能预约主老师加课"]
  submit --> pending["预约状态: PENDING<br/>时间段状态: HELD"]
  pending --> review["班级管理员审核预约"]
  review -- "通过" --> approved["预约状态: APPROVED<br/>时间段状态: BOOKED<br/>必要时写入 primaryTeacherId"]
  review -- "拒绝" --> rejected["预约状态: REJECTED<br/>时间段恢复: OPEN"]
```

## 3. 家长取消预约

Precondition:

- 家长已登录。
- 预约由当前家长提交，且预约状态为 `PENDING` 或 `APPROVED`。
- `PENDING` 预约尚未被班级管理员审核通过，可以由家长直接取消。
- `APPROVED` 预约已经确认占用时间段，且当前时间早于课程开始时间时，取消需要班级管理员审核。
- 同一预约不存在另一个有效的取消申请。

```mermaid
flowchart LR
  start["家长查看选中学生的预约"] --> pick["选择要取消的预约"]
  pick --> status{"预约状态"}
  status -- "PENDING" --> cancelPending["家长直接取消"]
  cancelPending --> pendingCancelled["预约状态: CANCELLED<br/>时间段恢复: OPEN"]
  pendingCancelled --> resetCheck["如果学生已无有效预约<br/>清空 primaryTeacherId"]
  status -- "APPROVED" --> submit["提交取消预约申请<br/>填写取消原因"]
  submit --> cancelReview["取消申请状态: PENDING"]
  cancelReview --> review["班级管理员审核取消预约"]
  review -- "通过" --> bookedCancelled["预约状态: CANCELLED<br/>时间段恢复: OPEN<br/>不扣课时"]
  bookedCancelled --> resetCheck
  review -- "拒绝" --> keep["预约保持: APPROVED<br/>时间段保持: BOOKED"]
```

## 4. 班级管理员维护老师和可约时间

Precondition:

- 班级管理员已登录。
- 班级管理员已绑定且只能管理一个班级。
- 被维护的老师属于该班级。

```mermaid
flowchart LR
  start["班级管理员进入后台"] --> classPage["进入负责班级"]
  classPage --> teacher["维护老师资料<br/>简介/国家/价格/音频链接"]
  teacher --> schedule["进入班级课表"]
  schedule --> slot["新增/编辑/关闭可约时间"]
  slot --> validate{"老师属于本班级<br/>时间段合法"}
  validate -- "通过" --> saved["保存 Slot<br/>状态: OPEN/CLOSED"]
  validate -- "失败" --> error["提示错误并保留原数据"]
```

## 5. 班级管理员充值课时

Precondition:

- 班级管理员已登录。
- 学生属于该班级管理员负责的班级。
- 学生已存在有效的课时账户。

```mermaid
flowchart LR
  start["班级管理员进入课时管理"] --> student["选择本班级学生"]
  student --> amount["填写充值课时和备注"]
  amount --> confirm["确认充值"]
  confirm --> balance["更新课时余额"]
  balance --> transaction["写入 LessonTransaction<br/>类型: RECHARGE"]
```

## 6. 课程结束后自动扣课

Precondition:

- 预约状态为 `APPROVED`。
- 对应 `Slot.endAt <= now`。
- 该预约尚未完成扣课或免扣处理。
- 自动扣课任务具备幂等保护，同一预约不能重复扣课。

```mermaid
flowchart LR
  start["定时任务扫描已结束课程"] --> match["匹配 APPROVED 预约<br/>Slot.endAt <= now"]
  match --> leave{"是否有有效请假申请"}
  leave -- "已批准" --> waive["标记免扣<br/>lessonDeductionStatus: WAIVED_BY_LEAVE"]
  leave -- "待审核" --> pendingLeave["暂不扣课<br/>lessonDeductionStatus: PENDING_LEAVE_REVIEW"]
  leave -- "无或已拒绝" --> balance{"课时余额是否足够"}
  balance -- "足够" --> deduct["扣减课时余额"]
  deduct --> transaction["写入 LessonTransaction<br/>类型: AUTO_DEDUCT"]
  balance -- "不足" --> failed["标记余额不足<br/>FAILED_INSUFFICIENT_BALANCE"]
```

## 7. 家长课前请假，班级管理员审核请假

Precondition:

- 家长已登录。
- 预约属于当前家长名下学生。
- 预约状态为 `APPROVED`。
- 当前时间早于课程开始时间。
- 同一预约不存在另一个有效的请假申请。
- 班级管理员已登录，且请假申请对应学生属于该班级管理员负责的班级。

```mermaid
flowchart LR
  start["家长查看已确认预约"] --> pick["选择尚未开始课程"]
  pick --> time{"当前时间早于课程开始"}
  time -- "是" --> submit["提交请假申请"]
  submit --> pending["请假状态: PENDING"]
  time -- "否" --> reject["系统拒绝创建请假"]
  pending --> list["班级管理员查看本班级待审核请假"]
  list --> review["审核请假申请"]
  review -- "通过" --> approved["请假状态: APPROVED<br/>预约标记免扣"]
  approved --> slot["Slot 保持 BOOKED<br/>不自动重新开放"]
  review -- "拒绝" --> rejected["请假状态: REJECTED"]
  rejected --> deduct["课程结束后正常自动扣课"]
```
