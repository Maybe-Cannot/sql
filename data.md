# 数据库设计准则

1NF 
    不可含有多值属性和内部结构
2NF 
    非主属性函数完全依赖于主键，意味着主键每个属性必须在确定后能够唯一对应目标属性
    eg:(学号/课程)->姓名，因为课程与姓名无关
3NF 
    不存在非主属性的传递依赖，即不应当有A->B->C
BCNF 
    如果存在非键主属性，且它对其余因素有影响，则需要拆分
    不仅非主属性要听主键的，主属性之间也不许私下勾结
4NF 
    一张表内不应当有多个多值属性
    比如学号决定课程，同时也决定借书，则课程和借书不应当在一张表中
5NF 
    如果三者之间存在“环形”制约，必须拆成三个二元关系，否则连接查询会出假数据

我们提高到4NF，进行数据库的建立（基于test.jsonl样本）

统一约定：
1) s_id统一表示一次会话ID，取自type=session记录的id（UUID）。
2) 除表1外，其余业务节点主键统一为(s_id, id)。
3) id为节点ID（样本中常见为8位16进制字符串）。
4) 所有数组都拆子表并使用seq_no保序。

表1：session
字段：s_id(key)/type/version/session_timestamp/cwd
主键：(s_id)
字段含义：
1) s_id：会话唯一标识，来源session.id。
2) type：记录类型，固定为"session"。
3) version：轨迹格式版本号（样本为3）。
4) session_timestamp：会话创建时间，ISO-8601字符串。
5) cwd：会话启动时工作目录。

表2：session_node（会话节点总表）
字段：s_id(key)/id(key)/parent_id/type/node_timestamp
主键：(s_id, id)
外键：(s_id) -> 表1(s_id)
字段含义：
1) s_id：所属会话ID。
2) id：当前节点ID。
3) parent_id：父节点ID，可为NULL（根节点）。
4) type：节点类型，取值"model_change"/"thinking_level_change"/"custom"/"message"。
5) node_timestamp：节点产生时间，ISO-8601字符串。

表3：model_change
字段：s_id(key)/id(key)/provider/model_id
主键：(s_id, id)
外键：(s_id, id) -> 表2(s_id, id)
约束：表2.type = "model_change"
字段含义：
1) s_id：所属会话ID。
2) id：对应session_node中的节点ID。
3) provider：模型提供方，如"qwen-portal"。
4) model_id：模型标识，如"coder-model"。

表4：thinking_level_change
字段：s_id(key)/id(key)/thinking_level
主键：(s_id, id)
外键：(s_id, id) -> 表2(s_id, id)
约束：表2.type = "thinking_level_change"
字段含义：
1) s_id：所属会话ID。
2) id：对应session_node中的节点ID。
3) thinking_level：思考等级，如"off"。

表5：custom_event
字段：s_id(key)/id(key)/custom_type
主键：(s_id, id)
外键：(s_id, id) -> 表2(s_id, id)
约束：表2.type = "custom"
字段含义：
1) s_id：所属会话ID。
2) id：对应session_node中的节点ID。
3) custom_type：自定义事件类型，样本为"model-snapshot"。

表5_1：custom_model_snapshot（custom_type=model-snapshot时）
字段：s_id(key)/id(key)/snapshot_timestamp/provider/model_api/model_id
主键：(s_id, id)
外键：(s_id, id) -> 表5(s_id, id)
字段含义：
1) s_id：所属会话ID。
2) id：对应custom_event节点ID。
3) snapshot_timestamp：快照时间戳（毫秒整型）。
4) provider：模型提供方。
5) model_api：模型接口类型，如"openai-completions"。
6) model_id：模型标识。

表6：message（message节点头表）
字段：s_id(key)/id(key)/role/node_timestamp
主键：(s_id, id)
外键：(s_id, id) -> 表2(s_id, id)
约束：表2.type = "message"
字段含义：
1) s_id：所属会话ID。
2) id：消息节点ID。
3) role：消息角色，取值"user"/"assistant"/"toolResult"。
4) node_timestamp：message节点外层时间戳（ISO-8601字符串）。

表7：user_message（对应message.role = user）
字段：s_id(key)/id(key)/message_timestamp
主键：(s_id, id)
外键：(s_id, id) -> 表6(s_id, id)
约束：表6.role = "user"
字段含义：
1) s_id：所属会话ID。
2) id：消息节点ID。
3) message_timestamp：user消息体内部时间戳（毫秒整型）。

表7_1：user_content（user的content数组）
字段：s_id(key)/id(key)/seq_no(key)/content_type/text
主键：(s_id, id, seq_no)
外键：(s_id, id) -> 表7(s_id, id)
字段含义：
1) s_id：所属会话ID。
2) id：消息节点ID。
3) seq_no：数组顺序号，从1开始。
4) content_type：内容类型，样本为"text"。
5) text：文本内容。

表8：assistant_message（对应message.role = assistant）
字段：
s_id(key)/id(key)/assistant_timestamp/api/provider/model/stop_reason/response_id/error_message
usage_input/usage_output/usage_cache_read/usage_cache_write/usage_total_tokens
cost_input/cost_output/cost_cache_read/cost_cache_write/cost_total
主键：(s_id, id)
外键：(s_id, id) -> 表6(s_id, id)
约束：表6.role = "assistant"
字段含义：
1) s_id：所属会话ID。
2) id：消息节点ID。
3) assistant_timestamp：assistant消息体内部时间戳（毫秒整型）。
4) api：调用API类型，如"openai-completions"。
5) provider：模型提供方。
6) model：模型标识。
7) stop_reason：停止原因，如"stop"/"toolUse"/"error"。
8) response_id：上游响应ID。
9) error_message：错误信息，仅stop_reason="error"时非空。
10) usage_input：输入token数。
11) usage_output：输出token数。
12) usage_cache_read：缓存读取token数。
13) usage_cache_write：缓存写入token数。
14) usage_total_tokens：总token数。
15) cost_input：输入成本。
16) cost_output：输出成本。
17) cost_cache_read：缓存读取成本。
18) cost_cache_write：缓存写入成本。
19) cost_total：总成本。

表9：assistant_content（assistant的content数组父表）
字段：s_id(key)/id(key)/seq_no(key)/content_type
主键：(s_id, id, seq_no)
外键：(s_id, id) -> 表8(s_id, id)
字段含义：
1) s_id：所属会话ID。
2) id：消息节点ID。
3) seq_no：数组顺序号，从1开始。
4) content_type：内容类型，取值"text"/"thinking"/"toolCall"。

表9_1：assistant_content_text
字段：s_id(key)/id(key)/seq_no(key)/text
主键：(s_id, id, seq_no)
外键：(s_id, id, seq_no) -> 表9(s_id, id, seq_no)
约束：对应表9.content_type = "text"
字段含义：
1) s_id：所属会话ID。
2) id：消息节点ID。
3) seq_no：数组顺序号。
4) text：assistant文本内容。

表9_2：assistant_content_thinking
字段：s_id(key)/id(key)/seq_no(key)/thinking/thinking_signature
主键：(s_id, id, seq_no)
外键：(s_id, id, seq_no) -> 表9(s_id, id, seq_no)
约束：对应表9.content_type = "thinking"
字段含义：
1) s_id：所属会话ID。
2) id：消息节点ID。
3) seq_no：数组顺序号。
4) thinking：思考文本。
5) thinking_signature：思考内容签名/类型标识。

表9_3：assistant_content_tool_call
字段：s_id(key)/id(key)/seq_no(key)/tool_call_id/tool_name/arguments_json
主键：(s_id, id, seq_no)
外键：(s_id, id, seq_no) -> 表9(s_id, id, seq_no)
唯一约束：(s_id, tool_call_id)
约束：对应表9.content_type = "toolCall"
字段含义：
1) s_id：所属会话ID。
2) id：消息节点ID。
3) seq_no：数组顺序号。
4) tool_call_id：工具调用唯一ID。
5) tool_name：工具名称。
6) arguments_json：工具参数（结构不固定，保留JSON）。

表10：tool_result_message（对应message.role = toolResult）
字段：s_id(key)/id(key)/tool_call_id/tool_name/is_error/tool_timestamp/details_json
主键：(s_id, id)
外键：(s_id, id) -> 表6(s_id, id)
外键：(s_id, tool_call_id) -> 表9_3(s_id, tool_call_id)
约束：表6.role = "toolResult"
字段含义：
1) s_id：所属会话ID。
2) id：消息节点ID。
3) tool_call_id：对应assistant侧的tool call ID。
4) tool_name：执行的工具名。
5) is_error：工具是否报错（布尔值）。
6) tool_timestamp：toolResult消息体内部时间戳（毫秒整型）。
7) details_json：工具返回详情（结构不固定，保留JSON）。

表10_1：tool_result_content（toolResult的content数组）
字段：s_id(key)/id(key)/seq_no(key)/content_type/text
主键：(s_id, id, seq_no)
外键：(s_id, id) -> 表10(s_id, id)
字段含义：
1) s_id：所属会话ID。
2) id：消息节点ID。
3) seq_no：数组顺序号，从1开始。
4) content_type：内容类型，样本为"text"。
5) text：工具结果文本。

补充约束（跨表一致性）：
1) assistant的toolCall与toolResult通过(s_id, tool_call_id)关联，保证一次调用可追踪。
2) 表7_1、表9、表10_1使用seq_no保证数组顺序可重建。
3) 对于message.role不同类型，必须且仅能落入表7/表8/表10三者之一。

