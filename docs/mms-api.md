# MiaoSuan Model Support (MMS) API 文档

本文档梳理 `miaosuan.mms` 包当前对外暴露的接口，供模型开发和联调时参考。模块命名遵循 Python 包路径，例如 `miaosuan.mms.auto_addr`。除特别说明外，示例中出现的 `SimObj`、`Module` 等类型定义于 `miaosuan` 命名空间。

## 地址自动分配（`miaosuan.mms.auto_addr`）

**用途**：在构建链路或总线类模块时，为 MAC/接口对象提供自动或手动的地址分配能力，同时维护广播地址、未初始化地址等特殊取值。

### 常量

- `AA_AUTO_ASSIGN = -2`：请求动态地址，且在分配后将数值写入模块的指定整型属性。
- `AA_NO_ATTR = -4`：请求动态地址，但不会回写属性（适用于无属性存储能力的对象）。
- `AA_UNINIT_ADDR = -3`：通用的“未初始化地址”占位。
- `AA_BROADCAST = -1`：广播地址常量。
- `AA_ASSIGNMENT_DYNAMIC = 0` / `AA_ASSIGNMENT_STATIC = 1` / `AA_ASSIGNMENT_DYNAMIC_NO_ATTR = 2`：`AddressInfo.assignment_type` 取值，用于区分分配策略。
- `DEST_STATUS_VALID = 0` / `DEST_STATUS_INVALID = 1`：目标可达性标记，供 `AddressHandle.set_destination_status()` 使用。

### 异常

- `AutoAddressError(RuntimeError)`：在地址冲突、句柄已完成分配等非法状态下抛出。

### 地址句柄 `AddressHandle`

`AddressHandle` 由 `aa_address_handle_get()` 返回，是地址分配的状态载体。句柄在第一次创建时会记录待写入的属性名，并跟踪注册对象的数量；当所有对象都已经分配、登记或声明为未连接后，句柄自动进入“finalized”状态，并冻结最终地址表。

常用方法：

- `resolve(module: SimObj, address: int) -> int`：为单个模块分配地址。传入动态常量时会自动取空闲槽位；传入显式地址时若发生冲突会抛出 `AutoAddressError`。
- `verify_ring(modules: Sequence[SimObj], first_address: int) -> None`：登记一组按顺序静态编号的环；用于验证固定拓扑的地址配置。
- `resolve_ring(modules: Sequence[SimObj]) -> None`：登记一组待动态分配的环成员，最终地址会在句柄冻结时顺序给出。
- `register_unconnected_mac() -> None`：声明一个无需分配地址的 MAC，用于平衡计数使句柄正确完成。
- `get_address(module: SimObj) -> Optional[int]`：在句柄冻结后查询某模块的最终地址。
- `find_address_index(address: int) -> int`：在最终排序表中定位地址索引；未找到返回 `-1`。
- `is_valid_destination(address: int) -> bool` / `set_destination_status(address: int, status: int) -> None`：在冻结之后检查或更新目标可达性。
- `get_random_destination(source_address: int) -> int`：随机挑选一个仍然有效且不等于源地址的目的地址；若尚未冻结或没有可用地址则返回 `AA_UNINIT_ADDR`。
- `get_random_destination_from_range(low: int, high: int, source_address: int) -> int`：在给定的索引区间内随机挑选可达目的地址。

> **注意**：调用 `resolve*`/`verify_ring` 等方法必须发生在句柄冻结之前；一旦 `_assign_count + _unconnected_count == _elem_count` 条件满足，句柄会立即冻结，再次调用会触发 `AutoAddressError` 或忽略操作。

### 函数

- `aa_address_handle_get(address_category: str, attribute_name: str) -> AddressHandle`
  - 参数 `address_category`：地址类别（例如总线名称），用于在全局句柄表中索引/复用地址分配状态。
  - 参数 `attribute_name`：在动态分配成功后要写回的模块整型属性名称。
  - 说明：首次调用会创建新句柄；后续同类调用会复用句柄并将预期对象计数加一。若尝试以不同属性名复用同一类别将抛出 `AutoAddressError`。

- `aa_address_resolve(handle: AddressHandle, module: SimObj, address: int) -> int`
  - 说明：包装 `handle.resolve()`，根据传入地址类型完成静态或动态分配，返回最终地址。动态分配时若使用 `AA_AUTO_ASSIGN` 会调用 `module.set_attr_int(attribute_name, slot)` 写回。

- `aa_address_verify_ring(handle: AddressHandle, modules: Sequence[SimObj], first_address: int) -> None`
  - 说明：包装 `handle.verify_ring()`，用于校验一个固定顺序环是否与静态地址配置一致。

- `aa_address_resolve_ring(handle: AddressHandle, modules: Sequence[SimObj]) -> None`
  - 说明：包装 `handle.resolve_ring()`，将一组模块注册为待自动连续编号的环成员。

- `aa_unconnected_mac_register(handle: AddressHandle) -> None`
  - 说明：声明一个不需要地址的 MAC。常用于兼容原生 MMS 中“未连接接口”的计数。

## 进程注册表（`miaosuan.mms.process_registry`）

**用途**：在运行期维护进程（Process）级别的属性索引，实现基于属性匹配的进程发现、查询以及跨模块的关联验证。

### 枚举与常量

- `AttrType(IntEnum)`：定义属性值类型。
  - `INVALID (0)`：无效占位，不可用于实际注册。
  - `OBJ_ID (1)`：对象 ID（整数），用于节点或模块对象引用。
  - `STRING (2)`：字符串。
  - `NUMBER (3)`：浮点数；传入整数会被转换为 `float`。
  - `STRING_LIST (4)`：字符串序列。
  - `PROCESS_HANDLE (5)`：进程句柄或自定义对象。
  - `POINTER (6)`：任意指针或句柄对象，不做类型检查。
  - `INT32 (7)`：32 位整型，超出范围会抛 `ValueError`。
  - `INT64 (8)`：64 位整型。
  - `BOOL (9)`：布尔。
- 预定义属性名：
  - `ATTR_NODE_OBJ_ID = "node objid"`
  - `ATTR_MODULE_OBJ_ID = "module objid"`
  - `ATTR_PROCESS_HANDLE = "process handle"`
  - `ATTR_PROCESS_NAME = "process name"`

### 数据结构

- `ProcessAttribute(name: str, attr_type: AttrType, value: Any)`
  - 说明：属性描述对象，构造时会将 `attr_type` 规范为 `AttrType`，并根据类型自动完成值校验与复制（例如字符串列表会深复制）。
- `ProcessRecord`
  - 说明：线程安全的属性集合实现，内部维护按名称排序的 `ProcessAttribute` 列表。对外方法包括：
    - `get_attributes() -> Sequence[ProcessAttribute]`：返回属性的不可变副本。
    - `add_or_update_attribute(attribute: ProcessAttribute) -> None`：按名称插入或更新属性，若类型冲突则抛出 `TypeError`。
    - `find_attribute(name: str) -> Optional[ProcessAttribute]`：查找属性并返回副本。
    - `verify(spec_attrs: Sequence[ProcessAttribute], neighbor_module_id: int) -> bool`：用于进程发现逻辑，检查属性是否满足筛选条件并验证拓扑连接。
- `ProcessHandle = ProcessRecord`
  - 说明：为了与原生 MMS API 对齐，所有对外函数均以 `ProcessHandle` 为别名返回。

### 注册与属性维护函数

- `pr_register(node_obj_id: int, module_obj_id: int, proc_handle: Any, proc_name: str) -> ProcessHandle`
  - 说明：创建新的 `ProcessRecord`，并自动写入节点 ID、模块 ID、进程句柄及名称四个基础属性。注册完成后会同时加入全局列表与按节点分组的哈希表。
  - 异常：`proc_name` 为空字符串时抛 `ValueError`。

- `pr_discover(neighbor_obj_id: int, *attrs_to_match: ProcessAttribute) -> List[ProcessHandle]`
  - 说明：根据提供的属性列表筛选符合条件的进程。若传入 `neighbor_obj_id`，会限定搜索同一父节点的进程；否则可以通过属性中的 `node objid` 锁定节点。
  - 要求：至少提供一个 `ProcessAttribute`，否则抛 `ValueError`。属性类型必须与枚举匹配，否则在规范化阶段抛出 `TypeError`。

- `pr_attr_set(handle: ProcessHandle, name: str, attr_type: AttrType, value: Any) -> None`
  - 说明：为指定进程设置或更新单个属性。若属性已存在但类型不同，会抛 `TypeError`。`handle` 为空时抛 `ValueError`。

- `pr_attr_set_multi(handle: ProcessHandle, *attributes: ProcessAttribute) -> None`
  - 说明：批量设置多个属性，内部直接调用 `ProcessRecord.add_or_update_attribute()`。

- `pr_attr_get(handle: ProcessHandle, name: str) -> Any`
  - 说明：读取属性值并返回副本（例如字符串列表会返回新的列表），若属性不存在抛 `KeyError`。`handle` 为空抛 `ValueError`。

### 调试与运维

- `print_registry(node_id_hint: int = 0) -> None`
  - 说明：将当前注册表内容打印到标准输出。`node_id_hint` 为非零时只打印指定节点；否则会遍历全部节点并输出总计数量。输出格式包含每个属性的名称、类型和数值，便于问题排查。

### 常见使用流程示例

1. 调用 `pr_register()` 为模块启动时生成的进程建立注册记录。
2. 使用 `pr_attr_set()` 或 `pr_attr_set_multi()` 写入业务自定义的匹配属性（如服务类型、端口等）。
3. 通过 `pr_discover()` 结合邻居模块或节点 ID 进行查找，返回符合条件的一个或多个 `ProcessHandle`。
4. 若需要持久化或调试，可调用 `print_registry()` 查看注册表快照。
