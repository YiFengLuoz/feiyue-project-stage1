### 运行文件要配置环境
```
sudo apt update
sudo apt install python3.12-venv -y
```
---
### 每次修改内容要启动虚拟环境
- 要在终端书写，出现venv
```
python3 -m venv venv(创建虚拟环境)
source venv/bin/activate（激活虚拟环境）
```
### 要配置ai大模型的环境
- 举例配置api密钥[（.ven）,openai接口（mcp统一书写），谷歌，cluade模型]
```
pip install python-dotenv openai google-generativeai anthropic
```

## 知识点
### client.chat.completions.create
- 这是固定写法，所有兼容 OpenAI 接口的模型（阿里云、文心一言、本地大模型）都必须用这个格式调用：
---
### messages=[{"role": "user", "content": "test"}]
- 这是大模型唯一要求的消息格式（固定结构，不能改）：
- messages：消息列表（必须是数组 / 列表）
"role": "user"：角色 = 用户（你发的话）
"content": "test"：消息内容 = 测试词（极简，不浪费 token） 
---
### 注释版原代码
``` py
import os#操作系统
import json#读取文本文件
from datetime import datetime#保存数据

# ===================== 【核心配置】支持多种大模型 可自行添加/删除 =====================
# 格式：{"name": "模型名称", "endpoint": "接入点ID", "base_url": "调用地址", "api_key": "API密钥"}
#利用字典保存数据
MODELS = [
    {
        "name": "豆包1.8-Pro",
        "endpoint": "ep-xxxxxxx",
        "base_url": "https://ep-xxxxxxx.cn-beijing.volces.com/api/v3",
        "api_key": "你的火山方舟API密钥"
    },
    {
        "name": "豆包1.8-Lite",
        "endpoint": "ep-yyyyyyy",
        "base_url": "https://ep-yyyyyyy.cn-beijing.volces.com/api/v3",
        "api_key": "你的火山方舟API密钥"
    }
]
# ====================================================================================

# 全局配置
CURRENT_MODEL_INDEX = 0  # 默认使用第一个模型
conversation_history = []
HISTORY_FILE = "conversation.json"

# ===================== 工具函数 =====================
# 加载对话历史
def load_history():
    global conversation_history#修改全局变量
    #寻找本地对话文件
    if os.path.exists(HISTORY_FILE):
        try:
            #打开文件，只读，保持中文不会乱码
            with open(HISTORY_FILE, "r", encoding="utf-8") as f:
                conversation_history = json.load(f)
                #json转python列表或者字典
        except:
            conversation_history = []

# 保存对话历史
def save_history():
    with open(HISTORY_FILE, "w", encoding="utf-8") as f:
        json.dump(conversation_history, f, ensure_ascii=False, indent=2)
        #与load类似这个是写（读），主要作用一致，保存聊天记录。ture会转为乱码，这是关闭asc码的转换，首行缩进2

# 字符数截断机制
def truncate_conversation():
    """超出字符限制自动删除最早的对话"""
    model = get_current_model()
    max_total = model["max_chars"] - RESERVED_CHARS

    # 统计所有对话的总字符数
    total_chars = 0
    for ct in conversation_history:
        total_chars += len(ct["user"]) + len(ct["ai"])

    if total_chars <= max_total:
        return

    # 超出限制，删除最早的对话
    print(f"\n对话过长，自动清理最早记录...")
    while total_chars > max_total and len(conversation_history) > 0:
        removed = conversation_history.pop(0)
        total_chars -= len(removed["user"]) + len(removed["ai"])
        #while ...：循环执行，直到两个条件满足：总字符数 ≤ 上限，对话历史不为空
        #conversation_history.pop(0)：删除第一条对话（最早的那一条）
        #total_chars -= ...：总字符数减去被删掉的对话长度，更新数值
    print(f" 清理完成，当前字符数：{total_chars}/{max_total}")

# 获取当前模型信息
def get_current_model():
    return MODELS[CURRENT_MODEL_INDEX]

# API连接验证
def verify_api(model):
    print(f"\n正在验证模型：{model['name']}")
    try:
        from openai import OpenAI
        #导入库，初始化ai客户端
        client = OpenAI(api_key=model["api_key"], base_url=model["base_url"])
        client.chat.completions.create(
            model=model["endpoint"],
            messages=[{"role": "user", "content": "test"}]
        )
        #client你提前创建好的 AI 客户端（绑定了密钥 + 接口地址）
        #chat代表：调用聊天对话接口（和 AI 聊天专用）
        #completions代表：让 AI 生成回复 / 补全内容
        #create代表：创建一个新的对话请求
        #发送一个简单请求，通过即可用
        print(f" {model['name']} 验证成功！")
        return True
    except Exception as e:
        print(f" 验证失败：{str(e)}")
        return False
    #异常捕获，防止程序崩溃数据保存不到

# 调用AI接口
def call_ai(message):
    model = get_current_model()
    try:
        from openai import OpenAI
        client = OpenAI(api_key=model["api_key"], base_url=model["base_url"])
        response = client.chat.completions.create(
            model=model["endpoint"],
            messages=[{"role": "user", "content": message}]
        )
        return response.choices[0].message.content
        #model：指定用哪个模型（如 qwen-plus/gpt-3.5-turbo）
        #messages：必须是列表格式，包含对话内容
        #role: user：代表这是用户发送的消息
        #content: message：你输入的聊天内容（比如 “什么是 RAG”）
        #response：AI 返回的完整响应数据（包含回答、模型信息、状态等
    except Exception as e:
        return f"调用失败：{str(e)}"

# ===================== 命令处理函数 =====================
# 帮助命令
def show_help():
    print("""
 多模型AI助手命令说明
/model list   : 查看所有可用模型
/model [数字] : 切换模型（例：/model 0）
/help        : 查看帮助
/clear       : 清空当前对话
/reset       : 重置所有对话（删除文件）
/exit        : 退出程序
""")

# 列出所有模型
def list_models():
    print("\n=== 可用模型列表 ===")
    #enumerate(列表)
    #作用：遍历列表时，同时拿到「序号」和「元素本身」
    #i：自动生成的序号（从 0 开始）
    #model：MODELS 列表里的每一个模型字典
    #MODELS：你提前定义好的模型列表（长这样）
    for i, model in enumerate(MODELS):
        current = " 当前使用" if i == CURRENT_MODEL_INDEX else ""
        print(f"{i+1}. {model['name']} {current}")
        #三元运算符（简化版 if-else）
        #一行写完判断逻辑：
        #如果 条件成立 → 赋值 A，否则 赋值 B
        #这里的逻辑：
        #如果序号 i 等于 当前使用的模型索引 CURRENT_MODEL_INDEX
        #→ 显示 ✅ 当前使用
        #→ 否则显示空（什么都不写）
    print("===================\n")

# 切换模型
def switch_model(index):
    global CURRENT_MODEL_INDEX
    #if index >= 0 and index < len(MODELS):连续比较
    if 0 <= index < len(MODELS):
        CURRENT_MODEL_INDEX = index
        model = get_current_model()
        verify_api(model)
        #作用：自动测试新模型的密钥、接口是否可用
        #验证内容：API 密钥、base_url、模型名称是否正确，网络是否通畅
        print(f" 已切换至模型：{model['name']}")
    else:
        print(" 模型编号不存在！")

# ===================== 主程序 =====================
def main():
    load_history()
    print(" 多模型AI助手启动成功！")
    # 验证默认模型
    verify_api(get_current_model())
    show_help()
    #无限循环
    while True:
        user_input = input("user > ").strip()
        if not user_input:
            continue

        # 命令匹配
        if user_input == "/exit":
            save_history()
            print(" 再见！")
            break

        elif user_input == "/help":
            show_help()
#直接调用函数
        elif user_input == "/clear":
            global conversation_history
            conversation_history = []
            print(" 对话已清空！")

        elif user_input == "/reset":
            conversation_history = []
            if os.path.exists(HISTORY_FILE):
                os.remove(HISTORY_FILE)
            print(" 已重置所有对话！")

        elif user_input.startswith("/model"):
            #字符串方法：startswith(前缀)作用：判断用户输入的内容，是否以 /model 开头
            parts = user_input.split()
            #字符串方法：split()，默认按空格切割字符串，把一行命令拆成列表，这是处理命令行参数的标准写法
            if len(parts) == 2 and parts[1] == "list":
            #确保命令是两段式（防止输入 /model 无参数）
            #parts[1] == "list"：第二个参数是 list
                list_models()
            elif len(parts) == 2 and parts[1].isdigit():
                #判断数字
                switch_model(int(parts[1]))
            else:
                print(" 命令格式错误！输入 /help 查看帮助")

        # 正常对话
        else:
            reply = call_ai(user_input)
            model = get_current_model()
            print(f"{model['name']} > {reply}\n")
            # 保存对话
            conversation_history.append({
                "time": datetime.now().strftime("%Y-%m-%d %H:%M"),
                "model": model['name'],
                "user": user_input,
                "ai": reply
            })
            save_history()
#文件接口
if __name__ == "__main__":
    main()
```
