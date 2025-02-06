### CKD出货建议算法文档

#### 1. 概述

本算法旨在解决 CKD (全散件)需求的库存分配问题，通过合理分配库存，最大化满足需求量，最小化超出量，并综合考虑库存时间优先级和混箱约束的限制。
主要优化目标包括：

1. **最大化需求满足率**

2. **最小化大箱存在混箱PN情况下出货不匹配PN的数量**

3. **最小化超出物料分配量**

4. **优先使用早期库存**

5. **优先满足较早周次的需求**

在建模中，需求以出货单元（site）进行分组。大箱不能拆箱分配，但同一出货单元内的需求可以共享大箱。
通过建立混合整数规划模型，在满足约束条件的前提下，优化出库计划。

#### 2. 符号说明

| 符号           | 描述                                                                      |
|:-------------|:------------------------------------------------------------------------|
| **【需求属性】**   |                                                                         |
| $i$          | 需求编号，$i=1,2,\cdots,I$                                                   |
| $H_i$        | 需求 $i$ 的料号及替代组，$H_i = \{h_1, h_2, \cdots, h_n\}$                        |
| $Q_i$        | 需求 $i$ 的物料需求量                                                           |
| $W_i$        | 需求 $i$ 的周次                                                              |
| **【site属性】** |                                                                         |
| $u$          | 出货单元编号，$u=1,2,\cdots,U$                                                 |
| $D_u$        | 出货单元 $u$ 的需求集合，$D_u = \{i_1, i_2, \cdots, i_m\}$                        |
| **【库存属性】**   |                                                                         |
| $s$          | 库存（小箱）编号，$s=1,2,\cdots,S$                                               |
| $p_s$        | 小箱 $s$ 的料号                                                              |
| $q_s$        | 小箱 $s$ 的物料数量                                                            |
| $d_s$        | 小箱 $s$ 的入库日期                                                            |
| $m_s$        | 小箱 $s$ 的合箱号,$m_s = c$ 表示 $s$ 属于大箱 $c$（为空表示未合箱）                          |
| $c$          | 大箱编号，$c=1,2,\cdots,C$                                                   |
| **【决策变量】**   |                                                                         |
| $x_{i,s}$    | $x_{i,s} \in \{0, 1\}$, 是否使用小箱子 $s$ 满足需求 $i$ ,$x_{i,s}= 1$ 表示使用，否则为 0   |
| $y_i$        | $y_i \in \{0, 1\}$, 需求 $i$ 是否被满足  ,  $y_i= 1$ 表示满足，否则为 0                |
| $z_{u,c}$    | $z_{u,c} \in \{0, 1\}$, 是否在出货单元 $u$ 中使用大箱 $c$, $z_{u,c} = 1$ 表示使用，否则为 0 |

#### 3. 数学模型

##### 3.1 约束条件

1. **需求满足约束**：

    * 每个需求 $i$ 的替代组料号 $H_i$ 中的出货总量必须满足其需求量 $Q_i$

      $$
      \sum_{s \in S: p_s \in H_i} x_{i,s} \cdot q_s \geq y_i \cdot Q_i \quad \forall i \in I
      $$

    * 如果需求 $i $ 不能被满足，则不分配任何库存给该需求

      $$
      x_{i,s} \leq y_i \quad \forall i \in I, s \in S
      $$

2. **小箱库存使用约束**：

    * 每个小箱 $s$ 只能分配给一个需求

      $$
      \sum_{i \in I} x_{i,s} \leq 1 \quad \forall s \in S
      $$

3. **大箱出库约束**：

    * 每个大箱子 $c$ 只能分配给一个出货单元

      $$
      \sum_{u \in U} z_{u,c} \leq 1 \quad \forall c \in C
      $$

    * 如果小箱 $s$ 属于大箱 $c$，且分配给需求 $i$，则该大箱 $c$ 必须分配给需求 $i$ 所在的出货单元$u$

      $$
      x_{i,s} \leq z_{u,c} \quad \forall u \in U, i \in D_u, s \in S, m_s = c
      $$

    * 如果大箱 $c$ 被分配给出货单元 $u$，则大箱 $c$ 内的小箱不能分配到其他的出货单元

   $$
   \sum_{u' \in U, u' \neq u} \sum_{i' \in D_{u'}} x_{i',s} = 0 \quad \forall s \in S, m_{s'} = c
   $$

##### 3.2 目标函数

采用加权多目标优化，将各目标函数通过加权求和转化为单个优化目标：

$$
\max \left( \alpha_1 f_1 - \alpha_2 f_2 - \alpha_3 f_3 - \alpha_4f_4 - \alpha_5f_5 \right)
$$

其中：

* $\alpha_1$ 是最大化需求满足数量的权重

* $\alpha_2$ 是最小化混箱PN超出类型量的权重

* $\alpha_3$ 是最小化物料超出量的权重

* $\alpha_4$ 是最小化出库物料库存日期的权重

* $\alpha_5$ 是最小化需求被满足的周次的权重

1. **最大化需求的满足数量**：
   优先满足更多的需求

   $$
   f_1 = \sum_{i \in I} y_i
   $$

2. **最小化混箱PN超出类型量**：
   减少分配给出货单元的大箱中不属于该出货单元的需求料号集合的料号数量

   $$
   f_2 = \sum_{u \in U} \sum_{c \in C} e_{u,c}
   $$

   其中，$e_{u,c}$ 表示大箱 $c$ 在出货单元 $u$ 中的小箱PN比需求PN超出的类型数量：

   $$
   e_{u,c} \geq \sum_{s \in S: m_s = c, p_s \notin \bigcup_{i \in D_u} H_i} z_{u,c} \quad \forall u \in U, c \in C
   $$

3. **最小化物料超出量**：
   减小分配的出货量超出需求量的物料数量
   $$
   f_3 =  \sum_{u \in U} o_u
   $$

   其中，$o_u$ 为一个site下的超出量，由该site不合箱的小箱出货量 + 该site的大箱出货量- 该site满足的需求量

$$
o_u = \left( \sum_{s \in S: m_s = \emptyset} \sum_{i \in D_u} x_{i,s} \cdot q_s + \sum_{c \in C} z_{u,c} \cdot \sum_{s \in S: m_s = c} q_s \right) - \sum_{i \in D_u} y_i \cdot Q_i \quad \forall u \in U
$$
4. **最小化出库物料的库存日期**：
   优先使用较早的库存

   $$
   f_4 = \sum_{s \in S} \sum_{i \in I} x_{i,s} \cdot d_s
   $$

5. **最小化需求被满足的周次**：
   优先满足较早周次的需求

   $$
   f_5 = \sum_{i \in I} y_i \cdot W_i
   $$

#### 4. 总结

本算法通过混合整数规划模型，将需求、库存和装箱约束全面建模，结合多目标优化方法，实现了需求满足率、库存分配、混箱约束的全局优化。

#### 注：

1. 一个site内需求可以共享大箱，大箱的库存分配不直接关联到需求上，只能关联到site层 ，需求层按小箱分配

2. 超出量的计算方式： 按site，该site超出量 = 该site不合箱的小箱出货量+大箱出货量 - 该site满足的需求量

3. 混箱PN超出类型量的计算方式：按site ，该site大箱的PN超出类型量 = 该site大箱的PN - 属于该site的所有需求料号集合的PN







```java


package cn.hgplan.domain.ckd;

import cn.hgplan.controller.dto.ckd.request.CKDSiteDemand;
import cn.hgplan.controller.dto.ckd.request.CKDSiteRequest;
import cn.hgplan.controller.dto.ckd.request.CKDSiteStock;
import com.google.ortools.linearsolver.MPSolver;
import com.google.ortools.linearsolver.MPVariable;

import java.util.ArrayList;
import java.util.List;

public class CKDModel {

    // 出货单元
    public static class SiteManager {
        String siteId; // 出货单元编号
        List<Demand> demands; // 需求列表
        List<Stock> stocks; // 小箱库存列表
        List<MergeStock> mergeStocks; // 大箱库存列表
        MPVariable[][] x; // 决策变量：小箱是否满足需求 (x[i][s])
        MPVariable[] y;    // 决策变量：需求是否被满足 (y[i])
        MPVariable[] z;    // 决策变量：大箱是否使用 (z[c])

        public SiteManager(String siteId) {
            this.siteId = siteId;
            this.demands = new ArrayList<>();
            this.stocks = new ArrayList<>();
            this.mergeStocks = new ArrayList<>();
        }

        // 从CKDSiteRequest初始化出货单元
        public SiteManager(CKDSiteRequest siteRequest, MPSolver solver) {
            this(siteRequest.getSite());
            initializeDemands(siteRequest.getDemands(), solver);
            initializeStocks(siteRequest.getStock(), solver);
        }

        private void initializeDemands(List<CKDSiteDemand> siteDemands, MPSolver solver) {
            for (CKDSiteDemand demand : siteDemands) {
                addDemand(new Demand(demand, solver));
            }
        }

        private void initializeStocks(List<CKDSiteStock> siteStocks, MPSolver solver) {
            for (CKDSiteStock stock : siteStocks) {
                addStock(new Stock(stock));
            }
            handleMergeStock(siteStocks, solver); // 大箱
        }

        private void handleMergeStock(List<CKDSiteStock> siteStocks, MPSolver solver) {
            for (CKDSiteStock stock : siteStocks) {
                if (stock.getMergeCartonId() != null) {
                    MergeStock mergeStock = findMergeStock(stock.getMergeCartonId());
                    if (mergeStock == null) {
                        mergeStock = new MergeStock(stock.getMergeCartonId(), new ArrayList<>(), solver);
                        addMergeStock(mergeStock);
                    }
                    mergeStock.addSmallBox(new Stock(stock));
                }
            }
        }

        // 查找大箱ID是否存在
        private MergeStock findMergeStock(String mergeCartonId) {
            for (MergeStock mergeStock : mergeStocks) {
                if (mergeStock.getMergeCartonId().equals(mergeCartonId)) {
                    return mergeStock;
                }
            }
            return null;
        }

        public void addDemand(Demand demand) {
            this.demands.add(demand);
        }

        public void addStock(Stock stock) {
            this.stocks.add(stock);
        }

        public void addMergeStock(MergeStock mergeStock) {
            this.mergeStocks.add(mergeStock);
        }

        // 在构造函数中初始化决策变量
        public void initializeVariables(MPSolver solver) {
            int demandCount = demands.size();
            int stockCount = stocks.size();
            int mergeStockCount = mergeStocks.size();

            // 初始化决策变量 x, y, z
            this.x = new MPVariable[demandCount][stockCount];
            this.y = new MPVariable[demandCount];
            this.z = new MPVariable[mergeStockCount];

            // 初始化决策变量 x[i][s]：每个需求 i 是否通过库存 s 满足
            for (int i = 0; i < demandCount; i++) {
                for (int s = 0; s < stockCount; s++) {
                    x[i][s] = solver.makeBoolVar("x_" + i + "_" + s);
                }
            }

            // 初始化决策变量 y[i]：每个需求 i 是否被满足
            for (int i = 0; i < demandCount; i++) {
                y[i] = solver.makeBoolVar("y_" + i);
            }

            // 初始化决策变量 z[c]：每个大箱 c 是否使用
            for (int c = 0; c < mergeStockCount; c++) {
                z[c] = solver.makeBoolVar("z_" + c);
            }
        }

        public List<Demand> getDemands() {
            return demands;
        }

        public List<Stock> getStocks() {
            return stocks;
        }

        public List<MergeStock> getMergeStocks() {
            return mergeStocks;
        }

        // 需求类
        public static class Demand {
            long id; // 需求编号
            List<String> PNs; // 料号及替代组
            int week; // 周次
            long quantity; // 需求量
            private MPVariable y; // 需求是否满足

            // 从CKDSiteDemand初始化需求
            public Demand(CKDSiteDemand demand, MPSolver solver) {
                this.id = demand.getDemandId();
                this.PNs = new ArrayList<>(demand.getAltPn());
                this.PNs.add(demand.getPn()); // 添加pn到PNs
                this.week = demand.getWeek();
                this.quantity = demand.getQuantity();
                this.y = solver.makeBoolVar("y" + id);
            }

            public long getId() {
                return id;
            }

            public List<String> getPNs() {
                return PNs;
            }

            public int getWeek() {
                return week;
            }

            public long getQuantity() {
                return quantity;
            }
        }

        // 小箱库存类
        public static class Stock {
            String cid; // 小箱编号
            String pn; // 料号
            long quantity; // 物料数量
            int day; // 入库日期
            String mergeCartonId; // 合箱号（大箱编号），null表示未合箱

            // 从CKDSiteStock初始化库存
            public Stock(CKDSiteStock stock) {
                this.cid = stock.getCid();
                this.pn = stock.getPn();
                this.quantity = stock.getQuantity();
                this.day = stock.getDay();
                this.mergeCartonId = stock.getMergeCartonId();
            }

            public String getCid() {
                return cid;
            }

            public String getPn() {
                return pn;
            }

            public long getQuantity() {
                return quantity;
            }

            public int getDay() {
                return day;
            }

            public String getMergeCartonId() {
                return mergeCartonId;
            }
        }

        // 大箱库存类
        public static class MergeStock {
            String mergeCartonId; // 大箱编号
            List<Stock> smallBoxes; // 包含的小箱
            private MPVariable z; // 大箱是否使用

            // 初始化大箱
            public MergeStock(String mergeCartonId, List<Stock> smallBoxes, MPSolver solver) {
                this.mergeCartonId = mergeCartonId;
                this.smallBoxes = smallBoxes;
                this.z = solver.makeBoolVar("z" + mergeCartonId);
            }

            // 添加小箱
            public void addSmallBox(Stock smallBox) {
                this.smallBoxes.add(smallBox);
            }

            public String getMergeCartonId() {
                return mergeCartonId;
            }

            public List<Stock> getSmallBoxes() {
                return smallBoxes;
            }
        }
    }
}














package cn.hgplan.domain.ckd;

import com.google.ortools.linearsolver.MPSolver;
import com.google.ortools.linearsolver.MPObjective;
import com.google.ortools.linearsolver.MPConstraint;
import com.google.ortools.Loader;

import java.util.*;

public class SiteSolver {

    private static final Logger logger = LoggerFactory.getLogger(SiteSolver.class);

    private final String siteId; // 出货单元编号
    private final SiteManager siteManager; // 出货单元管理器
    private MPSolver solver; // OR-Tools 求解器

    private final double alpha1; // 需求满足率的权重
    private final double alpha2; // 混箱 PN 超出量的权重
    private final double alpha3; // 库存日期的权重

    public SiteSolver(CKDSiteRequest siteRequest, String solverType, double alpha1, double alpha2, double alpha3) {
        this.siteId = siteRequest.getSite();
        this.solver = MPSolver.createSolver(solverType);
        this.solver.setTimeLimit(10000); // 设置求解时间限制
        this.siteManager = new SiteManager(siteRequest, solver); // 初始化出货单元
        this.alpha1 = alpha1;
        this.alpha2 = alpha2;
        this.alpha3 = alpha3;
    }

    /**
     * 多目标加权求解
     */
    public OutputSiteSolution solve() {
        // 定义目标函数
        MPObjective objective = solver.objective();
        objective.clear();

        // 目标1: 最大化需求满足数量
        for (SiteManager.Demand demand : siteManager.getDemands()) {
            objective.setCoefficient(demand.y, alpha1);
        }

        // 目标2: 最小化混箱 PN 超出量
        for (SiteManager.MergeStock mergeStock : siteManager.getMergeStocks()) {
            objective.setCoefficient(mergeStock.z, -alpha2);
        }

        // 目标3: 最小化出库物料的库存日期
        for (SiteManager.Stock stock : siteManager.getStocks()) {
            for (SiteManager.Demand demand : siteManager.getDemands()) {
                if (demand.getPNs().contains(stock.getPn())) {
                    objective.setCoefficient(stock.x.get(demand), -alpha3 * stock.day);
                }
            }
        }

        // 设置目标为最大化
        objective.setMaximization();

        // 添加约束条件
        addConstraints();

        // 求解
        long start = System.currentTimeMillis();
        MPSolver.ResultStatus resultStatus = solver.solve();
        long end = System.currentTimeMillis();
        logger.info("求解耗时: {}ms", end - start);

        if (resultStatus != MPSolver.ResultStatus.OPTIMAL && resultStatus != MPSolver.ResultStatus.FEASIBLE) {
            throw new RuntimeException("No feasible solution found");
        }

        logger.info("目标函数值: {}", objective.value());

        // 生成结果
        return generateResults();
    }

    /**
     * 添加约束条件
     */
    private void addConstraints() {
        // 1. 需求满足约束
        for (SiteManager.Demand demand : siteManager.getDemands()) {
            MPConstraint constraint = solver.makeConstraint(demand.quantity, Double.POSITIVE_INFINITY, "demand_" + demand.id);
            for (SiteManager.Stock stock : siteManager.getStocks()) {
                if (demand.getPNs().contains(stock.getPn())) {
                    constraint.setCoefficient(stock.x.get(demand), stock.quantity);
                }
            }
            constraint.setCoefficient(demand.y, -demand.quantity);
        }

        // 2. 小箱库存使用约束
        for (SiteManager.Stock stock : siteManager.getStocks()) {
            MPConstraint constraint = solver.makeConstraint(0, 1, "stock_" + stock.cid);
            for (SiteManager.Demand demand : siteManager.getDemands()) {
                if (demand.getPNs().contains(stock.getPn())) {
                    constraint.setCoefficient(stock.x.get(demand), 1);
                }
            }
        }

        // 3. 大箱出库约束
        for (SiteManager.MergeStock mergeStock : siteManager.getMergeStocks()) {
            for (SiteManager.Stock stock : mergeStock.smallBoxes) {
                MPConstraint constraint = solver.makeConstraint(0, 0, "box_" + mergeStock.mergeCartonId + "_stock_" + stock.cid);
                for (SiteManager.Demand demand : siteManager.getDemands()) {
                    if (demand.getPNs().contains(stock.getPn())) {
                        constraint.setCoefficient(stock.x.get(demand), 1);
                    }
                }
                constraint.setCoefficient(mergeStock.z, -1);
            }
        }
    }

    /**
     * 生成库存分配结果
     */
    private OutputSiteSolution generateResults() {
        OutputSiteSolution output = new OutputSiteSolution();
        output.setSiteId(siteId);

        // 输出需求满足情况
        List<DemandAllocation> demandAllocations = new ArrayList<>();
        for (SiteManager.Demand demand : siteManager.getDemands()) {
            DemandAllocation allocation = new DemandAllocation();
            allocation.setDemandId(demand.id);
            allocation.setSatisfied(demand.y.solutionValue() > 0);

            // 输出该需求分配的小箱 ID
            List<String> allocatedBoxes = new ArrayList<>();
            for (SiteManager.Stock stock : siteManager.getStocks()) {
                if (demand.getPNs().contains(stock.getPn())) {
                    MPVariable x = stock.x.get(demand);
                    if (x != null && x.solutionValue() > 0) {
                        allocatedBoxes.add(stock.cid);
                    }
                }
            }
            allocation.setAllocatedBoxes(allocatedBoxes);
            demandAllocations.add(allocation);
        }
        output.setDemandAllocations(demandAllocations);

        // 输出大箱使用情况
        List<MergeStockUsage> mergeStockUsages = new ArrayList<>();
        for (SiteManager.MergeStock mergeStock : siteManager.getMergeStocks()) {
            MergeStockUsage usage = new MergeStockUsage();
            usage.setMergeCartonId(mergeStock.mergeCartonId);
            usage.setUsed(mergeStock.z.solutionValue() > 0);
            mergeStockUsages.add(usage);
        }
        output.setMergeStockUsages(mergeStockUsages);

        logger.info("出货单元 {} 计算完成", siteId);
        return output;
    }

    // 输出结果类
    public static class OutputSiteSolution {
        private String siteId;
        private List<DemandAllocation> demandAllocations;
        private List<MergeStockUsage> mergeStockUsages;

        // Getters and Setters
    }

    // 需求分配结果类
    public static class DemandAllocation {
        private long demandId;
        private boolean satisfied;
        private List<String> allocatedBoxes;

        // Getters and Setters
    }

    // 大箱使用结果类
    public static class MergeStockUsage {
        private String mergeCartonId;
        private boolean used;

        // Getters and Setters
    }
}
```