### **SKD 出货建议算法文档（修改版）**

---

## **1. 概述**

本算法旨在解决 SKD（半散件）需求的库存分配问题，相较于 CKD（全散件），进一步考虑库存仓库的优先级，严格遵循 BOM 结构的物料分配规则，并确保需求齐套，不允许超量出货。

每个 **site** 作为一个独立的出货单元，不同 **site** 之间的需求和库存不共享。  
通过合理分配库存，算法主要优化目标包括：

1. **最大化需求满足率**，确保尽可能多的需求被满足  
2. **优先满足较早周次的需求**，确保早期需求不会被后期需求挤占库存  
3. **优先使用入库较早的库存**，避免库存老化  

通过建立 **混合整数规划模型**，在满足约束条件的前提下，优化出库计划。

---

## **2. 问题描述**

### **库存类型**
库存分为以下几种类型：
- **小箱（Stock）**：不可拆分，需整箱出库  
- **大箱（Stock）**：由多个小箱组成，同一 `site` 内可共享  
- **Open PO**：未装箱库存，按量出库，仅包含散料  
- **Open DN**：待收货库存，可以预留但不能直接出库  
- **WIP（Work In Progress）**：生产中的库存，可用于补充需求  
- **Onway**：在途库存，预计到货但未到仓  

---

### **库存分配规则**
库存分配策略调整如下：

1. **需求匹配 `ship_location`（有 `shiploc`）**
   - **优先级顺序**：
     `kitting_location` → Open PO → Open DN → `ship_location` → WIP → Onway  

2. **需求无 `ship_location`（无 `shiploc`）**
   - **优先级顺序**：
     `kitting_location` → WIP → Onway  

3. **出库顺序规则**
   - **严格按照 BOM 结构优先级出库**：
     先出库半成品，再出库散料  

---

## **3. 数学模型**

### **3.1 符号说明**

| 符号                 | 描述                                                                           |
|--------------------|------------------------------------------------------------------------------|
| **【需求属性】**         |                                                                              |
| $i$                | 需求编号，$i=1,2,\cdots,I$                                                        |
| $Q_i$              | 需求 $i$ 的需求量                                                                  |
| $W_i$              | 需求 $i$ 的周次                                                                   |
| $B_i$              | 需求 $i$ 的 BOM 结构，$B_i = \{b_{i,1}, b_{i,2}, \dots\}$， $b_{i,j}$ 是一个物料节点       |
| $sl_i$             | 需求 $i$ 的 `ship_location`                                                     |
| $kl_i$             | 需求 $i$ 的 `kitting_location`                                                  |
| **【库存属性】**         |                                                                              |
| $s$                | 库存（小箱）编号，$s=1,2,\cdots,S$                                                 |
| $p_s$              | 库存 $s$ 的物料号                                                                  |
| $q_s$              | 库存 $s$ 的物料数量                                                                 |
| $d_s$              | 库存 $s$ 的入库日期                                                                 |
| $l_s$              | 库存 $s$ 的库位                                                                   |
| $m_s$              | 库存 $s$ 的合箱号，$m_s = c$ 表示 $s$ 属于大箱 $c$ （为空表示未合箱）                              |
| **【Open PO 属性】**   |                                                                              |
| $k$                | Open PO 编号, $k=1,2,\cdots,K$                                                 |
| $pn_k$             | Open PO $k$ 的物料编号                                                            |
| $q_k$              | Open PO $k$ 的物料数量                                                            |
| $l_k$              | Open PO $k$ 的 `kitting_location`                                                |
| **【其他库存来源】**       |                                                                              |
| $dn$               | Open DN 库存编号，$dn=1,2,\dots,DN$                                              |
| $w$                | WIP 库存编号，$w=1,2,\dots,W$                                                   |
| $ow$               | Onway 库存编号，$ow=1,2,\dots,OW$                                               |
| **【决策变量】**         |                                                                              |
| $x_{i,s}$          | $x_{i,s} \in \{0, 1\}$，是否使用小箱 $s$ 满足需求 $i$ ,$x_{i,s}= 1$ 表示使用，否则为 0         |
| $y_i$              | $y_i \in \{0, 1\}$，需求 $i$ 是否被满足，$y_i= 1$ 表示完全满足，否则为 0                    |
| $m_{i,k}$          | $m_{i,k} \in \mathbb{Z}$，Open PO $k$ 用于满足需求 $i$ 分配的物料数量，$m_{i,k}=0$ 表示未使用 |
| $m_{i,dn}, m_{i,w}, m_{i,ow}$ | 分别表示 Open DN、WIP、Onway 库存的分配量 |

---

### **3.2 约束条件**
1. **需求满足约束**
   - 需求必须齐套才允许出库，即要么完全满足，要么不满足：
     $$
     Supply_{i,0} = Q_i \cdot y_i \quad \forall i \in I
     $$

2. **BOM 结构约束**
   - **半成品库存优先**
     $$
     Supply_{i,j} = \sum_{p_s \in alt\_pn_{i,j}} x_{i,s} \cdot q_s + o_{i,j} \quad \forall i, j
     $$

   - **下层供应量计算**
     $$
     o_{i,j} = \frac{Supply_{i,k}}{unit\_qty_{i,k}} \quad \forall i, j ,parent\_pn_{i,k}=b_{i,j}
     $$

3. **库存来源优先级约束**
   - **优先使用 `kitting_location` 库存**
     $$
     m_{i,k} + m_{i,dn} + m_{i,w} + m_{i,ow} \leq Q_i
     $$

---

### **3.3 目标函数**
优化目标：
$$
\min \alpha_1 f_1 + \alpha_2 f_2 + \alpha_3 f_3 + f_{\text{priority}}
$$

1. **最大化需求满足率**
   $$
   f_1 = I-\sum_{i \in I} y_i
   $$

2. **最小化需求被满足的周次**
   $$
   f_2 = \sum_{i \in I} y_i \cdot W_i
   $$

3. **最小化出库物料的库存日期**
   $$
   f_3 = \sum_{s \in S} \sum_{i \in I} x_{i,s} \cdot d_s
   $$

4. **最小化仓库出库优先级惩罚**
   $$
   f_{\text{priority}} = \sum_{i \in I} \sum_{j \in B_i} \left(\sum_{s \in S} x_{i,s} \cdot loc_{priority}(l_s, sl_i, kl_i)\right)
   $$

权重设置：
- $f_1$: 100（高优先级）
- $f_2$: 10（中优先级）
- $f_3$: 1（低优先级）

---

## **4. 总结**
- **确保严格的库存使用顺序**，首先 `kitting_location`，然后 `ship_location`  
- **支持多种库存来源**（Stock, Open PO, Open DN, WIP, Onway）  
- **通过混合整数规划，实现最优库存分配**  

---
**注**：
1. 需求齐套的一个数量 = 一个机头（或机头展开一层的物料）+ 包材物料  
2. 库存分配具体到 BOM 层  
3. **为 CID 层级库存分组，减少变量规模**





### **SKD 出货建议算法文档（修改版）**

---

#### **1. 概述**

本算法旨在解决 SKD（半散件）需求的库存分配问题。
在 CKD（全散件）基础上，进一步考虑库存仓库的优先级，严格遵循 BOM 结构的物料分配规则，并确保需求齐套，不允许超量出货。
每个 site 作为一个独立的出货单元，不同 site 之间的需求和库存不共享。通过合理分配库存，算法主要优化目标包括：

1. 最大化需求满足率，确保尽可能多的需求被满足
2. 优先满足较早周次的需求，确保早期需求不会被后期需求挤占库存
3. 优先使用入库较早的库存，避免库存老化

通过建立混合整数规划模型，在满足约束条件的前提下，优化出库计划。

---

#### **2. 问题描述**

**库存类型**
• 库存分为小箱、大箱、Open PO、Open DN、WIP 和 Onway：
    ◦ **小箱**：不可拆分，需整箱出库
    ◦ **大箱**：由多个小箱组成，同一 site 内可共享
    ◦ **Open PO**：未装箱库存，按量出库，仅包含散料
    ◦ **Open DN**：在途库存，按量出库
    ◦ **WIP**：在制品库存，按量出库
    ◦ **Onway**：运输中库存，按量出库

**库存分配规则**
1. **kitting_location 有库存结构（箱子）**：
    ◦ `kitting_location` 的库存按箱子（小箱/大箱）分配，遵循 BOM 结构优先级出库（先出库半成品，再出库散料）。
2. **ship_location 只有料号+数量（Open PO）**：
    ◦ `ship_location` 的库存按数量分配，仅包含散料。

**分配优先级**
• 如果需求匹配 `ship_location`：
    ```
    kitting_location -> Open PO -> Open DN -> ship_location -> WIP -> Onway
    ```
• 如果需求不匹配 `ship_location`：
    ```
    kitting_location -> WIP -> Onway
    ```

---

#### **3. 数学模型**

##### 3.1 符号说明

| 符号                 | 描述                                                                           |
|--------------------|------------------------------------------------------------------------------|
| **【需求属性】**         |                                                                              |
| $i$                | 需求编号，$i=1,2,\cdots,I$                                                        |
| $Q_i$              | 需求 $i$ 的需求量                                                                  |
| $W_i$              | 需求 $i$ 的周次                                                                   |
| $B_i$              | 需求 $i$ 的 BOM 结构，$B_i = \{b_{i,1}, b_{i,2}, \dots\}$， $b_{i,j}$ 是一个物料节点       |
| $sl_i$             | 需求 $i$ 的 `ship_location`                                                        |
| $kl_i$             | 需求 $i$ 的 `kitting_location`                                                     |
| **【BOM属性】**        |                                                                              |
| $b_{i,j}$          | 需求 $i$ 的第 $j$ 个物料节点，引入一个虚拟成品节点 $b_{i,0}$                                     |
| $pn_{i,j}$         | 物料编号                                                                         |
| $parent\_pn_{i,j}$ | 父物料编号（为空表示最上层物料）                                                             |
| $unit\_qty_{i,j}$  | 单位用量，表示成品需要的当前物料数量                                                           |
| $alt\_pn_{i,j}$    | 主料∪替代物料列表                                                                    |
| **【库存属性】**         |                                                                              |
| $s$                | 库存(小箱)编号，$s=1,2,\cdots,S$                                                    |
| $p_s$              | 库存 $s$ 的物料号                                                                  |
| $q_s$              | 库存 $s$ 的物料数量                                                                 |
| $d_s$              | 库存 $s$ 的入库日期                                                                 |
| $l_s$              | 库存 $s$ 的库位                                                                   |
| $m_s$              | 库存 $s$ 的合箱号，$m_s = c$ 表示 $s$ 属于大箱 $c$ （为空表示未合箱）                              |
| **【Open PO 属性】**   |                                                                              |
| $k$                | Open PO 编号, $k=1,2,\cdots,K$                                                 |
| $pn_k$             | Open PO $k$ 的物料编号                                                            |
| $q_k$              | Open PO $k$ 的物料数量                                                            |
| $l_k$              | Open PO $k$ 的 `kitting_location`                                                |
| **【Open DN 属性】**   |                                                                              |
| $d$                | Open DN 编号, $d=1,2,\cdots,D$                                                 |
| $pn_d$             | Open DN $d$ 的物料编号                                                            |
| $q_d$              | Open DN $d$ 的物料数量                                                            |
| $l_d$              | Open DN $d$ 的 `kitting_location`                                                |
| **【WIP 属性】**      |                                                                              |
| $w$                | WIP 编号, $w=1,2,\cdots,W$                                                     |
| $pn_w$             | WIP $w$ 的物料编号                                                               |
| $q_w$              | WIP $w$ 的物料数量                                                               |
| $l_w$              | WIP $w$ 的 `kitting_location`                                                   |
| **【Onway 属性】**    |                                                                              |
| $o$                | Onway 编号, $o=1,2,\cdots,O$                                                   |
| $pn_o$             | Onway $o$ 的物料编号                                                             |
| $q_o$              | Onway $o$ 的物料数量                                                             |
| $l_o$              | Onway $o$ 的 `kitting_location`                                                 |
| **【决策变量】**         |                                                                              |
| $x_{i,s}$          | $x_{i,s} \in \{0, 1\}$,是否使用小箱子 $s$ 满足需求 $i$ ,$x_{i,s}= 1$ 表示使用，否则为 0         |
| $y_i$              | $y_i \in \{0, 1\}$,需求 $i$ 是否被满足  ,  $y_i= 1$ 表示完全满足，否则为 0                    |
| $z_{c}$            | $z_{c} \in \{0, 1\}$, 是否在出货单元中使用大箱 $c$, $z_{c} = 1$ 表示使用，否则为 0               |
| $m_{i,k}$          | $m_{i,k} \in \mathbb{Z} $  ,Open PO $k$ 用于满足需求 $i$ 分配的物料数量，$m_{i,k}=0$ 表示未使用 |
| $n_{i,d}$          | $n_{i,d} \in \mathbb{Z} $  ,Open DN $d$ 用于满足需求 $i$ 分配的物料数量，$n_{i,d}=0$ 表示未使用 |
| $p_{i,w}$          | $p_{i,w} \in \mathbb{Z} $  ,WIP $w$ 用于满足需求 $i$ 分配的物料数量，$p_{i,w}=0$ 表示未使用     |
| $q_{i,o}$          | $q_{i,o} \in \mathbb{Z} $  ,Onway $o$ 用于满足需求 $i$ 分配的物料数量，$q_{i,o}=0$ 表示未使用   |
| 【辅助变量】             |                                                                              |
| $  Supply_{i,j}$   | $Supply_{i,j}\in \mathbb{Z} $ ,bom 节点 $b_{i,j}$ 的实际出货量                       |
| $  o_{i,j}$        | $o_{i,j}\in \mathbb{Z} $,bom 节点 $b_{i,j}$ 的下层物料供应量                           |
| $l_{i,j}$          | $l_{i,j}\in \{0, 1\}$,bom 节点 $b_{i,j}$ 是否考虑下层散料供应                            |

##### 3.2 约束条件

1. **需求满足约束**：
    ◦ 需求必须齐套才允许出库，即要么完全满足，要么不满足
      $$
      Supply_{i,0} = Q_i \cdot y_i\quad \forall i \in I
      $$

2. **Bom 结构约束**：
    ◦ 所有 BOM 物料都齐套时，需求才会被满足
    ◦ 对于有下层展开的物料节点：
      $$
      Supply_{i,j} = \sum_{p_s \in alt\_pn_{i,j}} x_{i,s} \cdot q_s + o_{i,j} \quad \forall i, j
      $$
      其中，下层供应量= 下层子物料的供应量/单位用量
      $$
      o_{i,j} = \frac{Supply_{i,k}}{unit\_qty_{i,k}} \quad \forall i, j ,parent\_pn_{i,k}=b_{i,j}
      $$
    ◦ 对于无下层展开的物料节点：
      $$
      Supply_{i,j} = \sum_{p_s \in alt\_pn_{i,j}} x_{i,s} \cdot q_s + \sum_{pn_k \in alt\_pn_{i,j} }m_{i,k} + \sum_{pn_d \in alt\_pn_{i,j} }n_{i,d} + \sum_{pn_w \in alt\_pn_{i,j} }p_{i,w} + \sum_{pn_o \in alt\_pn_{i,j} }q_{i,o} \quad \forall i, j
      $$

3. **Bom 库存使用顺序约束**：
    ◦ 半成品库存优先，库存不足时才向下层展开
    ◦ 对于有下层展开的物料节点，`kitting_location` 的库存必须优先分配，再考虑下层供应：
      $$
      o_{i,j} \leq M \cdot l_{i,j} \quad \forall i, j
      $$
      $$
      Supply_{i,j} \cdot (1-l_{i,j}  )\leq \sum_{p_s \in alt\_pn_{i,j}} x_{i,s} \cdot q_s\quad \forall i, j
      $$

4. **小箱出库约束**：
    ◦ 每个小箱 $s$ 只能分配给一个需求
      $$
      \sum_{i=1}^{I} x_{i,s} \leq 1 \quad \forall s \in  S
      $$

5. **大箱出库约束**：
    ◦ 大箱出库时，所有小箱必须一起分配
      $$
      x_{i,s} = z_{u,c} \quad \forall c,m_s = c,i \in I
      $$

6. **Open PO 使用约束**：
    ◦ 每个 Open PO $k$ 的分配量不能超过其库存量
      $$
      \sum_{i=1}^{I} m_{i,k} \leq q_k \quad \forall k \in K
      $$

7. **Open DN 使用约束**：
    ◦ 每个 Open DN $d$ 的分配量不能超过其库存量
      $$
      \sum_{i=1}^{I} n_{i,d} \leq q_d \quad \forall d \in D
      $$

8. **WIP 使用约束**：
    ◦ 每个 WIP $w$ 的分配量不能超过其库存量
      $$
      \sum_{i=1}^{I} p_{i,w} \leq q_w \quad \forall w \in W
      $$

9. **Onway 使用约束**：
    ◦ 每个 Onway $o$ 的分配量不能超过其库存量
      $$
      \sum_{i=1}^{I} q_{i,o} \leq q_o \quad \forall o \in O
      $$

##### 3.3 目标函数

采用加权多目标优化，将各目标函数通过加权求和转化为单个优化目标求解

$$
\min \alpha_1 f_1 + \alpha_2 f_2 + \alpha_3 f_3 + f_{priority }
$$

1. **最大化需求的满足数量**：
   优先满足更多的需求,最小化未满足的需求量
   $$
   f_1 = I-\sum_{i \in I} y_i
   $$

2. **最小化需求被满足的周次**：
   优先满足较早周次的需求
   $$
   f_2 = \sum_{i \in I} y_i \cdot W_i
   $$

3. **最小化出库物料的库存日期**：
   优先使用较早的库存
   $$
   f_3 = \sum_{s \in S} \sum_{i \in I} x_{i,s} \cdot d_s
   $$

4. **最小化仓库出库优先级惩罚**：
   • 对于每个物料，如果 `kitting_location` 不存在可用库存，才考虑 `ship_location` 的库存。添加惩罚项，如果违反顺序，目标函数值会大幅增大。
   • 计算出库优先级惩罚项：
     $$
     f_{priority }= \sum_{i \in I} \sum_{j \in B_i} \left(\sum_{s \in S} x_{i,s} \cdot loc_{priority}(l_s, sl_i, kl_i) )+ \sum_{k \in K} \frac{m_{i,k}}{q_k}\cdot loc_{priority}(l_k, sl_i, kl_i) )\right)
     $$

**权重设置**
• 权重系数的设置是多目标优化中的关键环节，需要根据数据规模动态调整
• 归一化后权重：
  • $f_1$（未满足需求）：100
  • $f_2$（需求周次）：10
  • $f_3$（库存日期）：1

---

#### **4. 总结**

通过为库存库位和 BOM 结构中的物料节点分配合理的优先级，确保严格的库存使用顺序控制。
通过建立混合整数规划模型，算法能够在满足约束条件的前提下，最大化需求满足率，并优先使用早期库存和较早周次的需求。

#### **注：**
1. 需求齐套的一个数量等于一个机头（或机头展开一层的物料）+ 包材物料，只有 `kitting_location` 中存在半成品。
2. 库存分配具体到 BOM 层。
3. 为 `cid` 层级库存分组，减少变量规模。

---



以下是针对 **目标4** 的优先级函数的优化版本，确保逻辑清晰、表达准确，并突出优先级规则：

---

#### **目标4：最小化仓库出库优先级惩罚**

**优先级规则**  
对于每个物料，如果 `kitting_location` 不存在可用库存，才依次按照以下顺序考虑其他库位的库存：  
`ship_location` → `wip_location` → `onway_location`  
如果违反此顺序，目标函数值会大幅增大，以确保优先级规则被严格遵循。

**优先级惩罚项**  
计算出库优先级惩罚项，公式如下：  
$$
f_{priority} = \sum_{i \in I} \sum_{j \in B_i} \left( \sum_{s \in S} x_{i,s} \cdot loc_{priority}(l_s, kl_i, sl_i, wl_i, ol_i) + \sum_{k \in K} \frac{m_{i,k}}{q_k} \cdot loc_{priority}(l_k, kl_i, sl_i, wl_i, ol_i) \right)
$$

**优先级函数**  
$loc_{priority}(l_s, kl_i, sl_i, wl_i, ol_i)$ 是库位匹配优先级，定义如下：  
$$
loc_{priority}(l_s, kl_i, sl_i, wl_i, ol_i) =
\begin{cases}
1 & \text{如果 } l_s = kl_i \text{ 且库存来源为箱子} \\
100 & \text{如果 } l_s = kl_i \text{ 且库存来源为 Open PO} \\
1000 & \text{如果 } l_s = sl_i \\
10000 & \text{如果 } l_s = wl_i \\
100000 & \text{如果 } l_s = ol_i \\
\end{cases}
$$

**解释**  
1. **`kitting_location` 优先级最高**：  
   • 如果库存来自 `kitting_location`，优先使用箱子（惩罚值 = 1），其次是 Open PO（惩罚值 = 100）。
2. **其他库位优先级依次降低**：  
   • `ship_location` 的惩罚值为 1000，`wip_location` 为 10000，`onway_location` 为 100000。  
   • 惩罚值越大，表示优先级越低，违反规则时目标函数值会显著增加。

---

**修改点说明**  
1. **优先级规则更清晰**：  
   • 明确优先级顺序为 `kitting_location` → `ship_location` → `wip_location` → `onway_location`。  
   • 强调 `kitting_location` 的箱子优先于 Open PO。  
2. **惩罚值梯度更合理**：  
   • 惩罚值按优先级顺序递增，确保违反规则时目标函数值显著增大。  
3. **公式表达更简洁**：  
   • 使用统一的优先级函数 $loc_{priority}$，减少冗余代码，便于维护和扩展。

---

通过以上优化，目标4的优先级函数逻辑更加清晰，惩罚值梯度合理，能够有效确保库存分配严格遵循优先级规则。