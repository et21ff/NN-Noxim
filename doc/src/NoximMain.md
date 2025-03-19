## Noxim 核心数据结构分析

这段代码定义了 Noxim 模拟器的几个核心数据结构：

### 1. NoximCoord 类
```cpp
class NoximCoord {
  public:
    int x;            // X 坐标
    int y;            // Y 坐标
    int z;            // Z 坐标（该实现扩展了原来的 2D 版本）

    // 比较两个坐标是否相等，现在包括 z 坐标
    inline bool operator ==(const NoximCoord & coord) const {
        return (coord.x == x && coord.y == y && coord.z == z);
    }
};
```

这是一个表示网络节点三维坐标的类，原始版本只支持 2D 网络，现在扩展为 3D 网络。

### 2. NoximFlitType 枚举
```cpp
enum NoximFlitType {
    FLIT_TYPE_HEAD,   // 头部分片，包含路由信息
    FLIT_TYPE_BODY,   // 数据分片
    FLIT_TYPE_TAIL    // 尾部分片，标志数据包结束
};
```

定义了数据包分片(Flit)的三种类型，符合标准 NoC 的分片设计。

### 3. NoximPayload 结构
```cpp
struct NoximPayload {
    sc_uint<32> data;  // 32位数据总线
    
    // 比较两个载荷是否相等
    inline bool operator ==(const NoximPayload & payload) const {
        return (payload.data == data);
    }
};
```

定义了分片的有效载荷，使用 SystemC 的 32 位无符号整数类型。

### 4. NoximPacket 结构
```cpp
struct NoximPacket {
    int src_id;         // 源节点ID
    int dst_id;         // 目标节点ID
    double timestamp;   // 数据包生成时间戳
    int size;           // 数据包大小(分片数)
    int flit_left;      // 剩余待处理分片数
    int routing;        // 路由算法标识
    
    // 构造函数和初始化方法
    NoximPacket() { }
    NoximPacket(const int s, const int d, const double ts, const int sz) {
        make(s, d, ts, sz);
    }
    
    void make(const int s, const int d, const double ts, const int sz) {
        src_id = s;
        dst_id = d;
        timestamp = ts;
        size = sz;
        flit_left = sz;
    }
};
```

定义了完整的数据包结构，包含源目节点信息、时间戳和分片数量信息。

## Noxim 网络路由和通道数据结构分析

这段代码定义了网络芯片通信中的几个关键数据结构：

### 1. NoximRouteData 结构
```cpp
struct NoximRouteData {
    int current_id;    // 当前路由器ID
    int src_id;        // 源节点ID
    int dst_id;        // 目标节点ID
    int dir_in;        // 数据包进入方向
    bool XYX_routing;  // XYX路由算法标志
    int routing;       // 使用的路由算法类型
};
```
这个结构包含路由器进行路由决策所需的全部信息。

### 2. NoximChannelStatus 结构
```cpp
struct NoximChannelStatus {
    int free_slots;    // 缓冲区中的可用槽位
    bool available;    // 通道是否可用
    
    // 扩展功能
    bool throttle;     // 节流控制标志
    double temperature; // 温度信息
    int channel_count;  // 通道使用计数
    
    // 相等比较运算符
    inline bool operator ==(const NoximChannelStatus & bs) const {
        return (free_slots == bs.free_slots && available == bs.available);
    };
};
```
这个结构描述了通信通道的状态，包含了流量控制和热管理所需的信息。注意相等比较操作符没有考虑扩展的字段。

### 3. NoximNoP_data 结构
```cpp
struct NoximNoP_data {
    int sender_id;     // 发送方ID
    NoximChannelStatus channel_status_neighbor[DIRECTIONS]; // 不同方向邻居的通道状态
    
    // 相等比较运算符
    inline bool operator ==(const NoximNoP_data & nop_data) const {
        return (sender_id == nop_data.sender_id &&
                nop_data.channel_status_neighbor[0] == channel_status_neighbor[0] &&
                nop_data.channel_status_neighbor[1] == channel_status_neighbor[1] &&
                nop_data.channel_status_neighbor[2] == channel_status_neighbor[2] &&
                nop_data.channel_status_neighbor[3] == channel_status_neighbor[3]);
    };
};
```
这个结构实现了 NoP (Neighbors-on-Path) 信息交换机制，用于获取周围路由器的拥塞状态，支持自适应路由决策。

**注意**：比较操作符中只比较了四个方向(0-3)，虽然前面定义了 
DIRECTIONS为6，这可能是扩展为3D时的遗留问题。

这些数据结构对理解网络芯片中的路由机制、拥塞控制和热管理非常关键。

### NoximFlit 结构
```cpp
struct NoximFlit {
    int src_id;           // 源节点ID
    int dst_id;           // 目标节点ID
    NoximFlitType flit_type; // 分片类型(头/主体/尾)
    int sequence_no;      // 分片在数据包中的序号
    NoximPayload payload; // 有效载荷
    double timestamp;     // 生成时间戳
    int hop_no;          // 当前经过跳数
    int routing_f;       // 路由算法标志
    
    float data;          // 浮点数据 (用于神经网络数据)
    bool XYX_routing;    // XYX路由算法标志
    int src_Neu_id;      // 源神经元ID
    
    // 相等比较运算符
    inline bool operator ==(const NoximFlit & flit) const {
        return (flit.src_id == src_id && flit.dst_id == dst_id
            && flit.flit_type == flit_type
            && flit.sequence_no == sequence_no
            && flit.payload == payload && flit.timestamp == timestamp
            && flit.hop_no == hop_no);
    }
};
```
NoximFlit是网络通信的基本单位，一个数据包被分成多个 flit 进行传输。该结构除了基本的路由信息外，还包含了神经网络相关的扩展字段。

**值得注意的是**：相等比较运算符没有比较 `routing_f`、`data`、`XYX_routing` 和 `src_Neu_id` 这几个扩展字段。

### 时钟周期计算函数
```cpp
inline int getCurrentCycleNum(){
    return (int)(sc_time_stamp().to_double()/1000/CYCLE_PERIOD);
};
```
这个函数用于获取当前的模拟周期数：
1. sc_time_stamp()
 获取当前 SystemC 模拟时间
2. `to_double()` 将时间转换为皮秒(ps)数值
3. 除以 1000 转换为纳秒(ns)
4. 再除以 CYCLE_PERIOD（时钟周期常量，在其他地方定义）得到周期数
5. 整数转换得到当前周期编号

## Noxim 数据结构输出运算符重载分析

这段代码定义了几个输出运算符重载，用于方便地将 Noxim 数据结构打印到输出流：

### 1. NoximFlit 的输出运算符
```cpp
inline ostream & operator <<(ostream & os, const NoximFlit & flit)
{
    // 根据详细级别使用不同格式
    if (NoximGlobalParams::verbose_mode == VERBOSE_HIGH) {
        // 详细模式：打印所有信息
        os << "### FLIT ###" << endl;
        os << "Source Tile[" << flit.src_id << "]" << endl;
        os << "Destination Tile[" << flit.dst_id << "]" << endl;
        // 打印分片类型
        // 打印序列号、时间戳、跳数等
    } else {
        // 简洁模式：只显示关键信息
        os << "[type: ";
        // 简写分片类型(H/B/T)
        os << ", seq: " << flit.sequence_no << ", " << flit.src_id << "-->" << flit.dst_id << "]";
    }
    
    // 神经网络扩展信息
    os << ", src_Neu_id = " << flit.src_Neu_id;
    os << ", data = " << flit.data;
    
    return os;
}
```

### 2. NoximChannelStatus 的输出运算符
```cpp
inline ostream & operator <<(ostream & os, const NoximChannelStatus & status)
{
    // 显示通道状态：A=可用，N=不可用，加上空闲槽位数
    char msg = status.available ? 'A' : 'N';
    os << msg << "(" << status.free_slots << ")";
    return os;
}
```

### 3. NoximNoP_data 的输出运算符
```cpp
inline ostream & operator <<(ostream & os, const NoximNoP_data & NoP_data)
{
    // 显示发送方ID和所有方向的通道状态
    os << "      NoP data from [" << NoP_data.sender_id << "] [ ";
    for (int j = 0; j < DIRECTIONS; j++)
        os << NoP_data.channel_status_neighbor[j] << " ";
    cout << "]" << endl;
    return os;
}
```

### 4. NoximCoord 的输出运算符
```cpp
inline ostream & operator <<(ostream & os, const NoximCoord & coord)
{
    // 显示三维坐标
    os << "(" << coord.x << "," << coord.y << "," << coord.z <<")";
    return os;
}
```

这些重载运算符提供了一致的日志记录接口，方便调试和分析网络芯片的通信过程，特别是在仿真和性能分析时非常有用。

## SystemC 波形追踪函数重载分析

这段代码定义了 SystemC 的波形追踪函数重载，用于在仿真过程中记录和监控 Noxim 各数据结构的变化：

### 1. NoximFlit 的追踪函数
```cpp
inline void sc_trace(sc_trace_file * &tf, const NoximFlit & flit, string & name)
{
    sc_trace(tf, flit.src_id, name + ".src_id");
    sc_trace(tf, flit.dst_id, name + ".dst_id");
    sc_trace(tf, flit.sequence_no, name + ".sequence_no");
    sc_trace(tf, flit.timestamp, name + ".timestamp");
    sc_trace(tf, flit.hop_no, name + ".hop_no");
}
```
这个函数将 flit 中的关键字段添加到波形追踪文件中，便于后续分析。**注意**：并未包含所有字段，尤其是未追踪扩展字段如 `routing_f`、`data` 和 `src_Neu_id`。

### 2. NoximNoP_data 的追踪函数
```cpp
inline void sc_trace(sc_trace_file * &tf, const NoximNoP_data & NoP_data, string & name)
{
    sc_trace(tf, NoP_data.sender_id, name + ".sender_id");
}
```
这个函数仅追踪了 NoP 数据中的发送者 ID，而没有追踪通道状态数组。

### 3. NoximChannelStatus 的追踪函数
```cpp
inline void sc_trace(sc_trace_file * &tf, const NoximChannelStatus & bs, string & name)
{
    sc_trace(tf, bs.free_slots, name + ".free_slots");
    sc_trace(tf, bs.available, name + ".available");
}
```
这个函数追踪了通道状态中的空闲槽位数和可用性标志，但没有追踪温度和节流相关字段。

### 作用与意义

这些追踪函数允许 SystemC 仿真环境记录 NoC 网络通信的关键信号变化：
- 实现波形可视化以便调试和性能分析
- 提供时序数据用于计算网络延迟和吞吐量
- 作为功能验证的基础工具

这里很明显部分扩展功能（如温度相关字段）尚未完全集成到波形追踪系统中，可能需要进一步完善。

## 三维网络芯片坐标与ID转换函数分析

这段代码定义了网络芯片中节点的一维ID与三维坐标间的转换函数：

### 1. id2Coord 函数：ID转坐标
```cpp
inline NoximCoord id2Coord(int id)
{
    NoximCoord coord;
    
    // 计算z坐标（层数）
    coord.z = id / (NoximGlobalParams::mesh_dim_x*NoximGlobalParams::mesh_dim_y);
    
    // 计算在当前层中的剩余偏移
    int offset = id - coord.z*NoximGlobalParams::mesh_dim_x*NoximGlobalParams::mesh_dim_y;
    
    // 计算y坐标（行数）
    coord.y = offset / NoximGlobalParams::mesh_dim_x;
    
    // 计算x坐标（列数）
    coord.x = offset % NoximGlobalParams::mesh_dim_x;
    
    // 验证坐标范围的有效性
    assert(coord.x < NoximGlobalParams::mesh_dim_x);
    assert(coord.y < NoximGlobalParams::mesh_dim_y);
    assert(coord.z < NoximGlobalParams::mesh_dim_z);
    
    return coord;
}
```

### 2. coord2Id 函数：坐标转ID
```cpp
inline int coord2Id(const NoximCoord & coord)
{
    // 使用三维数组行优先索引计算
    int id = coord.z*NoximGlobalParams::mesh_dim_x*NoximGlobalParams::mesh_dim_y + 
             (coord.y * NoximGlobalParams::mesh_dim_x) + 
             coord.x;
             
    // 验证ID的有效性
    assert(id < NoximGlobalParams::mesh_dim_x * NoximGlobalParams::mesh_dim_y * NoximGlobalParams::mesh_dim_z);
    
    return id;
}
```

### 功能与应用

1. **行优先三维寻址**：采用标准的行优先寻址方式，先从z维开始，然后是y，最后是x
   
2. **转换公式**：
   - ID = z×(X_SIZE×Y_SIZE) + y×X_SIZE + x
   - Z = ID÷(X_SIZE×Y_SIZE)
   - Y = (ID - Z×X_SIZE×Y_SIZE)÷X_SIZE
   - X = (ID - Z×X_SIZE×Y_SIZE) mod X_SIZE

3. **用途**：这些函数在路由算法、节点通信和性能分析中频繁使用，是三维NoC仿真的核心部分

4. **边界检查**：使用assert确保计算结果在有效范围内，增强代码的健壮性

## 三维网络芯片坐标与ID转换函数分析

这段代码定义了网络芯片中节点的一维ID与三维坐标间的转换函数：

### 1. id2Coord 函数：ID转坐标
```cpp
inline NoximCoord id2Coord(int id)
{
    NoximCoord coord;
    
    // 计算z坐标（层数）
    coord.z = id / (NoximGlobalParams::mesh_dim_x*NoximGlobalParams::mesh_dim_y);
    
    // 计算在当前层中的剩余偏移
    int offset = id - coord.z*NoximGlobalParams::mesh_dim_x*NoximGlobalParams::mesh_dim_y;
    
    // 计算y坐标（行数）
    coord.y = offset / NoximGlobalParams::mesh_dim_x;
    
    // 计算x坐标（列数）
    coord.x = offset % NoximGlobalParams::mesh_dim_x;
    
    // 验证坐标范围的有效性
    assert(coord.x < NoximGlobalParams::mesh_dim_x);
    assert(coord.y < NoximGlobalParams::mesh_dim_y);
    assert(coord.z < NoximGlobalParams::mesh_dim_z);
    
    return coord;
}
```

### 2. coord2Id 函数：坐标转ID
```cpp
inline int coord2Id(const NoximCoord & coord)
{
    // 使用三维数组行优先索引计算
    int id = coord.z*NoximGlobalParams::mesh_dim_x*NoximGlobalParams::mesh_dim_y + 
             (coord.y * NoximGlobalParams::mesh_dim_x) + 
             coord.x;
             
    // 验证ID的有效性
    assert(id < NoximGlobalParams::mesh_dim_x * NoximGlobalParams::mesh_dim_y * NoximGlobalParams::mesh_dim_z);
    
    return id;
}
```

### 功能与应用

1. **行优先三维寻址**：采用标准的行优先寻址方式，先从z维开始，然后是y，最后是x
   
2. **转换公式**：
   - ID = z×(X_SIZE×Y_SIZE) + y×X_SIZE + x
   - Z = ID÷(X_SIZE×Y_SIZE)
   - Y = (ID - Z×X_SIZE×Y_SIZE)÷X_SIZE
   - X = (ID - Z×X_SIZE×Y_SIZE) mod X_SIZE

3. **用途**：这些函数在路由算法、节点通信和性能分析中频繁使用，是三维NoC仿真的核心部分

4. **边界检查**：使用assert确保计算结果在有效范围内，增强代码的健壮性

## Main函数分析
# Noxim 网络芯片模拟器初始化和波形追踪模块

这段代码实现了 Noxim 模拟器的两个主要功能模块：系统初始化和波形追踪配置。

### 1. 系统初始化部分

```cpp
// 初始化全局变量
drained_volume = 0;

// 显示欢迎信息
cout << endl << "\t\tNoxim - the NoC Simulator" << endl;
cout << "\t\t(C) University of Catania" << endl << endl;

// 解析命令行参数进行配置
parseCmdLine(arg_num, arg_vet);

// 一些被注释掉的调试代码，曾用于测试多维数组/队列
// ...

// 创建SystemC核心信号
sc_clock clock("clock", 1, SC_NS);  // 1纳秒周期的时钟
sc_signal <bool> reset;             // 重置信号

// 创建并连接NoC网络实例
NoximNoC *n = new NoximNoC("NoC");
n->clock(clock);
n->reset(reset);
```

### 2. 波形追踪配置部分

```cpp
// 初始化追踪文件指针
sc_trace_file *tf = NULL;

// 只有当用户通过命令行启用了trace_mode才执行追踪
if (NoximGlobalParams::trace_mode) {
    // 创建VCD格式波形文件
    tf = sc_create_vcd_trace_file(NoximGlobalParams::trace_filename);
    
    // 追踪全局控制信号
    sc_trace(tf, reset, "reset");
    sc_trace(tf, clock, "clock");
    
    // 使用三重循环遍历三维网络中的每个路由器节点
    for (i = 0; i < NoximGlobalParams::mesh_dim_x; i++) {
        for (j = 0; j < NoximGlobalParams::mesh_dim_y; j++) {
            for (k = 0; k < NoximGlobalParams::mesh_dim_z; k++) {
                char label[30];
                
                // 追踪各方向的请求信号
                sprintf(label, "req_to_east(%02d)(%02d)(%02d)", i, j, k);
                sc_trace(tf, n->req_to_east[i][j][k], label);
                // ...其他方向的请求信号...
                
                // 追踪各方向的确认信号
                sprintf(label, "ack_to_east(%02d)(%02d)(%02d)", i, j, k);
                sc_trace(tf, n->ack_to_east[i][j][k], label);
                // ...其他方向的确认信号...
            }
        }
    }
    // 注释部分是旧的2D网络追踪代码
}
```

1. **初始化环境** - 设置全局参数、解析命令行选项
2. **创建网络实例** - 实例化三维NoC网络及其连接
3. **波形追踪** - 可选功能，记录网络内部信号变化：
   - 生成VCD文件供波形分析工具使用
   - 追踪每个路由器节点的通信信号
   - 支持三维坐标系下的信号标识

### 3. 输出文件配置

```cpp
// 功耗追踪文件设置
char temperal[20];
string pwr_filename = string("results/POWER/PWR");

// 将数据包注入率添加到文件名
sprintf(temperal, "%f", NoximGlobalParams::packet_injection_rate);
pwr_filename = pwr_filename + "_pir-" + temperal;

// 将流量分布模式添加到文件名
sprintf(temperal, "%d", NoximGlobalParams::traffic_distribution);
pwr_filename = pwr_filename + "_traffic-" + temperal + ".txt";

// 打开功耗记录文件
results_log_pwr.open(pwr_filename.c_str(), ios::out);
if(!results_log_pwr.is_open())
    cout<<"Cannot open "<< pwr_filename.c_str() <<endl;

// 创建每个节点三种功耗指标的列标题
for(int z = 0; z < NoximGlobalParams::mesh_dim_z; z++){
    for(int y = 0; y < NoximGlobalParams::mesh_dim_y; y++){
        for(int x = 0; x < NoximGlobalParams::mesh_dim_x; x++){
            results_log_pwr<<"router["<<x<<"]["<<y<<"]["<<z<<"]\t";
            results_log_pwr<<"uP_mem["<<x<<"]["<<y<<"]["<<z<<"]\t";
            results_log_pwr<<"uP_mac["<<x<<"]["<<y<<"]["<<z<<"]\t";
        }
    }
}
```

类似地，为缓冲区状态创建输出文件：

```cpp
// 缓冲区状态文件设置
string buffer_filename = string("results/buffer/buffer");
// 文件命名类似功耗文件...
// ...
```

### 4. 芯片复位与随机种子设置

```cpp
// 开始复位过程
cout << "Reset...";
reset.write(1);  // 激活复位信号

// 设置随机数生成器种子，用于确保模拟可重现性
srand(NoximGlobalParams::rnd_generator_seed);

// 运行复位阶段，持续DEFAULT_RESET_TIME个周期
sc_start(DEFAULT_RESET_TIME * CYCLE_PERIOD, SC_NS);
reset.write(0);  // 释放复位信号
```

### 5. 执行网络仿真

```cpp
// 提示开始模拟
cout << " done! Now running for " << NoximGlobalParams::simulation_time 
     << " cycles..." << endl;
     
// 启动仿真器，运行指定的周期数
sc_start(NoximGlobalParams::simulation_time * CYCLE_PERIOD, SC_NS);
```

# Noxim 模拟结果收集与统计分析模块

这段代码实现了模拟完成后的结果收集、统计与报告生成功能，是整个模拟过程的最后阶段。

### 6. 模拟结束与清理
```cpp
// 关闭波形追踪文件（如果启用了追踪）
if (NoximGlobalParams::trace_mode)
    sc_close_vcd_trace_file(tf);

// 输出模拟完成信息和执行的周期数
cout << "Noxim simulation completed." << endl;
cout << " ( " << getCurrentCycleNum() << " cycles executed)" << endl;
```

### 7. 缓冲区状态收集
这部分收集网络中每个路由器各个方向缓冲区的详细状态：

```cpp
// 记录所有路由器各方向缓冲区的空闲槽位
results_buffer << "freeslots" << endl;
for(int d = 0; d < DIRECTIONS+1; d++) {
    // 遍历3D网格中的所有路由器
    for(int z=0; z < NoximGlobalParams::mesh_dim_z; z++) 
        for(int y=0; y < NoximGlobalParams::mesh_dim_y; y++) 
            for(int x=0; x < NoximGlobalParams::mesh_dim_x; x++) {
                // 记录当前方向缓冲区的空闲槽数
                results_buffer << n->t[x][y][z]->r->buffer[d].getCurrentFreeSlots() << "\t";
            }
    results_buffer << "\n";
}

// 记录缓冲区中数据包的等待时间、目的地ID、源ID等信息
// ...类似的循环结构

// 记录路由器预留表和开/关状态
// ...类似的循环结构
```

### 8. 传输统计汇总
```cpp
// 计算各种传输类型的总量
double total_transmit=0;
double total_not_transmit=0;
double total_adaptive_transmit=0;
double total_dor_transmit=0;
double total_dw_transmit =0;

// 遍历整个网络收集统计数据
for(int z=0; z < NoximGlobalParams::mesh_dim_z; z++) {
    for(int y = 0; y < NoximGlobalParams::mesh_dim_y; y++) {
        for(int x = 0; x < NoximGlobalParams::mesh_dim_x; x++) {
            // 累加各种传输类型的计数
            total_transmit += n->t[x][y][z]->pe->transmit;
            total_not_transmit += n->t[x][y][z]->pe->not_transmit;
            // ...其他传输类型
        }
    }
}

// 输出传输统计数据和比率
results_buffer << "transmit:" << "\t" << total_transmit << "\n";
results_buffer << "not_transmit:" << "\t" << total_not_transmit << "\n";
results_buffer << "Need retransmit rate:" << "\t" << total_not_transmit/total_transmit << "\n";
// ...其他传输类型统计
```

### 9. 空间流量负载分布（STLD）
```cpp
// 创建STLD文件
string STLD_filename = string("results/STLD/STLD");
// 文件命名包含注入率和流量模式
// ...

// 记录网格尺寸
results_STLD << NoximGlobalParams::mesh_dim_x << " " 
             << NoximGlobalParams::mesh_dim_y << " " 
             << NoximGlobalParams::mesh_dim_z << endl;

// 记录每个节点路由的分片数
for(int z=0; z < NoximGlobalParams::mesh_dim_z; z++) {
    for(int y=0; y < NoximGlobalParams::mesh_dim_y; y++)
        for(int x=0; x < NoximGlobalParams::mesh_dim_x; x++)
            results_STLD << n->t[x][y][z]->r->routed_flits << "\t";
    results_STLD << "\n";
}
```

### 10. 全局统计与完成验证
```cpp
// 创建全局统计对象并显示结果
NoximGlobalStats gs(n);
gs.showStats(std::cout, NoximGlobalParams::detailed);

// 检查是否达到了指定的数据排空量
if ((NoximGlobalParams::max_volume_to_be_drained > 0) && 
    (getCurrentCycleNum() / 1000 >= NoximGlobalParams::simulation_time * CYCLE_PERIOD)) {
    // 显示警告，建议增加模拟时间
    // ...
}

// 关闭输出文件
results_log_pwr.close();
```

1. **多维度数据收集**：从多个角度记录网络状态（缓冲区使用率、等待时间、路由表等）
2. **分类统计**：区分不同传输类型（自适应、确定性等）进行统计
3. **空间分析**：通过STLD文件记录空间流量分布，支持热点分析
4. **完整性检查**：验证是否完成了预期的数据传输量
5. **结构化输出**：各类统计数据按特定格式保存，便于后续分析工具处理
