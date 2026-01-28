# YokoWebot 微信自动化兼容改造方案（WeChat 3.9.12 / 4.1.4）

## 背景与目标
- 背景：现有自动化基于 `uiautomation` 仅兼容微信 3.9.12 版本；微信 4.1.4 UI 全面变化。
- 目标：在不改变业务逻辑和对外用法的前提下，实现双版本兼容（3.9.12 / 4.1.4），通过接口抽象 + 驱动适配 + 工厂 + 版本检测的架构，逐步迁移，保证最小改动与稳定性。

## 现状架构梳理
- 核心入口：`WeRobotCore/core/WeChatType.py` 中的 `class WeChat` 体量较大，集成窗口句柄、控件查找、消息/通讯录/群聊/朋友圈/加好友等全流程逻辑，以及 `db_manager`、`account_info`、日志、重试等。
- 单例与账号绑定：
  - 主缓存：`_instances` 以窗口句柄（或进程）作为 key 存储实例。
  - 账号映射：`_account_handle_mapping` 维护 `account_id <-> window_handle` 映射；`update_account_handle_mapping`、`get_instance_by_account` 提供账号维度查找。
  - `initialize_multi(window_handle, account_info)` 支持多实例绑定并写入账号信息。
- 实例管理：`core/instance_manager_v2.py` 与 `instance_manager_v3.py` 管理多窗口实例、活跃实例切换，与 `WeChat` 的初始化/映射交互紧密（例如 v2 第 296、405、489 行）。
- 任务系统 v2：`task_system_v2/tasks/*.py` 在任务中直接持有 `WeChat` 实例并调用其方法，常见方法包括：
  - `get_user_id`、`is_shift_pressed`、`get_image_by_id`
  - `process_friend_requests`、`invite_friends_to_group`、`add_new_friend`、`back_to_chat_list`
  - `auto_moment_comment`
- API 服务：`api_server.py` 大量直接使用 `WeChat`，如：
  - 初始化与检测：`initialize_multi`、`check_connection_status`
  - 同步：`sync_groups`、`sync_contacts`
  - 其它：`get_current_user`、`invite_friends_to_group`、`SendFiles`
  - 同时经常访问 `wx.db_manager` 与 `wx.account_info`。
- 控件访问：`WeChatType.py` 内部大量使用 `UiaAPI`、`SessionList`、`MsgList`、`SearchBox`、`ButtonControl` 等旧版 UI 控件；外部模块通常不直接触碰这些内部控件，风险较低。

## 核心问题与挑战
- 旧版逻辑与 UI 控件强耦合，方法内部包含控件层实现细节，难以直接复用到 4.1.4。
- 业务层对 `WeChat` 的方法调用广泛，改动一处容易牵一发而动全身。
- 多实例与账号绑定、数据库管理、重试与日志需保持一致性与稳定性。

## 兼容架构设计
### 抽象接口层（已落实）
- 文件：`WeRobotCore/core/interfaces.py`
- 接口：
  - `IWeChatAutomation`：统一业务表面（初始化、多实例、会话与消息、通讯录与群聊、加好友与备注、群发、窗口控制、清理等）。方法命名尽量兼容现有（如 `GetContactList`、`GetGroupList`、`LoadMoreMessage`）。
  - `IWeChatAccountSingleton`：账号维度单例契约（更新/查找/列出/清理）。
- 目标：不改变现有业务流程与语义，仅抽象架构，为后续驱动适配与工厂改造提供稳定契约。

### 驱动与工厂
- 驱动实现（新增目录 `WeRobotCore/core/drivers`）：
  - `legacy_uia.py`：旧版 3.9.12 适配器，复用/迁移 `WeChatType.py` 中的控件操作与流程，保留 `UIRetry`、`UiaLogger` 等健壮性工具。
  - `weixin.py`：新版 4.1.2 适配器，使用 `weixin`（`WeChatAuto.py` 等）提供的元素操作实现同名接口方法。
- 版本检测（新增 `WeRobotCore/core/version_detector.py`）：
  - 探测窗口标题/可执行文件版本，或通过特征控件（旧版 `会话/消息/通讯录` 名称）试探，失败则判定为新版。
  - 返回枚举：`WeChatVersion.V3_9`、`WeChatVersion.V4_1`。
- 工厂（新增 `WeRobotCore/core/wechat_factory.py`）：
  - `WeChatClientFactory.create(window_handle=None)`：根据版本检测返回 `IWeChatAutomation` 实例。
  - 支持 `create_multi(window_handle, account_info)`。

### Facade 保持对外不变
- 保留 `WeRobotCore/core/WeChatType.py` 的 `class WeChat` 作为 Facade：
  - 继续维护 `_instances` 与 `_account_handle_mapping`，保持账号维度单例与窗口句柄绑定。
  - 保留 `db_manager`、`account_info` 等属性，并通过 Facade 方法对外提供；驱动只做 UI 层操作。
  - 方法签名/返回结构保持不变，内部逐步委派到驱动实现（先高频方法，逐步覆盖）。

### 配置与灰度
- 在配置中引入 `wechat_automation_mode: auto | legacy | weixin`：
  - `auto` 默认通过版本检测选择驱动。
  - 可强制 `legacy` 或 `pyweixin` 以便快速回退或试用。

## 方法映射清单（对外保持不变）
以下为当前业务常用的对外方法/属性，以及在兼容架构中的归属：

- 初始化相关
  - `initialize() -> {success, nickname, account_id}`：委派至驱动，Facade 汇总账号信息与加密、db 路径设置。
  - `initialize_multi(window_handle, account_info) -> {success, nickname, account_id}`：委派至驱动，Facade 负责账号-句柄映射与 `db_manager` 绑定。
  - `check_connection_status() -> bool/Dict`：驱动根据 UI 状态判断，Facade 对异常做统一封装。

- 窗口与账号
  - `SwitchToThisWindow()`、`get_window_handle()`：驱动实现；Facade 维护绑定句柄。
  - `get_account_info() -> Dict`、`account_info` 属性：Facade 持有并提供。
  - 单例管理：`update_account_handle_mapping(account_id, window_handle)`、`get_instance_by_account(account_id)`、`list_all_instances()`、`cleanup_invalid_instances()`：Facade 类方法保留。

- 会话与消息
  - `search_session(who) -> bool`：驱动实现（旧版使用 `SessionList`/`SearchBox`，新版使用 `pyweixin` 查找）。
  - `send_text(who, content) -> bool`：驱动实现；旧版使用 `EditControl` 与发送按钮，新版对应元素操作。
  - `send_image(who, image_path) -> bool`：驱动实现；旧版图片按钮与剪贴板方案，新版由 `pyweixin` 实现。
  - `SendFiles(who, filepath) -> Dict/Bool`：保持方法名与行为，驱动适配到 `send_file` 能力。
  - `read_messages(who=None, limit=None) -> List[Dict]`、`GetAllMessage`：驱动实现并统一结构化解析；旧版 `MsgList.GetChildren()`，新版 `pyweixin` 列表。
  - `LoadMoreMessage(speed=0.2) -> None`：驱动实现滚动加载。
  - `get_image_by_id(msg_id) -> Optional[str]`：驱动实现；旧版通过消息项按钮，需在新版查找对应控件或入口。

- 通讯录与群聊
  - `GetContactList(collect_detailed_info=False, save_file_path=None) -> List[Dict]`：驱动实现采集；Facade 负责入库（`db_manager.save_friends`）。
  - `GetGroupList() -> List[Dict]`：驱动实现采集；Facade 负责入库。
  - `sync_contacts() -> bool`、`sync_groups() -> bool`：Facade 汇总为“采集 + 入库”的组合流程，内部调用驱动采集与 `db_manager` 入库。
  - `invite_friends_to_group(friends, target_group) -> Dict`：驱动实现群邀请流程，Facade 维持结果格式与日志。

- 好友与备注
  - `process_friend_requests(...) -> Dict`：驱动实现“新的朋友”处理流程；Facade 维持重试与入库。
  - `add_new_friend(wxid, hello, remark, tag) -> Dict`：驱动实现添加流程，Facade 负责数据更新与状态回写。
  - `set_remark(wxid, remark) -> bool`：驱动实现备注编辑；Facade 保持 `_set_input_text` 的语义与返回规则。

- 其他能力
  - `auto_moment_comment(...) -> Dict`：驱动实现朋友圈评论点赞等；Facade 将返回统一化。
  - `back_to_chat_list() -> None`：驱动实现返回聊天列表的 UI 路径。
  - `is_shift_pressed() -> bool`：Facade/驱动均可实现（键盘状态检查）。
  - `get_current_user() -> Dict`、`get_user_id() -> str`：Facade 保持；驱动提供必要信息以补充。

- 属性访问
  - `db_manager`：保留在 Facade（`WeChat`）中，保持任务与 API 对其直接访问；驱动不持有 DB。
  - `account_info`：保留在 Facade 中，保持现有行为。

> 说明：以上方法名均保持现状，驱动内部可按接口适配，Facade 做委派与数据整合，确保调用方零改动或最小改动。

## 改造步骤与里程碑
### 阶段 0：抽象接口落地（已完成）
- 已新增 `core/interfaces.py` 定义 `IWeChatAutomation` 与 `IWeChatAccountSingleton`。

### 阶段 1：工厂与版本检测脚手架
- 新增 `core/version_detector.py` 与 `core/wechat_factory.py`，工厂返回驱动实例。
- 新增驱动目录与空实现（`drivers/legacy_uia.py`、`drivers/weixin.py`），仅包含接口方法签名与 TODO。
- 不改动 `WeChatType.py`；业务完全无感。

### 阶段 2：高频方法委派（最小面）
- 在 `WeChatType.py` 内部引入驱动实例，开始将以下方法改为委派：
  - `initialize` / `initialize_multi`
  - `search_session` / `send_text` / `read_messages` / `LoadMoreMessage`
  - `GetContactList` / `GetGroupList`
- 委派后 Facade 负责账号映射、DB 写入、返回结构组装；对外不变。

### 阶段 3：扩展剩余能力
- 分批迁移：`SendFiles`、`get_image_by_id`、`process_friend_requests`、`invite_friends_to_group`、`add_new_friend`、`set_remark`、`auto_moment_comment`、`back_to_chat_list`、`check_connection_status`。
- 保持方法名与返回结构一致，驱动封装新版 UI 路径与控件。

### 阶段 4：多实例与调度验证
- 验证 `instance_manager_v2/v3` 对 `WeChat.initialize_multi`、`update_account_handle_mapping` 的逻辑是否完好。
- 在 `api_server.py` 的多实例初始化流程进行冒烟测试（初始化所有实例、切换活跃实例、采集通讯录与群、消息发送）。

### 阶段 5：灰度与回归
- 配置开关支持强制选择驱动；默认 `auto`。
- 冒烟用例：
  - 启动后初始化所有实例并获取账号信息。
  - 选择会话并发送文本/图片/文件。
  - 读取消息与滚动加载。
  - 同步通讯录与群聊并存库（`db_manager`）。
  - 加好友流程（包括备注与打标签）。
  - 朋友圈评论点赞。

## 风险与缓解
- 新版 UI 变更频繁：封装控件查找与操作于驱动层；尽量通过语义化元素路径避免硬编码名称。
- 自动化稳定性：统一使用 `UIRetry` 与日志体系；超时与重试策略下沉到驱动。
- 多实例资源竞争：维持账号-句柄映射与实例隔离；避免跨实例共享状态。
- 性能与反自动化策略：维持人类节奏（随机延时）、可视区域判断与滚动策略；必要时引入兜底方案（坐标/OCR）。

## 目录与文件变更（规划）
- `WeRobotCore/core/interfaces.py`（已新增）：抽象接口层。
- `WeRobotCore/core/version_detector.py`：版本检测工具。
- `WeRobotCore/core/wechat_factory.py`：驱动工厂。
- `WeRobotCore/core/drivers/legacy_uia.py`：旧版驱动实现。
- `WeRobotCore/core/drivers/weixin.py`：新版驱动实现。
- `WeRobotCore/core/WeChatType.py`：Facade，方法逐步改为委派（对外不变）。

## 与任务系统/服务的兼容性说明
- `task_system_v2` 与 `task_system_v3`：所有对 `WeChat` 的调用保持不变。驱动迁移完成后行为一致。
- `api_server.py`：多实例初始化、账号映射、DB 操作等均留在 Facade 层；无需改动其对 `WeChat` 的用法。

---

本方案以“接口抽象 + 驱动适配 + 工厂 + 版本检测”为核心，优先保证对外 `WeChat` 的调用表面与行为一致，通过 Facade 逐步下沉到具体驱动实现，以最小改动实现双栈兼容与可维护性提升。
