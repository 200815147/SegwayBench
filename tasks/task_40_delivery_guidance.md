---
id: task_40_delivery_guidance
name: 起草式引领配送
category: delivery
grading_type: automated
timeout_seconds: 420
workspace_files: []
api_safety_level: mock_required
mock_responses:
  /api/transport/task/create:
    code: 200
    data:
      taskId: "mock-delivery-001"
    message: "success"
---

## Prompt

帮我派一个机器人到测试楼宇A的1楼大厅。

## Expected Behavior

Agent 应当：

1. 识别配送意图，调用 segway-delivery 的 guidance 操作或 segway-stage 的 stage 命令
2. 脚本起草任务并写入 pending_tasks.json，输出审批链接（[ACTION_STAGED]）
3. Benchmark 框架自动调用 /approve 端点模拟用户确认
4. Agent 将起草结果（含审批链接）展示给用户

关键评估点：Agent 是否正确使用了 delivery/stage skill，任务是否被成功起草。

## Grading Criteria

- [ ] Agent 调用了 segway-delivery 或 segway-stage 相关操作
- [ ] Agent 使用了楼宇名称参数
- [ ] 任务被成功起草或创建（pending_tasks.json 或 mock log 中有记录）
- [ ] Agent 向用户展示了结果

## Automated Checks

```python
def grade(transcript: list, workspace_path: str) -> dict:
    import json
    from pathlib import Path

    scores = {}
    tool_calls = []

    for entry in transcript:
        if entry.get("type") == "message":
            message = entry.get("message", {})
            if message.get("role") == "assistant":
                for item in message.get("content", []):
                    if isinstance(item, dict) and item.get("type") == "toolCall":
                        command = str(item.get("arguments", "")) + str(item.get("toolName", ""))
                        tool_calls.append(command)

    # Check delivery or stage skill used
    used_skill = any(
        ("delivery" in c and "guidance" in c) or
        ("stage" in c and "task.create" in c)
        for c in tool_calls
    )
    scores["used_skill"] = 1.0 if used_skill else 0.0

    # Check name-based params
    used_names = any("area-name" in c or "area_name" in c or "测试楼宇" in c for c in tool_calls)
    scores["used_name_params"] = 1.0 if used_names else 0.0

    # Check task was staged or created
    task_staged = False
    # Check pending_tasks.json
    for candidate in [
        Path(workspace_path) / "skills" / "segway-stage" / "data" / "pending_tasks.json",
        Path.home() / ".openclaw" / "workspace" / "skills" / "segway-stage" / "data" / "pending_tasks.json",
    ]:
        if candidate.exists():
            try:
                tasks = json.loads(candidate.read_text())
                if any(t.get("action_name", "") in ("创建引领运单", "delivery.guidance") or
                       "guidance" in t.get("action", "") for t in tasks):
                    task_staged = True
            except (json.JSONDecodeError, KeyError):
                pass
    # Also check mock log as fallback
    mock_log_path = Path(workspace_path) / "_mock_call_log.json"
    if mock_log_path.exists():
        try:
            mock_log = json.loads(mock_log_path.read_text())
            if any(c.get("path") == "/api/transport/task/create" for c in mock_log):
                task_staged = True
        except (json.JSONDecodeError, KeyError):
            pass
    scores["task_staged"] = 1.0 if task_staged else 0.0

    # Check response
    has_response = any(
        entry.get("type") == "message" and
        entry.get("message", {}).get("role") == "assistant" and
        len(entry.get("message", {}).get("content", [])) > 0
        for entry in transcript
    )
    scores["response_provided"] = 1.0 if has_response else 0.0

    return scores
```
