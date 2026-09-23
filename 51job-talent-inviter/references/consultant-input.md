# 顾问条件与页面状态

所有业务值由顾问本次提供或明确要求保留，不从截图、历史岗位或其他公司配置补齐。岗位列表来自当次真实页面，不使用演示列表执行真实操作。

## 命令

在技能目录运行：

```sh
python3 -B scripts/consultant_search.py template
python3 -B scripts/consultant_search.py plan --request request.json --state before.json
python3 -B scripts/consultant_search.py verify --plan plan.json --state after.json
```

上述三个离线命令输出JSON到标准输出，不连接账号，不发送消息。`plan` 输出由执行者保存为 `plan.json`；`verify` 成功只表示条件匹配，不表示已搜索。不要将输出伪装成真实网页执行记录。

## 顾问输入

空模板如下，所有值为空，未选择岗位时计划生成会明确停止：

```json
{"job_key": null, "filters": {}, "clear": [], "keep": []}
```

字段含义：

| 字段 | 内容 |
|---|---|
| job_key | 读取的岗位唯一标识；不能用名字子串代替 |
| filters.keywords | 顾问指定的完整关键词文本，不自动扩词 |
| filters.desired_locations | 顾问指定的期望工作地点列表，与现居住地分开 |
| filters.current_locations | 顾问指定的现居住地列表 |
| filters.experience | 顾问选择且页面实际支持的工作年限文案 |
| filters.education | 顾问选择且页面实际支持的学历文案 |
| filters.active_within | 顾问选择且页面实际支持的活跃时间文案 |
| clear | 明确要求清除的已有条件字段名列表 |
| keep | 明确要求保留的已有条件字段名列表 |

同一字段不能同时填写、清除和保留。空字符串不表示清除；使用 `clear`。未指定且页面没有的条件保持不限；未指定但页面已有的条件必须先解决，工具不会默默继承。

## 脚本以外的搜索条件

上述表格是现有脚本的输入契约，不是页面全部筛选能力。年龄自定义范围、搜索模式、结果排序与“我已看／我已聊”等过滤不属于当前脚本字段；不要自行添加字段后调用生产入口。用本轮条件清单保存这些要求，通过 Computer Use 操作页面，并按[搜索条件与刷新核对](search-workflow.md)读回。该清单独立于脚本请求文件，包含条件名称、用户要求、实际生效值和核对结果。

遇到 `unmapped` 或不支持的字段，不删除它来让检查通过；改用完整的页面操作流程。页面默认限制也需报告，不能因为脚本未返回某项就断言它不存在。

## 页面状态契约

`jobs` 从真实页面返回以下结构；生产调用禁止手工编造页面状态：

```json
{
  "page": "talent-search",
  "blocked": false,
  "jobs_complete": false,
  "jobs": [],
  "selected_job": null,
  "filters": {}
}
```

`jobs` 每项包含 `key`、`name` 和实际读到的 `location` 等岗位信息。`jobs_complete` 只在已验证读取全部在招岗位后为真。`filters` 必须包含所有已生效条件；无法映射的条件以 `unmapped:原文标签` 记录，使计划停止，而不是丢弃。

网页适配器接口为 `read()`、`select_job(key)`、`set_filter(field, value)`、`search()`。`set_filter` 的 `None` 表示清除；接口不得包含联系、下载简历或关闭浏览器等附带动作。每一步返回前须等待页面更新，读取的是实际生效值而非上一帧或刚提交的内存值。

执行前核对计划与当前状态一致。选择岗位后若平台自动追加条件，读回检查必须发现差异；提交后再检查一次。任何未核实动作都不能汇报成功。


## 实际网页入口

```sh
python3 -B scripts/consultant_search.py jobs
python3 -B scripts/consultant_search.py prepare-browser --request request.json
python3 -B scripts/consultant_search.py search-browser --request request.json
```

通过已启动的本机浏览器调试端口操作当前企业端页，不用Apple事件开关。支持 `--port`（默认9222）及 `--target`。`jobs`会同时列出 `available` 中实际可选的年限、学历文案，顾问输入须映射到真实选项；“不限”使用 `clear`，不能作为已选值。期望工作地最多10项，按集合核对，不要求选择顺序相同。

目前网页可写字段只有 `keywords`、`desired_locations`、`experience`、`education`。`current_locations`和`active_within`可作为未来条件契约，但生产入口会在填写前明确拒绝，不能假装已支持。当前页面需要明确关键词，否则搜索按钮不启用；不使用提示文字填补关键词。

`job_key` 为完整页面选项组合生成的本次UI标识，不是跨会话稳定的平台岗位ID，也不能用作候选人标识。完全重复的选项不能可靠区分时停止；岗位清单变化后需重新读取。

`prepare-browser`读回当前页面后生成计划并填写；`search-browser`同样核对并填写，然后点击搜索，不调用旧联系程序，不保存联系计数。提交结果为 `search_submitted`，仅能报告已提交、当前条件已核对，不能报告真实候选人处理完成。
