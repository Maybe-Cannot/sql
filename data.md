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

我们提高到4NF,进行数据库的建立

表1：
session表，用于记录不同session的信息
type/id(key)/version/timestamp/cwd/s_id

表二：
会话节点表
s_id(key)/id(key)/parentId/type/timestamp
    s_id:
    id: 8位16进制
    parentid：8位16进制 + NULL
    type："model_change","thinking_level_change","model-snapshot","message"
    timestamp：字符串，"2026-03-26T18:33:58.964Z"


表三：
model_change
s_id(key)/id(key)/provider/modelId

表4：
thinking_level_change
s_id(key)/id(key)/thinkingLevel

表5：
custom
s_id(key)/id(key)/customType/data（json）
"data":  {
                     "timestamp":  1774550033509,
                     "provider":  "qwen-portal",
                     "modelApi":  "openai-completions",
                     "modelId":  "coder-model"
                 },


表6：
message,描述节点
s_id(key)/id(key)/role/timestamp
role：user/assistant/toolResult

表7
user

表8
assistant

表9
assistant_content
"text"/"thinking"/"toolCall"

表10
toolResult

