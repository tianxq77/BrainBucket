# 1.需求 for MRP采购分料逻辑
2024-11-22 MRP新逻辑变更: （by 志锦）
1. 同一时间优先级内run两次（与原逻辑保持一致）;
2. run_index = 1时, 可用供给过滤，跨工厂增加按lead_time的过滤, lead_time从供给上取;
3. 供给排序，在优先级排序之后按可用日期升序排列;
4. run_index = 1时, 对于未齐套的需求, 需要按需求序列顺序滚动计算;
5. 对于存在半成品替代组的情况, 报错处理;
6. 输入数据增加time_date_mapping_list;


1. 计算流程的变更: MRP采购及MRP调拨两个流程合成一个流程, 输出写入两个文件夹; (2天)
2. MRP采购: 按time_priority计算, run两次, 考虑lead_time的过滤, 增加滚动计算; (7天)
3. 供给可用性过滤: lead_time的过滤, 前两周的过滤, (不要区分before & after); (3天)
4. 计算需求根据MRP采购及MRP调拨的场景计算分组; (1天)


1. JSONIO
2. CalcCore, RamaxelAlgoFlow (yuan)
3. policy
4. 供给
  4.1 初始化
  4.2 LT条件过滤 (yuan)
  4.3 排序（替代组中排序也需要相应更改）
5. 齐套性检查及滚动计算（solution & policy）
6. UpdateOwnDetailPool for MRP (yuan)



## 1.1 需求 for me
### 1.1.1 齐套逻辑判断
遍历所有物料分配详情，判断每个物料的分配总量是否满足其需求量，一旦发现某物料不满足齐套条件，无需继续检查剩余物料

* 如果某物料的净需求量非0

  * 如果无下层则认为不齐套，直接将 `m_KittingFlag` 设置为 `false` 并返回

  * 如果有下层，进入下层物料判断
* 如果所有物料都不缺料，最下层净需求为0，则将 `m_KittingFlag` 设置为 `true`

### 1.1.2 物料分配详情计算
* 过滤供给

  * 与原mrp逻辑差别在于不用根据run index区分时间范围，删去可用日期判断

* 区分有无bom

  * 有bom

    * 根据bom依次对成品自身、原材料、半成品、替代组过滤。（需求上的约束、bom、工厂库位）

    * 在以上过滤条件后新增根据end_date和LT过滤：FactorySupplyCalcAttr::_FilterForLeadTime(const int &end_date)

  * 无bom

    * 只算顶层库存,只run一次（run_index=0）

* 扣料：逐步使用库存扣减

### 1.1.3 物料分配详情更新

与原逻辑基本一致，只是在更新前多一个齐套判断，更新kitting flag

1. **获取需求分配详情对象**

2. **更新分配详情中的 OwnCalcDetailPool**

   * 如果分配详情对象已存在

   * 如果分配详情对象不存在，新建一个 `DemandAssignDetail` 对象并插入AssignSeqDetail

**原理论逻辑**：

1. **根据需求信息获取需求分配详情对象**：_GetSpDemandAssignDetailValue

   根据需求信息的时间优先级、需求天、原始优先级逐层进入AssignSeqDetail，在m_SpDemandAssignDetailPool 中查找与 sp_demand_info 的 DemandId 匹配的对应分配详情

   1. _GetSpDemandAssignDetailValue属于 AssignSeqDetail 类，从 m_TimeAssignSeqDetailPool 中根据 time_priority 查找并返回对应的 DemandAssignDetail 对象。

      具体步骤如下：

      1. 从传入的 sp_demand_info_calc_attr 中获取 sp_demand_info。

      2. 从 sp_demand_info 中获取 time_priority（比如以一周为一个优先级，则第一周第二周分别为0，1）。

      3. 在 m_TimeAssignSeqDetailPool 中查找 time_priority 对应的记录。

      4. 如果找到对应记录，则调用其 GetSpDemandAssignDetailValue 方法返回结果。

      5. 如果未找到对应记录，则返回 std::nullopt。

   2. GetSpDemandAssignDetailValue为 TimeAssignSeqDetail 的类的方法，该方法接收一个 DemandInfo 智能指针作为参数，并尝试从 m_DayAssignSeqDetailPool 映射中查找与请求日期对应的 DemandAssignDetail 对象。具体步骤如下：

      1. 获取 sp_demand_info 中的请求日期 request_date。

      2. 在 m_DayAssignSeqDetailPool 中查找 request_date。

      3. 如果找到对应的条目，则调用该条目的 GetSpDemandAssignDetailValue 方法并返回结果。

      4. 如果未找到对应的条目，则返回 std::nullopt。

   3. GetSpDemandAssignDetailValue 为 DayAssignSeqDetail 的类的方法，从优先级分配序列详情池中获取与给定需求信息对应的详细分配信息。

      具体步骤如下：

      1. 获取需求信息的原始优先级。

      2. 在优先级分配序列详情池中查找该优先级。

      3. 如果找到，则调用对应优先级的 GetSpDemandAssignDetailValue 方法并返回结果。

      4. 如果未找到，则返回 std::nullopt 表示没有找到对应的详细分配信息。

   4. GetSpDemandAssignDetailValue为 PriorityAssignSeqDetail 的类的方法，从 m_SpDemandAssignDetailPool 容器中查找与传入的 sp_demand_info 对象具有相同 DemandId 的 DemandAssignDetail 对象。

      具体步骤如下：

      1. 使用 std::find_if 算法在 m_SpDemandAssignDetailPool 中查找与 sp_demand_info 的 DemandId 匹配的元素。

      2. 如果找到匹配的元素，则返回该元素的共享指针。

      3. 如果没有找到匹配的元素，则返回 std::nullopt 表示未找到。

2. **更新分配详情中的 OwnCalcDetailPool**：

   * 如果分配详情对象已存在，InsertOwnCalcDetailPool分别更新替代组池和非替代组池

     *  DemandAssignDetail ：：InsertOwnCalcDetailPool，其功能是将传入的 own_calc_detail_pool 对象中的数据插入到当前对象的内部数据结构中。具体步骤如下：

       1. 获取 own_calc_detail_pool 中的非替代组池，并调用 _InsertOwnCalcDetailPool 方法将其插入。

          将 own_calc_detail_pool 中的每个 OwnCalcDetail 对象插到 m_OwnDemandMaterialDetailPool 中。具体步骤如下：

          1. 遍历 own_calc_detail_pool 中的每个 OwnCalcDetail 对象。

          2. 获取当前对象的 material_id。

          3. 在 m_OwnDemandMaterialDetailPool 中查找是否存在相同 material_id 的 OwnDemandMaterialDetail 对象。

          4. 如果存在，则调用 Merge 方法合并数据。

             1.  OwnDemandMaterialDetail 类的一个成员函数 Merge，用于合并 OwnCalcDetail 对象中的材料信息。具体功能如下：

                1. 获取 own_calc_detail 中的 MaterialNetQuantity 并赋值给当前对象的 m_MaterialNetQuantity。

                2. 获取 own_calc_detail 中的 AssignMaterialQuantity 并累加到当前对象的 m_AssignMaterialQuantity。

                3. 遍历 own_calc_detail 中的 MaterialFactoryCalcDetailPool，将每个 MaterialFactoryCalcDetail 转换为 MaterialFactoryAssignDetail 并添加到当前对象的 m_MaterialFactoryAssignDetailPool。

          5. 如果不存在，则创建一个新的 OwnDemandMaterialDetail 对象并添加到 m_OwnDemandMaterialDetailPool 中。

       2. 获取 own_calc_detail_pool 中的替代组池，并调用 _InsertAlternativeOwnCalcDetailPools 方法将其插入。是 DemandAssignDetail 类的一个成员函数，其主要功能是从 alternative_own_calc_detail_pools 中提取数据，并将其插入到 m_OwnDemandMaterialDetailPool 中。具体步骤如下：

          1. 遍历 alternative_own_calc_detail_pools 中的每一个 AlternativeOwnCalcDetailPool。

          2. 从每个 AlternativeOwnCalcDetailPool 中获取父材料ID、替代码、毛需求量、净需求量和子计算详情池。

          3. 创建一个 OwnAlternativeInfo 对象，包含上述信息。

          4. 遍历子计算详情池中的每一个 OwnCalcDetail。

          5. 在 m_OwnDemandMaterialDetailPool 中查找与当前 OwnCalcDetail 的材料ID匹配的记录。

          6. 如果找到匹配的记录，则调用 MergeForAlternative 方法合并数据。

             OwnDemandMaterialDetail类的一个成员函数MergeForAlternative。该函数的主要功能是将own_calc_detail对象中的材料净量、分配量、毛量以及工厂计算细节合并到当前对象中，并设置当前对象的替代信息指针。

             具体步骤如下：

             1. 获取并设置材料净量。

             2. 累加分配材料量。

             3. 累加材料毛量。

             4. 遍历工厂计算细节池，将每个细节复制到当前对象的工厂分配细节池中。

             5. 设置当前对象的替代信息指针。

          7. 如果没有找到匹配的记录，则创建一个新的 OwnDemandMaterialDetail 对象并添加到 m_OwnDemandMaterialDetailPool 中。

   * 如果分配详情对象不存在，新建一个 `DemandAssignDetail` 对象

3. **插入新对象


## 1.2 需求 for all
### 1.2.1 需求滚动


### 1.2.2 供给过滤


### 1.2.3 逻辑串联

# 2.代码逻辑
## 2.1 线程池
