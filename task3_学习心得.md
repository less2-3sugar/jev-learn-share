# Task3 学习心得 - JEV Cookbook

通过本次学习，我运行了第三章《架构模式》的 Notebook，初步理解了推测性扇出、置信度门控、复合评分和意图路由四种模式。同时，我还尝试修改了意图路由部分的代码，添加新的用户消息并成功运行。以下是我的一些理解：

1. **推测性扇出**  
   在一次 System One 请求中可以同时提出多个相互独立的问题，提前获得可能需要的判断结果，后续再通过 Python 代码选择需要使用的答案。这样可以减少多次调用 API 带来的开销。

2. **置信度门控**  
   不能只根据模型给出的分类结果直接执行操作，还需要结合 Confidence 判断结果是否足够确定。例如，可以设置不同的阈值，让高置信度的请求进入后续流程，而低置信度的请求转人工审核。不过，置信度高并不代表结果一定正确。

3. **复合评分**  
   对于一个比较复杂的评价任务，可以拆分成多个独立的 Score 问题，然后通过 Python 对各项评分进行归一化和加权求和。我理解到，即使模型给出的各项评分不变，只要改变权重，最终的综合评分和排序也可能发生变化。

4. **意图路由（本次重点实践）**  
   我在 Notebook 原有代码的基础上，新增了“请帮我播放一首音乐”这条消息，并在 Choice 中添加了 `other` 类别。通过真实 API 调用，JEV 成功将其识别为 `other`，最后由 Python 路由到人工客服。在修改过程中，我还发现 `zip()` 会按照较短列表的长度进行遍历，因此需要注意测试消息和离线示例数据的数量。

**总结：** 第二章主要学习 JEV 如何进行类型化判断，而第三章更侧重于如何将这些判断结果应用到实际程序中。我认为最重要的一点是：**JEV 负责语义判断，Python 负责控制业务流程。**

以下是我修改的代码：
SUPPORT_MESSAGES = [
    "我的订单 #12345 到哪了？三天前就显示发货了。",
    "X 型号的意式咖啡机附赠不锈钢拉花缸吗？",
    "那双跑鞋我想退货，偏小半码。",
    ("这是第三次坏了，客服一直互相推。我要全额退款，让你们经理给我回电话，"
     "不然我就向银行发起拒付了。"),
    "。。。",
    "请帮我播放一首音乐",  #此处增加了一条信息
]

ROUTER_OFFLINE.append(
    dict(intent=("other", 0.90), complexity=0.0)
)  #为了让新增的一条信息可以执行得到结果，在ROUTER_OFFLINE中新增了一条
......
def route(msg, answers):
    intent, complexity = answers["intent"], answers["complexity"]

    if intent.confidence < 0.5:
        return human_agent(msg)
    # if intent.choice == "order_status":
    #     return deterministic_order_lookup(msg)
    if intent.choice == "product_question":
        return specialist_llm("产品", msg)
    if intent.choice == "return_exchange":
        return specialist_llm("退货", msg)
    if intent.choice == "other":
        return human_agent(msg)
    # complaint：复杂度高于"多步流程"或置信度不足时升级人工
    if complexity.score > 1 or complexity.confidence < 0.5:
        return human_agent(msg)
    return specialist_llm("投诉处理", msg)

for msg, off in zip(SUPPORT_MESSAGES, ROUTER_OFFLINE):
    resp = ts.call(msg, ROUTER_QUESTIONS, offline_answers={
        "intent": _FakeAnswer("choice", choice=off["intent"][0], confidence=off["intent"][1],
            probabilities={}),
        "complexity": _FakeAnswer("score", score=off["complexity"], confidence=0.85,
            probabilities={}, legend={0: "简单", 1: "多步", 2: "需升级"}),
    })
    intent, complexity = resp.answers["intent"], resp.answers["complexity"]
    shown = msg if len(msg) <= 40 else msg[:40] + "…"
    print(f"「{shown}」")
    print(f"   intent={intent.choice:<16} conf={intent.confidence:.2f}   complexity={complexity.score:.2f}")
    print(f"   → {route(msg, resp.answers)}")
    print()


执行结果如下：
「我的订单 #12345 到哪了？三天前就显示发货了。」
   intent=order_status     conf=1.00   complexity=0.19
   → [投诉处理 专家 LLM] 生成针对性回复…

「X 型号的意式咖啡机附赠不锈钢拉花缸吗？」
   intent=product_question conf=1.00   complexity=0.02
   → [产品 专家 LLM] 生成针对性回复…

「那双跑鞋我想退货，偏小半码。」
   intent=return_exchange  conf=1.00   complexity=0.07
   → [退货 专家 LLM] 生成针对性回复…

「这是第三次坏了，客服一直互相推。我要全额退款，让你们经理给我回电话，不然我就向银…」
   intent=complaint        conf=0.95   complexity=1.99
   → [人工客服] 创建工单并排队

「。。。」
   intent=other            conf=1.00   complexity=0.39
   → [人工客服] 创建工单并排队

「请帮我播放一首音乐」
   intent=other            conf=1.00   complexity=0.22
   → [人工客服] 创建工单并排队
