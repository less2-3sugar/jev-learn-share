<img width="2144" height="1271" alt="image" src="https://github.com/user-attachments/assets/68d3508d-ba0e-4d60-8d88-327ad1232448" />本次 Task 5，我在浏览器中运行智能家居 3D Demo，使用真实 Jev 处理文字指令。由于没有 STEPFUN_API_KEY，我没有把传统 LLM 和双路对比的报错当作实验数据，而是重点观察 Jev 的判断、程序派发的动作和设备状态变化。
我先输入“把全屋的灯都关掉”，五个房间的灯均关闭，页面显示耗时 1636 ms；随后输入“打开客厅的灯”，只有客厅灯开启，其余房间的灯保持关闭，耗时 3097 ms。两次结果都符合本次预期。追踪面板显示，Jev 会同时预判多类设备动作，程序只采纳与指令相关的 light_action，剪掉风扇、门锁等无关分支。
这让我理解了模型与程序的分工：Jev 提供结构化判断，程序根据目标范围和规则更新模拟设备。我也遇到过连接中断后自动降级为本地模拟，因此记录数据时需要核对每轮是否标明“真实 Jev”。两次耗时不同，说明少量运行结果不能代表普遍速度或准确率。若用于真实业务，我会让 Jev 负责识别智能家居指令，再由程序检查权限和设备状态后执行。

我的运行界面如下：

<img width="2144" height="1271" alt="把灯全部关掉" src="https://github.com/user-attachments/assets/37049c36-9f61-401c-9470-2bf88716e6e0" />
<img width="2145" height="1293" alt="把客厅的灯打开" src="https://github.com/user-attachments/assets/03fe2e16-3457-4dbe-bcd8-c98c2866e81a" />


