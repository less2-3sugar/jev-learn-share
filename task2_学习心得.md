Task2 学习心得 - JEV Cookbook

通过本次学习，我最终自己构建了一个客服工单系统，并且跑通了。以下是我的一些理解：

1. 核心概念理解
SystemOne包含了state（证据）和一些问题（这些问题需要具有原子性），并且还需要判断每个问题属于Choice、Noul、Score中的哪个类型。

2. 逻辑判断
更具体SystemOne的输出进行一些判断（比如：使用if-else语句）。

3. 关键指标理解（最值得提及的）
Probability：每个情况的概率分布。
Confidence：结果分布是否集中。
Accuracy：结果的正确性。
在Score的结果中，confidence即使相同，但是我们还需要根据probability分布来进行具体分析。



客服工单系统代码如下：
from typesafe_sdk import TypeSafeClient, Choice, Noul, Score

client = TypeSafeClient(model="jev-1.13.0")

state = {
    "message":"我的订单被扣了两次钱，我想退掉其中一笔，而且这个问题已经影响我今天的使用了！",
    "order-id":"A104",
    "order_amount":"56",
    "refund-policy":"重复扣款可退掉重复付的款"
}

result = client.system_one(
    state=state,
    questions={
        "department":Choice(
            instructions="这个问题应该由哪个部门来处理？",
            criteria={
                "billing":"负责退款业务",
                "technical":"负责程序的维护",
                "account":"负责账户的维护",
                "other":"负责其他问题的维护"
            }
        ),
        "refund":Noul(
            instructions="是否同意退款？"
        ),
        "urgency":Score(
            instructions="这个用户的请求的紧急程度",
            criteria=[
                "很紧急，30分钟内必须回复",
                "稍微紧急，3小时内必须回复",
                "中等程度，半天内回复即可",
                "可以暂缓，一天回复即可"
            ]
        )
    }
)
print(result)

refund_probability=result.answers["refund"].noul
if refund_probability>0.9:
    print("同意退款")
else:
    print("转人工，进行审核")

执行结果如下：
department:
billing
confidence=1.0
probabilities={
    billing: 1.0,
    technical: 0.0,
    account: 0.0,
    other: 0.0
}

refund:
noul=0.9

urgency:
score=0.56
probabilities={
    0: 0.47,
    1: 0.50,
    2: 0.03,
    3: 0.00
}
