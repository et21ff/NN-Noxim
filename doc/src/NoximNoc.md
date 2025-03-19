# NoximNoC 模块结构分析

## 主要组成部分

1. **基础I/O端口**
clock：整个 NoC 的输入时钟信号
reset：系统复位信号
2. **网络通信信号**
   - 模块定义了六个方向（东、西、南、北、上、下）的通信信号，实现了一个完整的三维网状拓扑
   - 每个方向包含多种信号类型：
     - `req_to_X`：请求信号，指示通信请求
     - `ack_to_X`：确认信号，响应通信请求
     - `flit_to_X`：数据信号，传输实际数据单元(flit)
     - `free_slots_to_X`：缓冲区可用空间信息
3. **路由与监控信号**
   - `NoP_data_to_X`：网络数据包信息，用于邻近节点路由决策
   - `RCA_to_X`：区域拥塞感知(Regional Congestion Awareness)信号
   - `on_off_to_X`：节流控制信号，用于动态网络流量控制
   - `DW_tag_to_X`：向下路由标签，支持特定路由算法

4. **物理结构**
NoximTile *t[MAX_STATIC_DIM][MAX_STATIC_DIM][MAX_STATIC_DIM]：三维网格中的处理节点数组

5. **全局表格与模型**
grtable：全局路由表
gttable：全局流量表
nnmodel：神经网络模型
6. **统计计数器**
   - 多种计数器数组，用于追踪不同类型的数据包和流量信息

## 主要方法

1. **flitsMonitor()**
    - 用于监控和统计整个网络中数据包(flits)的数量
    - 在每个时钟周期检查所有网格节点，累计计数所有路由器中的数据包
    - 通过遍历三维网格结构(x,y,z)获取综合统计信息
    - 输出当前网络中数据包总数

2. **构造函数与初始化**
    - `SC_CTOR(NoximNoC)`: 模块构造函数，完成网络初始化
    - `buildMesh()`: 构建三维网格结构
    - 初始化热点接口(Thermal Interface)，用于温度监控
    - 分配功率和温度跟踪数组，用于热模型模拟
    - 注册`entry`方法到SystemC内核，对时钟和复位信号敏感

3. **析构函数**
    - 释放热点接口资源，确保内存正确释放

4. **支持方法**
    - `searchNode(const int id)`: 根据节点ID查找对应的网格节点
    - `EmergencyDecision()`: 紧急情况决策机制，用于处理网络异常状态
    ## 私有成员与方法

    1. **核心基础方法**
        - `buildMesh()`: 构建完整的三维网格拓扑结构，创建并连接所有网络节点
        - `entry()`: SystemC线程入口点，处理时钟事件和系统行为

    2. **热管理接口**
        - `Thermal_IF* HS_interface`: 热点(HotSpot)接口指针，用于与热模拟工具交互
        - `bool LastIsEmergency`: 标记上一状态是否为紧急温度状态，用于热管理决策

    3. **功耗跟踪系统**
        - `vector<double> instPowerTrace`: 存储瞬时功耗数据的向量
        - `vector<double> overallPowerTrace`: 存储总体功耗数据的向量
        - `transPwr2PtraceFile()`: 将瞬态功耗数据写入跟踪文件，用于功耗分析
        - `steadyPwr2PtraceFile()`: 将稳态功耗数据写入跟踪文件，用于长期功耗评估

    4. **温度监控系统**
        - `vector<double> TemperatureTrace`: 存储系统各点温度数据的向量
        - `setTemperature()`: 更新并设置系统温度模型，用于热管理和散热分析

## NoximNoC::buildMesh 函数详解
buildMesh()函数是 NoximNoC 类的核心方法，负责构建并初始化整个三维网络芯片系统。这个函数展示了构建复杂片上网络的完整过程，包括网格创建、信号连接和初始状态设置。

### 初始化与资源加载

函数开始先检查必要资源的可用性：
- 如果使用基于表的路由算法，加载路由表
- 如果使用基于表的流量分布，加载流量表
- 加载神经网络模型（标记为 "tytyty"，可能是实验性功能）

### 三维网格构建

通过三重嵌套循环创建 x、y、z 三个维度的网络节点。每个节点是一个 NoximTile 对象，包含自己的路由器和处理元件。对于每个节点：
- 创建并命名节点（如 "Tile[00][01][02]"）
- 配置路由器参数（ID、预热时间、缓冲区深度和路由表）
- 设置处理元件的本地 ID、流量表和神经网络模型
- 连接到全局时钟和重置信号

### 信号连接系统

函数详细定义了每个节点与相邻节点之间的互连，包括：
- 请求/应答/数据信号（req_rx/tx, ack_rx/tx, flit_rx/tx）连接到六个方向（东、西、南、北、上、下）
- 缓冲区状态信号（free_slots 和 free_slots_neighbor）
- 邻居路径数据（NoP_data）用于路由决策
- 区域拥塞感知（RCA）监控信号，用于智能流量管理
- 节流控制信号（on_off），用于紧急情况下的流量调节
- 向下路由标签（DW_tag），支持特定路由算法

### 边界条件处理

函数处理网格边界节点的特殊情况：
- 创建一个空的 NoP 数据结构作为边界值
- 初始化边界节点的各类信号（清零或设为无效）
- 将边界方向的路由表条目标记为无效，防止路由到网格外部

### 初始状态设置

最后，函数设置系统的初始状态：
- 初始化各种计数器（本地、邻居和接收的数据包统计）
- 设置紧急模式和节流标志（默认为关闭状态）
- 为特定位置的节点（如 (3,3)）设置特殊的节流参数，可能用于测试热点管理

函数在测试模式下，会将特定节点配置为紧急状态以测试系统的热点处理能力。最后，通过全局参数设置节点的节流配置，为特定位置的节点赋予不同的处理策略。

## NoximNoC::entry() 方法 - 热管理与节流控制系统

`NoximNoC::entry()` 是整个网络芯片的心脏功能，负责热量监控、功耗跟踪和动态流量控制。这个周期性执行的方法实现了网络芯片的热管理系统，尤其关键于3D芯片架构中，其中热量问题尤为突出。

### 1. 功耗重置与初始化

系统复位时会重置所有路由器的功耗状态：

```cpp
if (reset.read()) {
   //in reset phase, reset power value 
   for(int k=0; k < NoximGlobalParams::mesh_dim_z; k++) {
        for(int j=0; j < NoximGlobalParams::mesh_dim_y; j++) {    
            for(int i=0; i < NoximGlobalParams::mesh_dim_x; i++) {        
                t[i][j][k]->r->stats.power.resetPwr();
                t[i][j][k]->r->stats.power.resetTransientPwr();
            }
        }
    }
}
```

### 2. 周期性热监控

函数以固定间隔（

TEMP_REPORT_PERIOD

）执行热管理操作：

```cpp
if(( getCurrentCycleNum() % (int) (TEMP_REPORT_PERIOD) ) == 0 ) {
    // 热管理代码...
}
```

### 3. 功耗追踪与温度计算

```cpp
//accumulate steady power after warm-up time
if( getCurrentCycleNum() > (int)(NoximGlobalParams::stats_warm_up_time * CYCLE_PERIOD) )
   steadyPwr2PtraceFile();

//transient power
transPwr2PtraceFile();
HS_interface->Temperature_calc(instPowerTrace, TemperatureTrace);
setTemperature();
```

这段代码先记录功耗数据，然后使用热点接口计算实际温度并更新到系统中。

### 4. 紧急热控制决策

```cpp
bool IsEmergency = EmergencyDecision();  //检查当前温度是否超过热限制，决定是否进入紧急模式
```

### 5. 流量统计与节流配额计算

```cpp
// 收集统计数据
for(int z=0; z < NoximGlobalParams::mesh_dim_z; z++) {
    for(int y=0; y < NoximGlobalParams::mesh_dim_y; y++) 
        for(int x=0; x < NoximGlobalParams::mesh_dim_x; x++) {
            tot_cnt_local[x][y][z]+=t[x][y][z]->pe->cnt_local;
            // 其他统计数据...
            
            // 如果不在紧急状态，计算新的流量配额
            if(!LastIsEmergency) {
                t[x][y][z]->pe->Quota_local = (int)((double)t[x][y][z]->pe->cnt_local * 
                                              (float)(1-NoximGlobalParams::throt_ratio));
                t[x][y][z]->r->Quota_neighbor = (int)((double)t[x][y][z]->r->cnt_neighbor * 
                                               (float)(1-NoximGlobalParams::throt_ratio));
            }
        }
}
```

这段代码展示了动态流量节流控制的核心：系统跟踪每个节点的流量情况，并根据节流比率（throt_ratio）计算下一周期的流量配额。只有在非紧急状态下才更新配额，这确保了热点区域不会因频繁调整而产生不稳定。

### 6. 计数器重置与状态更新

```cpp
LastIsEmergency = IsEmergency;  // 记录当前紧急状态供下次使用

// 重置所有流量计数器
for(int z=0; z < NoximGlobalParams::mesh_dim_z; z++) 
    for(int y=0; y < NoximGlobalParams::mesh_dim_y; y++) 
        for(int x=0; x < NoximGlobalParams::mesh_dim_x; x++) {                             
            t[x][y][z]->pe->cnt_local = 0;  // 重置计数器
            t[x][y][z]->r->cnt_neighbor = 0;
            t[x][y][z]->r->cnt_received = 0;
        }
```

这个方法体现了现代芯片设计中动态温度管理（DTM）的完整实现，通过实时监控与控制来平衡性能和热安全。

## NoximNoC功耗跟踪与监控函数
transPwr2PtraceFile()函数是Noxim网络芯片模拟器中的功耗数据收集模块，用于追踪和记录整个三维网络中每个节点的瞬时功耗状态。这个函数是热管理系统的核心组件之一，提供了温度模拟所需的功耗输入数据。

### 功能与数据收集过程

该函数通过三重嵌套循环遍历整个三维网络结构，针对每个节点收集三种不同类型的功耗数据：

```cpp
for(o=0; o < NoximGlobalParams::mesh_dim_z; o++){
    for(n=0; n < NoximGlobalParams::mesh_dim_y; n++) {
        for(m=0; m < NoximGlobalParams::mesh_dim_x; m++) {
            // 计算一维数组索引
            idx = m + n*NoximGlobalParams::mesh_dim_x + o*NoximGlobalParams::mesh_dim_y*NoximGlobalParams::mesh_dim_x;
            
            // 收集路由器核心的瞬时功耗
            instPowerTrace[3*idx] = t[m][n][o]->r->stats.power.getTransientRouterPower();
            results_log_pwr << instPowerTrace[3*idx]<<"\t";
```

### 收集的功耗类型

函数针对每个节点收集三种关键组件的功耗数据：

1. **路由器核心功耗**：网络通信操作消耗的能量
   ```cpp
   instPowerTrace[3*idx] = t[m][n][o]->r->stats.power.getTransientRouterPower();
   ```

2. **存储器功耗**：缓冲区和存储操作消耗的能量
   ```cpp
   instPowerTrace[3*idx+1] = t[m][n][o]->r->stats.power.getTransientMEMPower();
   ```

3. **浮点MAC单元功耗**：计算操作消耗的能量
   ```cpp
   instPowerTrace[3*idx+2] = t[m][n][o]->r->stats.power.getTransientFPMACPower();
   ```

### 数据组织与存储

函数使用了巧妙的索引方案组织功耗数据：

```cpp
// 路由器功耗存储于 3*idx 位置
instPowerTrace[3*idx] = t[m][n][o]->r->stats.power.getTransientRouterPower();

// 存储器功耗存储于 3*idx+1 位置
instPowerTrace[3*idx+1] = t[m][n][o]->r->stats.power.getTransientMEMPower();

// 浮点MAC功耗存储于 3*idx+2 位置
instPowerTrace[3*idx+2] = t[m][n][o]->r->stats.power.getTransientFPMACPower();
```

### 瞬时功耗重置

每次收集完功耗数据后，函数会重置瞬时功耗计数器，为下一周期的统计做准备：

```cpp
t[m][n][o]->r->stats.power.resetTransientPwr(); //清除这段时间内Power累计
```

### 数据输出与记录

所有功耗数据会实时写入到功耗日志文件中：

```cpp
results_log_pwr << instPowerTrace[3*idx]<<"\t";
// ...其他功耗数据...
results_log_pwr<<"\n";
```

这个功耗跟踪系统提供了精细的数据粒度，能够区分不同硬件组件的功耗贡献，为网络芯片的热管理和功耗优化提供关键参考数据。这种分类跟踪方式特别适合进行热点分析，确定哪些组件或区域是主要的功耗和热量来源。

## 稳态功耗收集与长期功耗分析

`steadyPwr2PtraceFile()` 函数实现了网络芯片中稳态功耗（steady state power）的收集和累积，这与瞬态功耗跟踪互为补充，共同构成完整的功耗监控系统。

### 稳态功耗收集的工作原理

函数通过遍历整个三维网格，收集每个节点的稳态功耗数据：

```cpp
for(o=0; o < NoximGlobalParams::mesh_dim_z; o++){
    for(n=0; n < NoximGlobalParams::mesh_dim_y; n++) {
        for(m=0; m < NoximGlobalParams::mesh_dim_x; m++) {
            idx = m + n*NoximGlobalParams::mesh_dim_x + o*NoximGlobalParams::mesh_dim_y*NoximGlobalParams::mesh_dim_x;
            
            // 累加路由器核心的稳态功耗
            overallPowerTrace[3*idx] += t[m][n][o]->r->stats.power.getSteadyStateRouterPower();
```

### 与瞬态功耗收集的关键区别

1. **数据累加而非覆盖**：稳态功耗被累加到 `overallPowerTrace` 数组中
   ```cpp
   overallPowerTrace[3*idx] += t[m][n][o]->r->stats.power.getSteadyStateRouterPower();
   ```
   而瞬态功耗会覆盖 `instPowerTrace` 数组中的值

2. **无重置操作**：不同于瞬态功耗收集后会重置计数器，稳态功耗继续累积

3. **不输出到日志**：稳态功耗数据通常不会每次都写入日志文件，而是用于最终分析

### 功耗分类

与瞬态功耗相同，稳态功耗也分为三种类型：

1. **路由器核心稳态功耗**
   ```cpp
   overallPowerTrace[3*idx] += t[m][n][o]->r->stats.power.getSteadyStateRouterPower();
   ```

2. **存储器稳态功耗**
   ```cpp
   overallPowerTrace[3*idx+1] += t[m][n][o]->r->stats.power.getSteadyStateMEMPower();
   ```

3. **浮点MAC单元稳态功耗**
   ```cpp
   overallPowerTrace[3*idx+2] += t[m][n][o]->r->stats.power.getSteadyStateFPMACPower();
   ```

### 应用价值

稳态功耗收集主要用于以下场景：
- 长期系统行为分析
- 芯片整体功耗预算评估
- 热点趋势预测
- 功耗均衡设计验证

通过同时追踪瞬态和稳态功耗，Noxim系统能够提供全面的功耗分析，既能捕捉短期波动，又能展现长期趋势，为芯片设计师提供宝贵的优化依据。

## NoximNoC热紧急状态决策机制
EmergencyDecision()函数是三维网络芯片温度管理系统的核心决策组件，负责根据各节点温度情况决定紧急降温策略。该函数设计了多种精细的节流控制方案，以应对不同程度的热危机。

函数根据 `NoximGlobalParams::throt_type` 参数实现五种不同的节流策略：

### 1. 正常模式 (THROT_NORMAL)

将所有节点设置为非紧急状态：

```cpp
if(NoximGlobalParams::throt_type == THROT_NORMAL) {
    for(int z=0; z < NoximGlobalParams::mesh_dim_z; z++) 
        for(int y=0; y < NoximGlobalParams::mesh_dim_y; y++) 
            for(int x=0; x < NoximGlobalParams::mesh_dim_x; x++) {
                t[x][y][z]->pe->emergency = false;
                t[x][y][z]->r->emergency = false;
            }
}
```

### 2. 测试模式 (THROT_TEST)

在预定位置创建热点以测试系统响应：

```cpp
if(z < (NoximGlobalParams::mesh_dim_z - 2)){
    if(((x == 3))&&((y == 3))){
        t[x][y][z]->pe->emergency = true;
        t[x][y][z]->r->emergency = true;
    }
}
```

### 3. 全局节流 (THROT_GLOBAL)

如有任何节点超过温度阈值，则全网进入紧急状态：

```cpp
if(t[x][y][z]->r->stats.temperature > TEMP_THRESHOULD)
    emergency = true;

if(emergency) {
    isEmergency = true;
    for(int z=0; z < NoximGlobalParams::mesh_dim_z; z++)
        for(int y=0; y < NoximGlobalParams::mesh_dim_y; y++)
            for(int x=0; x < NoximGlobalParams::mesh_dim_x; x++) {
                t[x][y][z]->pe->emergency = true;
                t[x][y][z]->r->emergency = true;
            }
}
```

### 4. 分布式节流 (THROT_DISTRIBUTED)

仅对过热节点实施节流控制，并可选择沿垂直方向扩展：

```cpp
for(int z=NoximGlobalParams::mesh_dim_z-1; z >= 0 ; z--) // 从底层(靠近散热器)向上检查
    // 如果温度过高
    if(t[x][y][z]->r->stats.temperature > TEMP_THRESHOULD) {	
        isEmergency = true;
        #ifdef __Throttle_vertical__
            // 垂直节流 - 控制整列
        #else
            // 只控制热点
            t[x][y][z]->pe->emergency = true;
            t[x][y][z]->r->emergency = true;
        #endif
    }
```

### 5. 垂直梯度节流 (THROT_VERTICAL)

采用垂直方向梯度控制策略，对热点区域采用不同级别的流量限制：

```cpp
if(t[x][y][z]->r->stats.temperature > TEMP_THRESHOULD) {
    // 增加紧急级别
    t[x][y][z]->pe->emergency_level++;
    
    // 
    for(int zz = z; zz<t[x][y][z]->pe->emergency_level+z; zz++) {
        t[x][y][zz]->pe->emergency = true;
        
        // 最上层节点限流50%
        if(zz == t[x][y][z]->pe->emergency_level+z-1) {								
            t[x][y][zz]->pe->Q_ratio = 0.5;
        } else {
            // 其他层完全阻断流量
            t[x][y][zz]->pe->Q_ratio = 0;
        }
    }
}
```

## 热管理设计理念

这个函数体现了几个重要的热管理原则：

1. **从下向上检测**：算法从靠近散热器的底层向上检测，反映了三维芯片中热量向上积累的物理特性

2. **分层控制**：垂直节流策略实现了精确的分层流量控制，更靠近热源的层节流更严格

3. **紧急级别动态调整**：系统会根据持续的热对热点及以上层节点应用不同程度的流量控制状况动态增加紧急级别，扩大控制范围

4. **恢复机制**：当温度恢复正常时，系统会重置紧急状态并恢复正常流量配额

这种多层次的热管理策略使得系统能够根据不同的热状况采取适当的措施，在保持系统性能的同时有效防止热失控。
