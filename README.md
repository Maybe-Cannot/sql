# 项目方案

基于数据库的PDG构建，与评估。

系统状态建模

1. agent系统：
    整体描述：环境文件/agent/工具
    会话描述，请求对象（用户）/回应对象（agent）
    agent能力描述：
        权限：允许访问任意机密等级的信息/工具
        工具访问：允许配置若干基础工具，每个工具具有若干个参数
2. 行动轨迹描述
    我们的行动轨迹由整体体现为一个会话session文件
    行动轨迹由若干个结点组成
    共有5种类型的结点
    ["session","model_change","thinking_level_change","model-snapshot","message"]
    除了"session"结点之外，每个结点有通用属性：
        "id":  "25de8a5c"                               // 每个结点自身的表示
        "parentId":  null                               // 表明结点之间的关系
        "timestamp":  "2026-03-26T18:33:53.500Z"        // 时间消息

    除此之外，
        {
        "type":  "model_change",
        自有结点
        "provider":  "qwen-portal",
        "modelId":  "coder-model"
    }
    {
        "type":  "thinking_level_change",
        自有结点
        "thinkingLevel":  "off"
    }
    {
        "type":  "custom",
        自有结点
        "customType":  "model-snapshot",
        "data":  {
                     "timestamp":  1774550033509,
                     "provider":  "qwen-portal",
                     "modelApi":  "openai-completions",
                     "modelId":  "coder-model"
                 }
    }

    "message"结点仅包含某一个message字段
    每个字段有"role"字段，表明身份；有一个"content"结点承载结点信息；"timestamp"承载时间信息


    我们需要建立表项，在数据库中记录相关数据

2. 建图
    根据parentid，建立流式图，添加一个标记字段（是否涉及工具操作）来区分事件结点和操作结点
    只检查"role" ""stopReason"
3. 

