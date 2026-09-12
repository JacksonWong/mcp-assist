截至 v0.17.2，**mcp-assist 对 continue\_conversation 机制的具体实现：**

continue\_conversation 的处理逻辑集中在 custom\_components/mcp\_assist/agent.py 文件中的 \_build\_response\_result 方法里。

当处理过程中发生异常时，continue\_conversation 被硬编码为 False

**continue\_conversation 的判定逻辑总结**

| **条件**                                                                                      | **`continue_conversation` 值**                                  |
| ------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| 用户表达结束意图（`_detect_user_ending_intent` 返回 True）                                              | `False`                                                        |
| `follow_up_mode == "none"`                                                                  | `False`                                                        |
| `follow_up_mode == "always"`                                                                | `True`                                                         |
| `follow_up_mode == "default"` 且 LLM 通过 `set_conversation_state` 工具设置了 `_expecting_response` | 使用该工具设置的值                                                      |
| `follow_up_mode == "default"` 且未设置 `_expecting_response`                                    | 由 `_detect_follow_up_patterns` 根据响应文本模式检测(pattern detection)决定 |

​

用户结束意图检测（\_detect\_user\_ending\_intent）：
\- 用户消息中至少包含一个停用词/短语，

\- 用户消息中包含≤1个非停用词（不包括客服名称和匹配短语）

跟进模式检测（\_detect\_follow\_up\_patterns）：
模式 1：以问号结尾
模式 2：疑问句（用户可配置）

​

set\_conversation\_state 工具执行（在 \_execute\_tool\_calls 中）：

continue\_conversation 的判定优先级是：
用户结束意图（强制 False） > follow\_up\_mode 配置 > LLM 的 set\_conversation\_state 指示 > 文本模式检测（? 结尾 / 跟进短语）

​

continue\_conversation 存在完整且多层级的处理逻辑，基于用户配置（follow\_up\_mode）、LLM用户意图检测、LLM 工具指示（set\_conversation\_state），以及 (Pattern Detection) 响应文本模式检测（固化的文本：问号结尾/跟进短语）来综合判定。

​

**优化后的 System Prompt：**

```markdown
You are a helpful Chinese Simplified-speaking Home Assistant voice assistant. Respond naturally and conversationally to user requests in 简体中文.
如果当前语境有明显可跟进的下一步动作，应主动地提出跟进建议，并`set_conversation_state(expecting_response=true)`，
否则应主动结束对话，并`set_conversation_state(expecting_response=false)`。

## Weather
- Default city: 广州
---

永远不要输出 * 星号、emoji 表情符号
用最简洁的话语回复用户
```

​

**优化后的 Technical Prompt：**

```markdown
You are controlling a Home Assistant smart home system. You have access to sensors, lights, switches, and other devices throughout the home.

## Conversation Continuation Rules
You control whether the conversation continues via the `set_conversation_state` tool.
**Call `set_conversation_state(expecting_response=true)` ONLY when:**
- You ask a specific, context-relevant question (e.g., "Should I also turn off the bedroom lights?")
- The task is partially complete and you need clarification (e.g., "Which room did you mean?")
**Call `set_conversation_state(expecting_response=false)` or do NOT call the tool when:**
- The task is fully complete (e.g., "The kitchen lights are on.")
- The user said an ending word: "stop", "thanks", "bye", "done", "never mind", "cancel"
- There is no natural next step to suggest

## CRITICAL RULES
**Never guess entity IDs. Always make TWO tool calls for device control.** For ANY device-related request, you MUST:
1. FIRST call discover_entities to find the actual entities
2. THEN call perform_action (to control) or get_entity_details (to check status) using discovered IDs
3. **NEVER respond that you performed an action without actually calling perform_action**
4. This applies EVERY TIME - even for follow-up questions about different entities
5. Do NOT ask generic "anything else?" questions without specific context.
6. The user can end the conversation at any time with ending words.

**Common mistake:** Calling only discover_entities and then claiming you performed an action. This is WRONG. You must call perform_action to actually execute the action.

## Available Tools
- **discover_entities**: find devices by name/area/floor/label/domain/device_class/state (Make sure to accurately identify the device name; eg.: 'air conditioner 空调' and 'air purifier 空净 空气净化器' are not the same thing.)
- **perform_action**: control devices using discovered entity IDs
- **get_entity_details**: check states using discovered entity IDs, including area/floor/label context
- **get_entity_history**: get historical state changes for an entity (answers "when did X happen?")
- **list_areas/list_domains**: list available areas with floor/label context and device types
- **run_script**: execute scripts that return data (e.g., camera analysis, calculations)
- **run_automation**: trigger automations manually
- **set_conversation_state**: indicate if expecting user response
- **search**: search the web for current information
- **read_url**: read and extract content from web pages
- **IMPORTANT**: call_service is not available - use perform_action instead

## Device Control Workflow
**CRITICAL:** For ANY device control request, you MUST make TWO separate tool calls:

Example - "Turn on the kitchen light":
  1. discover_entities(domain="light", area="Kitchen")  # Find the light entity
  2. perform_action(domain="light", action="turn_on", target={{"entity_id": "light.kitchen"}})  # Actually turn it on

Example - "Set living room temperature to 22":
  1. discover_entities(domain="climate", area="Living Room")  # Find the thermostat
  2. perform_action(domain="climate", action="set_temperature", target={{"entity_id": "climate.living_room"}}, data={{"temperature": 22}})  # Set the temperature

**Never skip the perform_action step.** Discovering an entity does not control it - you must call perform_action to execute the action.

## Scripts (use run_script tool)
Scripts can perform complex operations and return data. **CRITICAL:** Always discover scripts first to get the correct entity ID.
- Script IDs use underscores (e.g., "script.stovsug_kjokken"), NOT spaces
- Script IDs must include the "script." domain prefix
- If script name has spaces in UI, the entity ID will use underscores instead

Example workflow:
  1. discover_entities(domain="script", name_contains="camera")
  2. run_script(script_id="script.llm_camera_analysis", variables={{"camera_entities": "camera.living_room", "prompt": "Is anyone there?"}})

## Automations (use run_automation tool)
Trigger automations manually. Check the index for available automations.

Example:
  run_automation(automation_id="alert_letterbox")

## Discovery Strategy
Use the index below to see what device_classes and domains exist, then query accordingly.
Floors and labels are first-class Home Assistant concepts. Check the index and area list to see available floor and label names, then use discover_entities with floor or label filters when relevant (for example, "upstairs" is usually a floor, not an area).
Areas, floors, entities, and sometimes devices may also have aliases. Treat aliases as valid user-facing names during discovery.

For ANY device request:
1. Check the index to understand what's available
2. Use discover_entities with appropriate filters (device_class, area, floor, label, domain, name_contains, state)
3. If no results, try broader search

## Response Rules
- Short, concise replies in plain text only
- Use Friendly Names (e.g., "Living Room Light"), never entity IDs
- Use natural language for states ("on" → "turned on", "home" → "at home")

{response_mode}

## Index
{index}

Current area: {current_area}
Current time: {time}
Current date: {date}
```

​

模式匹配 **Follow-up 关键字：**

```markdown
还有什么, 还有其他, 你会, 我应该, 我可以, 哪个, 怎么能, 那个怎么样, 有没有, 需要我, 需要, 需要帮你, 需要帮您, 吗, ？
```

​

**##mcp server 配置##**

MCP Server Settings 是所有 mcp-assist profile 共享的

mcp server IP 白名单支持 单个IP 和 IP范围，192.168.31.x 和 192.168.31.x/32 都支持，但因为配置更新逻辑存在bug，添加白名单后需重启HA。

Openclaw 作为mcp client时，设置连接方式为：流式HTTP，服务器地址：http\://192.168.31.x:8090 即可。

验证是否连通：

openclaw mcp doctor mcp-assist --probe

可通过 HA设置→系统→日志 查看mcp-assist的mcp server日志信息

​

_<u>截至 v0.17.2 这个版本，除了 server\_type: openai 这个模式，其它模式均不具备 mcp 工具完整处理逻辑。</u>_

​

​

​

**##SESSION 的处理逻辑##**



通过 openai compatible api 从 http header 透传 session id 仅在 hermes 端被支持，openclaw 需自行解决；

当 server\_type: openai 时，mike\_notte 原版对 Session ID 没有任何约束，openclaw 的 OpenAI compatible API 端点默认情况下是每请求无状态的——每次调用都生成新 session key；

当 server\_type: openclaw 时，可将模型字段设置为 agent:agent\_id:session\_id 来固定 Session ID；

​

OpenClaw 把 OpenAI 的 model 字段当作 agent target，可选值为：

| **\`model\` 值**                                   | **路由到**                                          |
| ------------------------------------------------- | ------------------------------------------------ |
| \`openclaw\`                                      | 配置的默认 agent                                      |
| \`openclaw/default\`                              | 默认 agent（\*\*稳定别名\*\*，即使默认 agent id 变了也安全，推荐硬编码） |
| \`openclaw/\<agentId>\` 或 \`openclaw:\<agentId>\` | 指定 agent                                         |
| \`agent:\<agentId>\`                              | 兼容别名                                             |

​

Openclaw gateway API 提供两个机制来固定 Session ID：

方式 A：请求体带 user 字段（推荐，标准 OpenAI 字段）
Gateway 会基于 user 字段派生一个稳定 session key，相同 user 值的调用共享同一个 agent 会话：

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "openclaw/default",
    "user": "conv:YOUR_CONVERSATION_ID",
    "messages": [{"role":"user","content":"Summarize my tasks for today"}]
  }'
```

_<u>注意粒度：user 应该按对话线程赋值（如 conv:thread-123），不要用账号级标识，否则该账号下所有会话/设备会共享同一个 OpenClaw 会话</u>_

​

fork from mike\_notte mcp-assist v0.17.2 repo，修改 \~/mcp-assist/.../agent.py，实现上述逻辑：

```python
if self._current_conversation_id:
    payload["user"] = f"mcp-assist:{self._current_conversation_id}"
```

​

**为了在HA集成时不跟原版冲突，需修改一系列元数据，避免重复出现 “mcp-assist” 字面量，全局搜索并改为 mcp\_assist\_openclaw：**

| **Step** | **文件**                                                                                       | **改了什么**                                                                                                 |
| -------- | -------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| 1        | custom\_components/mcp\_assist\_openclaw/const.py                                            | DOMAIN = "mcp\_assist\_openclaw" + SYSTEM\_ENTRY\_UNIQUE\_ID = "mcp\_assist\_openclaw\_system\_settings" |
| 2        | .../manifest.json                                                                            | domain / name / codeowners / documentation / issue\_tracker 全部指向Jackson的 fork                            |
| 3        | .../strings.json                                                                             | entity.conversation.mcp\_assist → mcp\_assist\_openclaw                                                  |
| 4        | .../translations/\*.json (22 个)                                                              | 全部同上                                                                                                     |
| 5        | 文件夹 custom\_components/mcp\_assist/ → custom\_components/mcp\_assist\_openclaw/(git mv 保留历史) | ​                                                                                                        |
| 6        | 残留字面量检查                                                                                      | grep 'mcp\_assist' / "mcp\_assist" 都是空的 ✅                                                                |

**​**

修改完成后，git push 到 jackson's repo，在HACS中引入自定义仓库→重启HA，设置→设备与服务→添加集成。

​

方式 B：x-openclaw-session-key 请求头

```bash
curl ... -H 'x-openclaw-session-key: myapp-thread-123' ...
```

* 适合跨多个 client/线程做显式路由的场景
* 使用应用自有 key，避开保留命名空间 subagent:、cron:、acp:，否则返回 400 invalid\_request\_error
* 注意：用它显式选中/延续 incognito 会话需要 operator.admin 权限，否则 403

​

_<u>只要不做 `/new，/reset` 或不配置 daily，idle reset，该 key 下的 `sessionId` 保持不变</u>_

_<u>安全提醒：官方强调这个端点等价于完整 operator 访问权限，Gateway务必只监听 loopback/tailnet/私有入口，不要暴露公网</u>_

