# Simulink 模型资源浏览器
![[模型资源浏览器.png]]
上图是 **Simulink 的 Model Explorer（模型资源浏览器）** 层级树。图中各节点含义如下：

**1) Simulink Root**  
这是整个 Model Explorer 的总根节点，展开后会显示当前会话里打开的 MATLAB 工作区、Simulink 模型和 Stateflow 图。它的作用是提供一个统一入口，方便在多个模型、图和工作区之间切换。适用范围是**整个当前 MATLAB/Simulink 会话**。典型场景是你同时打开了多个模型，需要快速定位某个模型的配置、工作区变量或外部数据。([MathWorks](https://www.mathworks.com/help/simulink/slref/modelexplorer.html "Model Explorer - View, modify, and add elements of Simulink models, Stateflow charts, and workspace variables - MATLAB"))

**2) Base Workspace（位于 Simulink Root 下）**  
这是 **MATLAB base workspace** 本身。文档说明它对所有打开的模型和图都可见，也就是全局共享工作区。适用范围是**会话级全局变量**，更适合临时试验、快速参数调试、旧项目兼容。MathWorks 也明确建议：base workspace 更适合临时存放数据；如果要永久保存，应该存成 MAT 文件或脚本。([MathWorks](https://www.mathworks.com/help/simulink/slref/modelexplorer.html "Model Explorer - View, modify, and add elements of Simulink models, Stateflow charts, and workspace variables - MATLAB"))

**3) DCU_ASW（模型节点）**  
这是你当前打开的那个 Simulink 模型本体。展开它后，会看到这个模型所拥有的资源节点，例如 **Configurations、Model Workspace、External Data**。适用范围就是**当前这个模型**。典型场景是你要看这个模型自己的配置参数、局部变量、外部数据源，而不是整个 MATLAB 会话的全局内容。([MathWorks](https://www.mathworks.com/help/simulink/slref/modelexplorer.html "Model Explorer - View, modify, and add elements of Simulink models, Stateflow charts, and workspace variables - MATLAB"))

**4) Configurations**  
这个节点显示的是该模型的**配置集（configuration sets）** 和 **配置引用（configuration references）**。配置集本质上是一组模型参数值，例如求解器类型、仿真开始/停止时间等。默认情况下，配置集属于单个模型；如果要给多个模型共用，通常需要把配置做成 **freestanding configuration set**，放到 base workspace 或数据字典里，再让多个模型引用它。适用范围是**模型仿真、代码生成、实时执行相关的配置参数管理**。典型场景是你希望多模型使用同一套求解器、代码生成目标或诊断设置，避免每个模型单独维护。([MathWorks](https://www.mathworks.com/help/simulink/slref/modelexplorer.html "Model Explorer - View, modify, and add elements of Simulink models, Stateflow charts, and workspace variables - MATLAB"))

**5) Model Workspace**  
这是**模型私有工作区**。文档定义它只在当前模型作用域内可见，变量带有独立命名空间；同名变量可以在不同模型工作区里同时存在且取不同值。它适合存放模型局部数据，比如参数、`Simulink.Parameter` 对象、模型参数、模型参数化所需的数值变量，以及模型参数。适用范围是**单个模型**，尤其适合被引用模型、组件化模型、需要提高可移植性和数据封装性的场景。MathWorks 也明确建议：把局部设计数据放进 model workspace，可以提升模型可移植性，并把数据所有权绑定到模型文件。([MathWorks](https://www.mathworks.com/help/simulink/slref/simulink-concepts-models.html "Simulink Models - MATLAB & Simulink"))

**6) External Data**  
这是该模型的**外部数据源入口**。文档说明这里会列出模型的外部数据源，包括**base workspace（如果已启用访问）**、数据字典，以及 MAT 文件等外部数据关联。适用范围是**模型级的数据源管理**，不是单纯某一个变量本身。典型场景是你希望把设计数据从模型文件中拆出去，放到 MAT 文件或数据字典里，以便持久保存、共享、版本管理和更好的工程化维护。([MathWorks](https://www.mathworks.com/help/simulink/slref/modelexplorer.html "Model Explorer - View, modify, and add elements of Simulink models, Stateflow charts, and workspace variables - MATLAB"))

**7) External Data 下的 Base Workspace**  
这不是“另一个”独立的工作区，而是**同一个 MATLAB base workspace 作为该模型的外部数据源**被展示出来。只有在模型允许访问 base workspace 时，它才会出现在 External Data 下面；如果模型链接了数据字典，也可以通过相关选项允许字典访问 base workspace。适用范围是**模型对全局工作区的访问入口**。典型场景是旧模型仍依赖 base workspace 中的变量，或者你在迁移到数据字典之前，先保持与原有脚本/工作流兼容。这个理解属于根据文档做出的归纳：一个是“全局工作区节点”，一个是“作为外部数据源的工作区入口”。([MathWorks](https://www.mathworks.com/help/simulink/slref/modelexplorer.html "Model Explorer - View, modify, and add elements of Simulink models, Stateflow charts, and workspace variables - MATLAB"))

**实际使用上可以这样理解：**  
`Base Workspace` 适合“全局、临时、快速试验”；`Model Workspace` 适合“局部、封装、可移植”；`External Data` 适合“外部持久化数据源（MAT 文件/数据字典）”；`Configurations` 适合“模型参数配置管理”。这是 MathWorks 推荐的数据组织方向：尽量把数据作用域收窄，把共享数据放到更明确的外部载体里。([MathWorks](https://www.mathworks.com/help/simulink/ug/determine-where-to-store-data-for-simulink-models.html "Determine Where to Store Variables and Objects for Simulink Models - MATLAB & Simulink"))