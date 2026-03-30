# MiaoSuan Python API 文档

本文档描述 `miaosuan` 包对外暴露的核心 API，供模型开发时查阅。除特别说明外，所有函数均可通过顶层命名空间调用，例如：

```python
import miaosuan as ms
ms.sim_time()
```

---

## 核心概念

### 对象层次

妙算引擎的仿真对象呈树状层次结构：

```
Scenario（场景）
  └── Platform（平台）
        └── Device（设备）
              └── Module（模块）  ← 进程执行所在的基本单元
```

- **`SimObj`**：所有仿真对象的基类，管理 ID、父对象、子对象及属性。
- **`Module(SimObj)`**：模块，是进程运行的容器，同时也是流（Stream）、通知线（NotifyLine）的持有者。`Module` 有多种类型，对应不同的仿真行为（处理器、队列、无线收发等），通过 `OBJ_TYPE_*` 常量区分。
- **`Process`**：进程，描述一个状态机（FSM），通过 `ProcessBuilder` API 或 `@process_model` 装饰器构建。
- **`Packet`**：数据包，承载模块间交互的数据负载。
- **`Event`**：事件，驱动仿真时钟推进的基本单元。

### 进程模型编写方式

有两种方式构建进程模型：

**方式一：手动构建（`ProcessBuilder`）**

通过 `ProcessBuilder.begin()`、`add_state()`、`add_transition()` 手动拼接状态机。适合需要动态行为的场景。

**方式二：声明式装饰器（`@process_model`）**

使用 `@ms.process_model`、`@ms.state_enter`、`@ms.state_exit`、`@ms.transition` 装饰器在类上声明状态机，引擎自动完成拼接。推荐作为主要建模方式。

两种方式可混合使用。

---

## 模型管理（`model_manager`）

- `init_model_manager(bases: Iterable[str], token: str, workspace: str) -> None`
  - 参数 `bases`: 模型仓库的本地路径或 HTTP/HTTPS URL；可同时配置多个基址。
  - 参数 `token`: 访问远程模型仓库时附带的 JWT 字符串，可为空。
  - 参数 `workspace`: 访问远程模型仓库时附带的 workspace id，可为空字符串。
  - 说明: 初始化模型加载器并清空缓存，仿真启动前调用一次。
- `register_process_model(name: str, generator: Callable[[], FsmProcess]) -> None`
  - 参数 `name`: 进程模型名称，需要与场景模型中的引用一致。
  - 参数 `generator`: 返回实现 `run()`/`init()` 的 FSM 实例的可调用对象。
  - 说明: 在 Python 层注册进程模型实现，供 `pro_create()` 使用。
- `register_pipeline_stage_model(name: str, stage: Any) -> None`
  - 参数 `name`: 流水线阶段模型名称。
  - 参数 `stage`: 阶段实现对象或可调用；若可调用会被包装为阶段对象。
  - 说明: 注册流水线阶段模型，供模型管理器按名称查询。

---

## 进程模型装饰器（`@process_model` 声明式 DSL）

- `@process_model(name: str)`（类装饰器）
  - 说明: 声明并注册一个进程模型类。使用 `@state_enter`、`@state_exit`、`@transition` 装饰类的方法，引擎会在类初始化时自动拼接状态机，并将该类注册到模型管理器中。
  - 示例：

    ```python
    @ms.process_model("my_proc")
    class MyProcess:
        @ms.state_enter("Idle", begin=True)
        def enter_idle(self) -> None:
            ...

        @ms.state_exit("Idle")
        def exit_idle(self) -> None:
            ...

        @ms.transition("Idle", "Send")
        def should_send(self) -> bool:
            return True
    ```

- `@state_enter(state: str, begin: bool = False)`（方法装饰器）
  - 参数 `state`: 所属状态名称。
  - 参数 `begin`: 若为 `True`，标记该状态为 FSM 的初始状态。
  - 说明: 将方法标记为某状态的**进入动作**。与 `@process_model` 配合使用；类中若自行定义了 `init` 方法则自动拼接被跳过。

- `@state_exit(state: str)`（方法装饰器）
  - 参数 `state`: 所属状态名称。
  - 说明: 将方法标记为某状态的**退出动作**。

- `@transition(from_state: str, to_state: str, transition_id: Optional[str] = None)`（方法装饰器）
  - 参数 `from_state`: 转移的起始状态名称。
  - 参数 `to_state`: 转移的目标状态名称。
  - 参数 `transition_id`: 可选，用于工具链（编辑器等）区分同一对状态间的多条转移线。
  - 说明: 将方法标记为某转移的**条件函数**。被装饰的方法必须是接受 0 个参数、返回 `bool` 的实例方法。

- `@pipeline_stage(name: str)`（函数装饰器）
  - 参数 `name`: 流水线阶段模型名称。
  - 说明: 声明并注册一个流水线阶段函数/可调用对象。
  - 示例：

    ```python
    @ms.pipeline_stage("generic_wireless_closure")
    def generic_wireless_closure(packet) -> None:
        ...
    ```

---

## 仿真驱动与时间管理（`simulation`, `api`）

- `run_sim(name: str, params: dict, duration: float) -> None`
  - 参数 `name`: 场景模型名称（不含后缀）。
  - 参数 `params`: 仿真参数字典，包含统计配置等。
  - 参数 `duration`: 目标仿真时长（秒）。
  - 说明: 构建场景、启动事件循环并在达到时长或事件为空时结束。
- `tick_once() -> Optional[Event]`
  - 说明: 推进事件队列一次，返回已处理的事件；可用于手动驱动仿真。
- `queue_size() -> int`
  - 说明: 返回当前待处理事件数。
- `sim_time() -> float`
  - 说明: 返回当前仿真时间（秒）。

---

## 中断类型常量（`engine`）

- `INTRPT_TYPE_SELF = 1`：模块自发调度的事件（通过 `intrpt_schedule_self`）。
- `INTRPT_TYPE_STRM = 2`：数据包到达事件（通过输入流 `Stream` 触发）。
- `INTRPT_TYPE_BEGSIM = 3`：仿真开始事件，在时钟推进至大于 0 时触发。
- `INTRPT_TYPE_REMOTE = 4`：远程中断事件（通过 `intrpt_schedule_remote` 触发）。

---

## 模块上下文与中断（`api`）

- `self_obj() -> Optional[SimObj]`
  - 说明: 返回当前执行上下文所属的模块对象，若无模块则为 `None`。在进程进入/退出动作中可用。
- `id_self() -> int`
  - 说明: 获取当前模块的对象 ID；无上下文时返回 `-1`。
- `intrpt_schedule_self(time: float, code: int) -> Event`
  - 参数 `time`: 触发仿真时间。
  - 参数 `code`: 用户自定义事件代码。
  - 说明: 为当前模块调度自发中断事件。
- `intrpt_schedule_remote(time: float, code: int, module: SimObj) -> Event`
  - 参数 `module`: 目标模块对象（通常为 `Module` 类型）。
  - 说明: 为其他模块调度远程中断事件，并继承当前模块安装的 ICI。
- `intrpt_type() -> int` / `intrpt_code() -> int` / `intrpt_strm() -> int`
  - 说明: 读取当前正在处理的事件类型、代码或关联流索引。事件类型对应 `INTRPT_TYPE_*` 常量。
- `intrpt_ici() -> Ici`
  - 说明: 返回当前事件附带的 ICI 数据对象。
- `ici_create(fmt_name: str) -> Ici`
  - 说明: 根据 ICI 模型名称创建 ICI 对象。
- `ici_install(ici: Ici) -> None`
  - 说明: 将 ICI 安装到当前模块，后续发送的数据包或事件会携带该 ICI。传入 `None` 可清除已安装的 ICI。

---

## 进程模型 API（`ProcessBuilder`, `pro_*`）

- `ProcessBuilder(fsm: FsmProcess, parent: Optional[Process] = None)`
  - 方法：
    - `begin(state: str) -> ProcessBuilder`: 设置起始状态。
    - `add_state(name: str, enter_func: Optional[Callable], exit_func: Optional[Callable]) -> ProcessBuilder`: 注册状态及进入/退出动作。
    - `add_transition(from_state: str, to_state: str, condition: Callable[[], bool], execution: Optional[Callable]) -> ProcessBuilder`: 定义状态转移及条件。
- `pro_self() -> Optional[Process]`
  - 说明: 返回当前正在执行的进程对象。
- `pro_self_id() -> int`
  - 说明: 返回当前进程 ID。
- `pro_root(module: Module) -> Optional[Process]`
  - 说明: 获取模块的根进程。
- `pro_equal(a: Optional[Process], b: Optional[Process]) -> bool`
  - 说明: 判断两个进程句柄是否相同。
- `pro_id(proc: Process) -> int`
  - 说明: 返回指定进程的 ID。
- `pro_parent(proc: Process) -> Optional[Process]`
  - 说明: 返回进程的父进程（若有）。
- `pro_create(name: str, shared_memory: Any) -> Optional[Process]`
  - 说明: 依据已注册的进程模型名称创建子进程，返回进程句柄。
- `pro_parent_mem_access() -> Any`
  - 说明: 访问当前进程父进程的共享内存。
- `pro_mod_mem_install(user_data: Any) -> None` / `pro_mod_mem_access() -> Any`
  - 说明: 在当前模块安装并读取模块级共享数据。
- `pro_invoke(handle: Process, arg_mem: Any) -> None`
  - 说明: 在当前模块中调用另一个进程，将 `arg_mem` 作为参数内存传递。
- `pro_invoker(handle: Process) -> Tuple[Optional[Process], int]`
  - 说明: 返回调用者进程及调用类型，调用类型常量为 `PROINV_DIRECT`, `PROINV_INDIRECT`, `PROINV_ERROR`。
- `pro_module(handle: Process) -> Optional[Module]`
  - 说明: 获取进程所属模块。
- `pro_arg_mem_access() -> Any`
  - 说明: 在被调用进程内部取回调用参数。

---

## 通知管道（`notify`）

- `notify_set_int(name: str, value: int) -> None` / `notify_set_int64` / `notify_set_float64` / `notify_set_string` / `notify_set_bool`
  - 说明: 在当前模块的输出通知线上设置值，类型需与通知定义匹配。若类型不匹配会抛出 `TypeError`。
- `notify_get(name: str) -> Any`
  - 说明: 读取输入通知并在未设置时抛出 `RuntimeError`。
- `notify_get_int(name: str) -> int` / `notify_get_int64` / `notify_get_float64` / `notify_get_string` / `notify_get_bool`
  - 说明: 类型安全的读取接口，类型不匹配会抛出 `TypeError`。
- `notify_value_type_from_string(type_str: str) -> NotifyValueType`
  - 说明: 将字符串表示转换为 `NotifyValueType` 枚举值。支持的字符串：`int`、`int64`、`float64`、`string`、`bool`（及其别名如 `integer`、`double` 等）。
- `notify_value_type_to_string(val_type: NotifyValueType) -> str`
  - 说明: 将 `NotifyValueType` 枚举值转换回规范字符串。

---

## 数据包与链路（`packet`）

- 创建与格式
  - `pk_create() -> Packet`: 创建未格式化数据包。
  - `pk_create_fmt(format_name: str) -> Packet`: 按模型名称加载格式并创建数据包。
  - `pk_format(packet: Packet) -> str`: 返回数据包格式名称。
  - `pk_id(packet: Packet) -> int`: 获取数据包唯一标识。
- 大小与优先级
  - `pk_total_size_get(packet: Packet) -> int` / `pk_total_size_set(packet, size_bits) -> None`: 读取或设置数据包总比特数（bit），总大小 = 所有字段大小之和 + bulk 大小。
  - `pk_bulk_size_get(packet: Packet) -> int` / `pk_bulk_size_set(packet, size_bits) -> None`: 读取或设置 bulk 大小（bit，可为负），用于微调总大小。
  - `pk_priority_get(packet: Packet) -> int` / `pk_priority_set(packet, priority: int) -> None`: 读取或设置优先级。
- 创建信息
  - `pk_creation_time_get(packet) -> float` / `pk_creation_time_set(packet, time_val) -> None`: 访问创建时间。
  - `pk_creation_module_get(packet) -> Optional[Module]` / `pk_creation_module_set(packet, module) -> None`: 管理创建模块。
- 时间戳
  - `pk_stamp(packet: Packet) -> None`: 使用当前仿真时间和模块进行标记。
  - `pk_stamp_time_get(packet) -> float` / `pk_stamp_time_set(packet, time_val) -> None`: 访问或设置标记时间。
  - `pk_stamp_module_get(packet) -> Optional[Module]` / `pk_stamp_module_set(packet, module) -> None`: 访问或设置标记模块。
- ICI 与生命周期
  - `pk_ici(packet: Packet) -> Optional[Ici]`: 返回随包携带的 ICI。
  - `pk_copy(packet: Packet) -> Packet`: 深拷贝数据包（包含嵌套包）。
  - `pk_destroy(packet: Packet) -> None`: 清理内部状态。
- 名称字段（NFD）— 非类型化
  - `pk_nfd_set(packet, name, value)` / `pk_nfd_get(packet, name)`: 通过字段名读写数据包字段值，不做类型校验。
- 名称字段（NFD）— 强类型化
  - 设置: `pk_nfd_set_string(packet, name, value)` / `pk_nfd_set_int(packet, name, value)` / `pk_nfd_set_int64(packet, name, value)` / `pk_nfd_set_float64(packet, name, value)` / `pk_nfd_set_packet(packet, name, value)` / `pk_nfd_set_object(packet, name, value)` / `pk_nfd_set_pointer(packet, name, value)`
  - 读取: `pk_nfd_get_string(packet, name)` / `pk_nfd_get_int(packet, name)` / `pk_nfd_get_int64(packet, name)` / `pk_nfd_get_float64(packet, name)` / `pk_nfd_get_packet(packet, name)` / `pk_nfd_get_object(packet, name)` / `pk_nfd_get_pointer(packet, name)`
  - 说明: 类型化接口在类型不匹配时抛出 `TypeError`，适合需要静态类型保障的场景。
- 数据链路操作
  - `pk_send(packet: Packet, stream_index: int) -> None`: 在当前模块的输出流上发送数据包并触发远端事件。
  - `pk_send_quiet(packet: Packet, stream_index: int) -> None`: 将数据包入队但不立即触发事件。
  - `pk_get(stream_index: int) -> Packet`: 从输入流队列取包，若为空会抛出异常。
  - `pk_deliver_delayed(packet: Packet, target_module: Module, in_stream_index: int, delay: float) -> None`: 将数据包延迟投递到目标模块的输入流。

---

## 传输数据属性（`tda`）

- 常量分组
  - 点到点收发：`TDA_PT_*`（如 `TDA_PT_BG_DELAY`, `TDA_PT_TX_DELAY` 等）定义索引。
  - 总线收发：`OPC_TDA_BU_*` 系列。
  - 无线收发：`OPC_TDA_RA_*` 系列及匹配状态常量（`OPC_TDA_RA_MATCH_*`）。
- 数据访问
  - `td_get(packet: Packet, index: int) -> Any`: 根据索引读取数据包的 TDA。
  - `td_set(packet: Packet, index: int, value: Any) -> None`: 写入 TDA 值。

---

## 仿真对象（`simobj`）

- 类型常量：`OBJ_TYPE_PROCESSOR`、`OBJ_TYPE_QUEUE`、`OBJ_TYPE_RA_RX`、`OBJ_TYPE_RA_TX`、`OBJ_TYPE_RA_TX_CH`、`OBJ_TYPE_RA_RX_CH`、`OBJ_TYPE_NODE`、`OBJ_TYPE_PLATFORM`、`OBJ_TYPE_SCENARIO`、`OBJ_TYPE_P2P_TX_CH`、`OBJ_TYPE_P2P_RX_CH`、`OBJ_TYPE_P2P_TX`、`OBJ_TYPE_P2P_RX`、`OBJ_TYPE_LINK`。

- **`SimObj`**（仿真对象基类）
  - `get_id() -> int`: 返回对象 ID。
  - `get_parent() -> Optional[SimObj]` / `get_children() -> List[SimObj]` / `get_child(name: str) -> Optional[SimObj]`: 访问拓扑关系。
  - `get_obj_type() -> int`: 返回对象类型常量。
  - `add_child(name: str, child: SimObj) -> None`: 将子对象加入当前对象。
  - `add_attr_defs(defs: List[AttributeDef]) -> None`: 向对象追加属性定义。
  - 属性读写：`get_attr_int|int64|double|string|bool|sim_obj(name)` / `get_attr_obj(name)` / `set_attr_*`。
  - `get_attr_array_count(name: str) -> int`: 返回数组类型属性的元素个数。
  - `get_attr_value(name: str) -> Any` / `set_attr_value(name: str, value: Any) -> None`: 按路径访问（支持 `.` 与 `[]`），例如 `"items[0].price"`。
  - `set_child_attr_value(child_name: str, attr_name: str, value: Any) -> None`: 修改子节点属性。
  - `set_attr_value_recursively(name: str, value: Any, id_name_map: Optional[Dict[int, Dict[str, str]]] = None) -> None`: 递归设置属性。
  - **平台专用**: `set_pos(x: float, y: float, alt: float) -> None` / `get_pos() -> Tuple[float, float, float]`：设置或读取平台位置（单位：度、度、米）。读取时会根据附加的轨道（Orbit）或轨迹（Trajectory）自动计算当前时刻的位置。

- `get_sim_obj(obj_id: int) -> Optional[SimObj]`
  - 说明: 通过 ID 查询模拟对象实例。

---

## 拓扑帮助函数（`topology`）

- `topo_parent(obj: SimObj) -> Optional[SimObj]`: 返回父对象。
- `topo_child_count(obj: SimObj, child_type: int) -> int`: 统计指定类型的子对象数量。
- `topo_child(obj: SimObj, child_type: int, index: int) -> Optional[SimObj]`: 按类型与索引获取子对象。
- `get_in_streams() -> Dict[int, Stream]` / `get_out_streams() -> Dict[int, Stream]`: 在当前模块上下文中返回输入/输出流映射（键为流索引，值为 `Stream` 对象，含 `src`、`dst` 等字段）。
- `is_module_connected(module1: SimObj, module2: SimObj) -> bool`: 判断两个模块是否直接连接。

---

## 通用工具（`utils`）

- `convert_lon_lat_alt_to_geo(lon: float, lat: float, alt: float) -> Tuple[float, float, float]`: 将 WGS84 经纬高转换为 ECEF 地心坐标（米）。
- `convert_geo_to_lon_lat_alt(x: float, y: float, z: float) -> Tuple[float, float, float]`: 将 ECEF 地心坐标转换为 WGS84 经纬高。
- `get_distance_2d(lon: float, lat: float, lon2: float, lat2: float) -> float`: 计算两经纬度点的大圆距离（米，Haversine 公式）。
- `get_distance_geo2d(x: float, y: float, x2: float, y2: float) -> float`: 计算 ECEF XY 平面距离（米）。
- `get_distance_geo3d(x: float, y: float, z: float, x2: float, y2: float, z2: float) -> float`: 计算 ECEF 三维欧氏距离（米）。
- `get_distance_3d(lon: float, lat: float, alt: float, lon2: float, lat2: float, alt2: float) -> float`: 经纬高坐标下的三维距离（米）。

---

## 调制解调曲线（`modulation`）

调制解调曲线模型用于描述 `SNR -> BER` 的映射关系，模型文件后缀为 `.md.m`，内容为标准 JSON：

```json
{
  "x": [-10, -9, -8],
  "y": [0.33, 0.32, 0.30]
}
```

其中 `x` 表示信噪比 `snr`，`y` 表示误比特率 `ber`。引擎侧会对输入数据做校验（`x/y` 等长、非空、数值有限；`x` 不允许重复）。

- `mod_table_load(name: str) -> ModulationHandle`
  - 参数 `name`: 调制解调模型名称（不含后缀），实际加载文件为 `${name}.md.m`。
  - 返回: 可用于查询的 `ModulationHandle`。
- `mod_ber_get(handle: ModulationHandle, snr: float) -> float`
  - 说明: 根据 `snr` 从曲线中查询 `ber`（线性插值）。当 `snr` 小于最小 `x` 时返回最小 `x` 对应的 `y`，大于最大 `x` 时同理。
  - 示例：

    ```python
    import miaosuan as ms
    h = ms.mod_table_load("models/modulations/qpsk")
    ber = ms.mod_ber_get(h, snr=3.5)
    ```

---

## 统计系统（`stats`）

- 配置常量：`STATS_PARAM_ENABLED`、`STATS_PARAM_MODE`、`STATS_PARAM_FILTERS`、`STATS_PARAM_FILE_PATH`、`STATS_PARAM_HUIHENG_URL`、`STATS_PARAM_HUIHENG_CLIENT`。
- 枚举类型：
  - `StatsSinkMode`: `OFF`、`FILE`、`HUIHENG`、`BOTH`。
  - `StatCaptureMode`: `ALL`、`BUCKET`。
  - `StatAgg`: `MAX`、`MIN`、`SUM`、`COUNT`、`SAMPLE_MEAN`、`SUM_PER_TIME`。
- `StatMode`
  - 字段：`mode`、`every_seconds`、`every_n_values`、`total_captures`、`agg`。
  - 说明: 描述统计采样模式，传入 `register_stat` 时可定制采样频率与聚合方式。
- `stats_init(params: Dict[str, object], sim_name: str, duration: float) -> None`
  - 说明: 解析统计配置并初始化输出目标；在仿真开始时调用。
- `stats_shutdown() -> None`
  - 说明: 关闭并清理所有统计输出，通常由引擎自动调用。
- `register_stat(name_template: str, mode: StatMode) -> StatHandle`
  - 说明: 按模板名称与模式注册指标，返回 `record(value: float)` 可调用的句柄；若不满足过滤规则则返回空操作句柄。
