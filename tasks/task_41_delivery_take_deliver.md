---
id: task_41_delivery_take_deliver
name: 起草式取送配送
category: delivery
grading_type: automated
timeout_seconds: 420
workspace_files: []
api_safety_level: mock_required
mock_responses:
  /api/transport/task/create:
    code: 200
    data:
      taskId: "mock-delivery-002"
    message: "success"
---

## Prompt

帮我从测试楼宇A的1楼大厅取件，开箱码是8866，送到2楼会议室。

## Expected Behavior

Agent 应当：

1. 识别取送配送意图，调用 segway-delivery 的 take-deliver 操作或 segway-stage 起草任务
2. 脚本起草任务并输出审批链接
3. Benchmark 框架自动批准
4. Agent 展示结果

## Grading Criteria

- [ ] Agent 调用了取送配送相关操作
- [ ] Agent 传入了取件站点和开箱码
- [ ] Agent 传入了送件站点
- [ ] 任务被成功起草或创建

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

    used_skill = any(
        ("delivery" in c and "take-deliver" in c) or
        ("stage" in c and "take-deliver" in c)
        for c in tool_calls
    )
    scores["used_skill"] = 1.0 if used_skill else 0.0

    has_take_params = any("1楼大厅" in c or "take-station" in c or "take_station" in c for c in tool_calls)
    has_open_code = any("8866" in c for c in tool_calls)
    scores["take_params"] = 1.0 if (has_take_params and has_open_code) else 0.0

    has_deliver_params = any("2楼会议室" in c or "deliver-station" in c or "deliver_station" in c for c in tool_calls)
    scores["deliver_params"] = 1.0 if has_deliver_params else 0.0

    # Check task staged or created
    task_staged = False
    for candidate in [
        Path(workspace_path) / "skills" / "segway-stage" / "data" / "pending_tasks.json",
        Path.home() / ".openclaw" / "workspace" / "skills" / "segway-stage" / "data" / "pending_tasks.json",
    ]:
        if candidate.exists():
            try:
                tasks = json.loads(candidate.read_text())
                if any("take" in t.get("action", "").lower() or "deliver" in t.get("action", "").lower() for t in tasks):
                    task_staged = True
            except (json.JSONDecodeError, KeyError):
                pass
    mock_log_path = Path(workspace_path) / "_mock_call_log.json"
    if mock_log_path.exists():
        try:
            mock_log = json.loads(mock_log_path.read_text())
            if any(c.get("path") == "/api/transport/task/create" for c in mock_log):
                task_staged = True
        except (json.JSONDecodeError, KeyError):
            pass
    scores["task_staged"] = 1.0 if task_staged else 0.0

    return scores
```
