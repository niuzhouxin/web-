## 环境准备
### 安装依赖库
首先，请确保你已经安装了 `openai` 库用于与大语言模型交互，以及 `python-dotenv` 库用于安全地管理我们的 API 密钥。
在你的终端中运行以下命令：
```
pip install openai python-dotenv
```
### 配置api密钥
为了让我们的代码更通用，我们将模型服务的相关信息（模型ID、API密钥、服务地址）统一配置在环境变量中，(不建议配置在系统环境变量里)
1. 在你的项目根目录下，创建一个名为 `.env` 的文件。
2. 在该文件中，添加以下内容。你可以根据自己的需要，将其指向 OpenAI 官方服务，或任何兼容 OpenAI 接口的本地/第三方服务。
```
# .env file  
# Tavily API 配置  
TAVILY_API_KEY=xxx
# 大语言模型 API 配置  
# 选项一：AIHubmix  
OPENAI_API_KEY=xxx 
OPENAI_BASE_URL=https://aihubmix.com/v1  
MODEL_NAME=coding-glm-4.7-free
```
### 封装基础LLM调用函数
为了让代码结构更清晰、更易于复用，我们来定义一个专属的LLM客户端类。这个类将封装所有与模型服务交互的细节，让我们的主逻辑可以更专注于智能体的构建。

## 使用langGraph的基础组件
完整的工作流代码
```python
from typing import TypedDict, Annotated  
  
import dotenv  
from langchain_openai import ChatOpenAI  
from langgraph.graph import START, END  
from langgraph.graph import StateGraph  
from langgraph.graph.message import add_messages  
  
dotenv.load_dotenv()  
  
  
class GraphState(TypedDict):  
    """图状态数据结构，类型为字典，这个类可以理解为全局上下文对象"""  
    messages: Annotated[list, add_messages]  # 设置消息列表，并添加归纳函数  
    node_name: str  
  
# 修改 llm 定义部分  
llm = ChatOpenAI(  
    model="deepseek-chat",  
    openai_api_key="sk-936a9839913243aa94456167e9cc7846",  
    openai_api_base="https://api.deepseek.com",  
) #模型信息  
  
  
def chatbot(state: GraphState) -> GraphState:  
    """聊天机器人函数"""  
    # 1.获取状态里存储的消息列表数据并传递给LLM  
    ai_message = llm.invoke(state["messages"])  
    # 2.返回更新/生成的状态  
    return {"messages": [ai_message], "node_name": "chatbot"}  
  
  
# 1.创建状态图，并使用GraphState作为状态数据  
graph_builder = StateGraph(GraphState)  
  
# 2.添加节点  
graph_builder.add_node("llm", chatbot)  
  
# 3.添加边  
graph_builder.add_edge(START, "llm")  
graph_builder.add_edge("llm", END)  
  
# 4.编译图为Runnable可运行组件  
graph = graph_builder.compile()  
  
# 5.调用图架构应用  
print(graph.invoke({"messages": [("human", "你好，你是？")], "node_name": "graph"}))
```
## 工具调用
```python
import json  
from typing import TypedDict, Annotated, Any, Literal  
  
import dotenv  
from langchain_community.tools import GoogleSerperRun  
from langchain_community.utilities import GoogleSerperAPIWrapper  
from langchain_core.messages import ToolMessage  
from langchain_core.runnables import RunnableConfig  
from pydantic import BaseModel, Field  
from langchain_openai import ChatOpenAI  
from langgraph.graph import START, END  
from langgraph.graph import StateGraph  
from langgraph.graph.message import add_messages  
  
dotenv.load_dotenv()  
  
  
class GoogleSerperArgsSchema(BaseModel):  
   query: str = Field(description="执行谷歌搜索的查询语句")  
  
  
# 1.定义工具与工具列表  
google_serper = GoogleSerperRun(  
   name="google_serper",  
   description=(  
       "一个低成本的谷歌搜索API。"  
       "当你需要回答有关时事的问题时，可以调用该工具。"  
       "该工具的输入是搜索查询语句。"  
   ),  
   args_schema=GoogleSerperArgsSchema,  
   api_wrapper=GoogleSerperAPIWrapper(),  
)  
  
  
class State(TypedDict):  
   """图状态数据结构，类型为字典"""  
   messages: Annotated[list, add_messages]  
  
  
tools = [google_serper]  
llm = ChatOpenAI(  
    model="deepseek-chat",  
    openai_api_key="sk-936a9839913243aa94456167e9cc7846",  
    openai_api_base="https://api.deepseek.com",  
    request_timeout=120,  # 超时设置，避免无限等待  
)  
llm_with_tools = llm.bind_tools(tools)  
  
  
def chatbot(state: State, config: RunnableConfig | None = None) -> Any:  
   """聊天机器人函数"""  
   # 1.获取状态里存储的消息列表数据并传递给LLM  
   ai_message = llm_with_tools.invoke(state["messages"])  
   # 2.返回更新/生成的状态  
   return {"messages": [ai_message]}  
  
  
def tool_executor(state: State, config: RunnableConfig | None = None) -> Any:  
   """工具调用执行节点"""  
   # 1.构建工具名字映射字典  
   tools_by_name = {tool.name: tool for tool in tools}  
  
   # 2.提取最后一条消息里的工具调用信息  
   tool_calls = state["messages"][-1].tool_calls  
  
   # 3.循环遍历执行工具  
   messages = []  
   for tool_call in tool_calls:  
       # 4.获取需要执行的工具  
       tool = tools_by_name[tool_call["name"]]  
       # 5.执行工具并将工具结果添加到消息列表中  
       messages.append(ToolMessage(  
           tool_call_id=tool_call["id"],  
           content=json.dumps(tool.invoke(tool_call["args"])),  
           name=tool_call["name"]  
       ))  
  
   # 6.返回更新的状态信息  
   return {"messages": messages}  
  
  
def route(state: State, config: RunnableConfig | None = None) -> Literal["tool_executor", "__end__"]:  
   """动态选择工具执行亦或者结束"""  
   # 1.获取生成的最后一条消息  
   ai_message = state["messages"][-1]  
   # 2.检测消息是否存在tool_calls参数，如果是则执行`工具路由`  
   if hasattr(ai_message, "tool_calls") and len(ai_message.tool_calls) > 0:  
       return "tool_executor"  
   # 3.否则生成的内容是文本信息，则跳转到结束路由  
   return END  
  
  
# 1.创建状态图，并使用GraphState作为状态数据  
graph_builder = StateGraph(State)  
  
# 2.添加节点  
graph_builder.add_node("llm", chatbot)  
graph_builder.add_node("tool_executor", tool_executor)  
  
# 3.添加边  
graph_builder.add_edge(START, "llm")  
graph_builder.add_edge("tool_executor", "llm")  
graph_builder.add_conditional_edges("llm", route)  
  
# 4.编译图为Runnable可运行组件  
graph = graph_builder.compile()  
  
# 5.调用图架构应用  
print("开始执行...")  
state = graph.invoke(  
    {"messages": [("human", "现在的黄金和白银价格？")]},  
    config={"recursion_limit": 25},  
)  
  
# 只打印最终 AI 回复  
for message in reversed(state.get("messages", [])):  
    if message.type == "ai" and (not hasattr(message, "tool_calls") or not message.tool_calls):  
        print("\n" + "=" * 50)  
        print("最终回复：")  
        print("=" * 50)  
        print(message.content)  
        break
```

这样就可以回答人类的问题，




















## 参考
https://datawhalechina.github.io/hello-agents/#/./README
https://github.com/ignite0522/CTFAgent?tab=readme-ov-file