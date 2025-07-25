#### **1. 核心架构：带权有向图**

**1.1. 数据结构**

*   **架构类型**: 带权有向图 (Weighted Directed Graph)。
*   **组件**:
    *   **节点 (Node)**: 代表记忆单元（概念、事物、事件、情境）。
    *   **边 (Edge)**: 代表节点间的有向关系。
*   **设计原理**: 模拟人脑记忆的**联想网络 (Associative Network)** 结构，其中记忆单元通过关系相互连接。

**1.2. 节点结构 (Node Structure)**

每个节点定义如下属性：

*   **ID**: 唯一的节点标识符。
*   **Type (节点类型)**: 节点的分类。
    *   `Detail`: 事实性、具体的信息 (e.g., "生日是10月26日")。
    *   `Concept`: 抽象概念 (e.g., "生日", "汽车")。
    *   `Context`: 用于组织和锚定相关记忆的元节点 (e.g., "朋友张三", "2025年夏天旅行")。
    *   **设计原理**: `Context`节点用于实现**情境依赖记忆 (Context-Dependent Memory)**，将记忆锚定在特定的时空或语义背景中。
*   **Content (内容层级)**: 分级存储，用于实现渐进式遗忘。
    *   `L3 (Full Detail)`: 完整信息。
    *   `L2 (Abstract)`: 摘要或关键词。
    *   `L1 (Pointer)`: 仅存的标题或存在性标识。
    *   **设计原理**: 模拟记忆衰退时从**具体细节到核心概要的遗忘过程 (Graceful Degradation)**，而非信息的瞬时完全丢失。
*   **Emotion Vector (情感向量)**: 基于VAD (Valence-Arousal-Dominance) 等模型的多维向量，量化记忆的情感属性。
    *   **设计原理**: 量化情感，为实现**闪光灯记忆 (Flashbulb Memory)** 和**情绪一致性记忆 (Mood-Congruent Memory)** 提供数据基础。
*   **Competition_Group_ID**: 可选标识符。同组节点在激活时相互抑制。
    *   **设计原理**: 实现**侧向抑制 (Lateral Inhibition)** 机制，用于在多个互斥选项中进行决策和消歧。
*   **Value (长期价值)**: 浮点数，表征记忆的长期重要性和稳固性。
    *   **计算公式**: `Value = w1*log(存在天数) + w2*回忆次数 + w3*高相关度连接数 + w4*情感强度`
    *   **情感强度 (Emotional Intensity)**: `唤醒度(A) + |愉悦度(V)|`
    *   **设计原理**: 模拟记忆的**长期强度 (Long-Term Memory Strength)**，其价值由多种因素决定，包括**情感显著性 (Emotional Salience)**对记忆巩固的加强作用。
*   **Activation Heat (短期热度)**: 临时浮点数，随时间快速衰减，表征近期激活水平。
    *   **设计原理**: 模拟记忆的**短期可及性 (Short-Term Accessibility)**，解释**近期效应 (Recency Effect)** 和**启动效应 (Priming Effect)**。
*   **Last Recalled Timestamp**: 时间戳，用于计算`Heat`衰减和遗忘。

**1.3. 边结构 (Edge Structure)**

每条边定义如下属性：

*   **Source / Target**: 源节点ID和目标节点ID。
*   **Relevance (相关度)**: 浮点数，范围[-1.0, 1.0]，定义连接的性质和强度。
    *   `> 0`: 兴奋性连接（激活）。
    *   `= 0`: 无直接关系。
    *   `< 0`: 抑制性连接（抑制）。
    *   **设计原理 (负权重)**: 建模概念间的静态对立关系（如“冷”与“热”），是知识结构稳定性的基础，此机制称为**直接抑制 (Direct Inhibition)**。
*   **Relation Type**: 关系的语义标签 (e.g., `is_a (是一种)`, `has_part (包含)`, `contradicts (矛盾)`, `cause (导致)` 等)。

#### **2. 动态机制**

**2.1. 记忆检索 (Search/Recall)**

一个分阶段的检索算法：
1.  **入口定位**: 根据查询关键词，通过索引定位一个或多个初始节点，并赋予初始激活能量。
2.  **激活扩散 (Spreading Activation)**: 激活能量从初始节点沿边扩散至邻近节点。能量传递量由边的`Relevance`值加权。源节点的`情感强度`可调节初始扩散能量。负`Relevance`传递抑制信号。
3.  **侧向抑制 (Lateral Inhibition)**: 若某节点的激活值超过预设阈值且其属于某个`Competition_Group_ID`，它将向该组内所有其他节点施加抑制信号，加速在互斥选项中的收敛。
4.  **结果排序**: 节点最终的“搜索激活值”与其自身的`Value`和`Heat`加权求和，得到最终排序分。得分最高的节点为检索结果。
*   **设计原理**: 整个过程基于**激活扩散理论 (Spreading Activation Theory)**，模拟了从线索到目标的联想回忆过程。

**2.2. 基于情感的查询与交互**

*   **情感查询**: 支持基于`Emotion Vector`的查询 (e.g., "检索愉悦度 > 0.8 且唤醒度 > 0.7 的记忆")。
*   **语气适配**: LLM可以解析检索结果的`Emotion Vector`，并相应调整其输出的语言风格。
*   **设计原理**: 应用**情绪一致性记忆 (Mood-Congruent Memory)** 原理，系统能够检索与特定情绪状态匹配的记忆，并基于此调整行为。

**2.3. 遗忘机制 (Forgetting)**

一个双重遗忘模型：
1.  **优雅降级 (Graceful Degradation)**: 基于节点的`Value`和`Last Recalled Timestamp`计算一个“保留率”。当保留率低于特定阈值时，节点`Content`从 L3 降级至 L2，再从 L2 降级至 L1。
2.  **完全遗忘 (Deletion)**: 当保留率低于最低阈值，或一个 L1 节点的`Value`持续低于某个阈值时，该节点及其关联的边将被从图中移除。
*   **设计原理**: 结合了基于**艾宾浩斯遗忘曲线 (Ebbinghaus Forgetting Curve)** 的时间衰退模型和基于重要性（`Value`）的保留机制，实现了受控的、从细节到概要的**渐进式遗忘**。

**2.4. 学习与巩固 (Learning & Consolidation)**

一个双重巩固过程：
*   **学习 (Learning)**: 向图中添加新的节点和边。创建时需指定其初始属性，如`Emotion Vector`、`Relevance`等。
*   **节点巩固 (Node Consolidation)**: 节点被成功检索后，其`回忆次数`增加，`Value`重新计算，`Heat`设为峰值，并更新`Last Recalled Timestamp`。
*   **关联巩固 (Associative Consolidation)**: 在一次成功的检索中，激活路径上的边的`Relevance`值会根据赫布学习规则得到加强。
*   **设计原理**: 关联巩固机制直接应用了**赫布理论 (Hebbian Theory)**（“共同激活的单元之间的连接会被强化”），通过经验来加强联想路径，形成思维习惯。

#### **3. 高阶认知功能**

通过对图结构的特定操作模式实现：

**3.1. 抽象化 (Abstraction)**

*   **向上追溯**: 沿`is_a`等层级关系边，从具体节点遍历至抽象的`Concept`节点。
*   **动态识别**: 通过算法分析一组节点的共同连接模式或属性，动态创建新的`Concept`节点来表征其共性。
*   **设计原理**: 基于**图式理论 (Schema Theory)** 和**概念层次 (Conceptual Hierarchies)**，通过对图结构的操作，实现从具体实例到抽象概念的归纳过程。

**3.2. 发散性思考 (Divergent Thinking)**

*   **约束下的随机游走**: 从一个起始节点出发，以边的`Relevance`和邻接节点的`Value`/`Heat`为概率进行加权随机遍历，探索相关但非直接的连接。
*   **情景模拟**: 临时在图中增删或修改节点/边，以代表一个“what-if”假设，然后运行激活扩散来探索该假设的潜在影响。
*   **类比推理**: 识别并匹配不同`Context`节点下子图的结构相似性，以实现跨领域的类比。
*   **设计原理**: 通过非确定性探索（随机游走）和结构模式匹配（类比推理）等算法，模拟创造力研究中的**发散思维 (Divergent Thinking)** 过程。

### **4. 关键设计原因与认知科学原理**

*   **图结构**: 实现**联想记忆 (Associative Memory)**。
*   **激活扩散**: 基于**激活扩散理论 (Spreading Activation Theory)**，模拟概念的联想激活。
*   **长期价值与短期热度**: 模拟**记忆强度 (Memory Strength)**、**近期效应 (Recency Effect)** 和**启动效应 (Priming Effect)**。
*   **闪光灯记忆 (Flashbulb Memory)**: 通过情感向量整合进长期价值，模拟高度情绪化事件形成的持久记忆。
*   **情绪一致性记忆 (Mood-Congruent Memory)**: 通过情感查询和受情感影响的激活扩散，模拟人更容易回忆起与当前情绪状态一致的记忆。
*   **遗忘机制**: 基于改进的**艾宾浩斯遗忘曲线**和**渐进式遗忘**理论，实现受重要性影响的、从细节到概要的优雅遗忘过程。
*   **情境节点**: 实现**情境依赖记忆 (Context-Dependent Memory)**。
*   **赫布理论 (Hebbian Theory)**: 通过加强共同激活路径上的边，模拟大脑通过经验强化神经通路，形成稳固联想和思维习惯的过程。
*   **直接抑制 (Direct Inhibition)**: 通过负权重边，精确建模两个特定记忆间的静态、语义对立关系（如逻辑矛盾），是知识结构稳定性的基础。
*   **侧向抑制 (Lateral Inhibition)**: 通过竞争组机制，实现“赢家通-吃”的动态决策模型，帮助系统在面对多个互斥选项时快速聚焦和解决歧义，是高效思维过程的关键。
*   **抽象化**: 基于**图式理论 (Schema Theory)** 和**概念层次 (Conceptual Hierarchies)**，模拟知识的组织与归纳。
*   **发散性思考**: 模拟创造力研究中的**发散思维**过程。