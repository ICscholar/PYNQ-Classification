
### Guide for Regenerating Vivado HLS and Vivado Projects

## Script Design Interface

In this project, the CNN topology is packaged as a Vivado HLS block design, which is then included in the Vivado project. Hence, we should regenerate Vivado HLS project first, before regenerating the overall Vivado project.

- Regenerate Vivado HLS Project (Take CIFAR_10 Model as Example)

Open Vivado HLS command tool. cd to PYNQ-Classification/hw/hls/CIFAR_10_wrapper directory. Run the following command.

`vivado_hls -f run_cifar_10.tcl`


- Copy PYNQ Board Files to Your Vivado Installation

The default Vivado installation does not contain the PYNQ board files. Here we use the Arty-z7-20 board files, sinice it is also applicable to PYNQ board configuration. Just copy the PYNQ-Classification/tools/PYNQ_BOARD_FILES/arty-z7-20/ folder under (vivado_installation)/data/boards/board_files/ directory. Vivado should automatically detect new board files at restart.

- Regenerate Vivado Project

Open Vivado command tool. cd to PYNQ-Classification/hw/base_project directory. Run the following command.

`vivado -source pynq_arch.tcl`

## Graphical Design Interface (Legacy)

The graphical design interface for our CIFAR-10 example, as demonstrated in our YouTube video, can be regenerated with our makefile.

`make compile_graphical`

The makefile will run HLS for each component block, followed by regenerating the SKETCHPAD project. The SKETCHPAD project is a Vivado Block Design project, where users can design CNN models by chaining layer blocks.

整套流程的核心逻辑是：**先用 Vivado HLS 生成/更新 CNN 的 HLS IP（打包成一个模块），再用 Vivado 把这个模块放进 Block Design，最后生成完整工程**。之所以顺序不能反，是因为 Vivado 工程要引用 HLS 导出的 IP。

---

# 一、整体概念先搞清

* **Vivado HLS**：把 C/C++/SystemC 写的算法（这里是 CNN）综合成可在 FPGA 上用的硬件 IP（RTL），并可导出为 Vivado 可用的 IP。
* **Vivado（Block Design）**：在 SoC（如 Zynq）上把各种 IP（ARM 处理器、AXI 总线、你的 CNN 模块等）“连线拼装”成一个系统，再生成比特流/工程。
* **Board files（板卡文件）**：Vivado 识别开发板型号所需的描述文件（器件、引脚、电源、IO 等）。默认不带 PYNQ 板卡文件，需要手动拷进去，这样建工程时就能直接选对应板卡。
* **本项目结构**：CNN 拓扑打包成一个 **Vivado HLS block**，再被更上层的 **Vivado 工程**引用。因此**必须先跑 HLS**，再生成 Vivado 工程。

---

# 二、准备工作（强烈建议）

1. **安装版本**
   统一用同一版 Vivado/Vitis HLS（文档写 Vivado HLS；从 2020.1 起叫 Vitis HLS，但命令行兼容）。尽量跟项目作者使用的版本一致，避免 IP 兼容问题。
2. **环境变量**

   * Linux：`source /tools/Xilinx/Vivado/<version>/settings64.sh`
   * Windows：用 Vivado HLS / Vivado 自带的 *Tcl Shell / Command Prompt* 打开。
3. **代码路径**
   把仓库（例如 `PYNQ-Classification/`）放在**不含空格和中文**的路径下，避免 Tcl/工具链路径解析出错。
4. **权限**
   往 Vivado 安装目录里拷板卡文件需要管理员/写权限。

---

# 三、步骤 1：重新生成 Vivado HLS 工程（以 CIFAR-10 为例）

**做什么**：运行项目里提供的 Tcl 脚本，让 HLS 把 CNN 代码综合成 IP。

**为什么**：后续 Vivado 工程会引用这个 HLS IP；IP 变了，上层工程才知道拿新版本。

**怎么做**：

```bash
# 打开 Vivado HLS 命令行工具（或 Vitis HLS）
cd PYNQ-Classification/hw/hls/CIFAR_10_wrapper
vivado_hls -f run_cifar_10.tcl
```

**运行时会发生什么（通常）**：

* 脚本会创建/打开一个 HLS 工程，设置顶层函数/时钟约束等；
* 进行 C 综合（C Synthesis），可能还会做仿真（csim/cosim，视脚本而定）；
* **导出 IP** 到某个目录（通常是 `solution*/impl/ip` 或脚本指定路径）。

**完成后你应能看到**：

* HLS 工程目录（例如 `CIFAR_10_wrapper/solution1/`）；
* 生成的 RTL（Verilog/VHDL）与一个可被 Vivado 识别的 IP 包；
* 控制台无报错，最后有 “Finished/Exporting IP…” 之类日志。

**常见坑**：

* 没有 source Vivado 环境脚本 → 命令找不到或库缺失；
* C/C++ 版本、头文件路径错误 → 调整 `run_cifar_10.tcl` 里的包含路径；
* 时序/资源约束过紧 → 先保证能导出 IP，再微调时钟周期/pragma。

---

# 四、步骤 2：把 PYNQ 板卡文件复制到 Vivado 安装目录

**做什么**：把 `arty-z7-20` 这套板卡文件复制到 Vivado 的 board\_files 目录。

**为什么**：Vivado 默认没带 PYNQ 板（或兼容板）的描述文件；放进去后，Vivado 建工程时就能选这块板（与 PYNQ 配置兼容）。

**怎么做**：

1. 找到你本机 Vivado 安装目录（示例）

   * Linux：`/tools/Xilinx/Vivado/<version>/data/boards/board_files/`
   * Windows：`C:\Xilinx\Vivado\<version>\data\boards\board_files\`
2. 复制项目里的板卡文件目录：

   ```
   PYNQ-Classification/tools/PYNQ_BOARD_FILES/arty-z7-20/
   ```

   到上面“board\_files”目录下（保持文件夹结构整洁）。
3. **重启 Vivado**，让它重新扫描板卡。

**完成后你应能看到**：

* 在 Vivado 新建工程时，**Boards** 列表里能看到 *arty-z7-20*（或相应名字）。

**常见坑**：

* 复制到错版本/错路径（比如放进 Vitis 安装目录）→ Vivado 里看不到板；
* 权限不足 → 以管理员/ sudo 执行拷贝。

---

# 五、步骤 3：重新生成 Vivado 工程（Block Design）

**做什么**：运行项目的顶层 Tcl 脚本，自动创建 Vivado 工程、导入 HLS IP、搭好 Block Design、生成系统。

**为什么**：省去手工搭 Block Design 的繁琐步骤，确保和作者视频/示例一致。

**怎么做**：

```bash
# 打开 Vivado 命令行工具（Tcl Shell）
cd PYNQ-Classification/hw/base_project
vivado -source pynq_arch.tcl
```

**运行时会发生什么（通常）**：

* 创建一个新的 Vivado 工程（工程名与路径由脚本定）；
* 选择目标器件/板卡（用你刚放进去的 board files）；
* 把 HLS 导出的 IP 加进来；
* 搭建 Block Design（Zynq PS、AXI interconnect、CNN IP、内存/中断连接等）；
* 可能会 `validate_bd_design`、`generate_block_design`、`synth_design`，再到实现/比特流（视脚本）。

**完成后你应能看到**：

* 一个可在 Vivado GUI 打开的完整工程；
* **IP Integrator** 里能看到 CNN 的 HLS 模块已经连好；
* 可能脚本会自动综合（Synthesis）甚至实现/生成 bitstream（具体看脚本）。

**常见坑**：

* HLS IP 没生成或路径不对 → `pynq_arch.tcl` 找不到 IP；回到步骤 1；
* 板卡文件没装好 → 工程创建失败或器件不匹配；
* 约束/时序报错 → 先只跑到综合，通过后再做实现。

---

# 六、图形化设计界面（Legacy）

**这是什么**：项目里曾经有个“图形化”工作流，**用 makefile 一键**跑每个组件的 HLS，然后生成一个名为 **SKETCHPAD** 的 Vivado Block Design 工程。你可以在 Vivado GUI 里像搭积木一样把各个卷积/池化等层级 IP 串起来设计 CNN。

**怎么用**：

```bash
make compile_graphical
```

**会做什么**：

* 逐个组件跑 HLS（每个 layer 都会生成一个小 IP）；
* 生成 **SKETCHPAD** 这个 Vivado 工程（Block Design 项目）。

**何时用它**：

* 想用**可视化拖拽**搭网络（教学/演示/快速试验）；
* 但它是 **Legacy（旧方式）**，新流程更推荐直接用上面的 HLS→顶层 Vivado 脚本一把梭。

---

# 七、最小可操作清单（可以直接照做）

1. 打开 **Vivado HLS 终端**

   ```bash
   cd PYNQ-Classification/hw/hls/CIFAR_10_wrapper
   vivado_hls -f run_cifar_10.tcl
   ```
2. 安装 **板卡文件**

   * 把 `PYNQ-Classification/tools/PYNQ_BOARD_FILES/arty-z7-20/`
     复制到 `…/Vivado/<ver>/data/boards/board_files/`
   * 重启 Vivado
3. 打开 **Vivado 终端**

   ```bash
   cd PYNQ-Classification/hw/base_project
   vivado -source pynq_arch.tcl
   ```

> 运行无报错后，你就有了一个包含 CNN HLS 模块的完整 Vivado 工程；接下来可以在 GUI 里查看 Block Design、综合/实现、导出 bitstream，或继续按项目文档部署到 PYNQ。

---

# 八、常见问题速查

* **找不到 `vivado_hls` / `vivado` 命令**：没有 `source settings64.sh`（Linux）或没用官方命令行（Windows）。
* **板卡选不到/看不到**：板卡目录复制错位置/版本，或没重启 Vivado。
* **Tcl 脚本失败路径相关**：避免含空格/中文路径；检查脚本里相对路径是否与仓库目录结构一致。
* **时序/资源不满足**：先放宽 HLS 时钟周期、去掉过激的 pipeline/unroll，或换更大的器件；确认与板卡（如 Zynq-7020 / Z7-20）匹配。
* **库/头文件缺失**：根据 `run_cifar_10.tcl` 的 include/define 提示修正。

---

# 九、名词对照与目录速记

* `hw/hls/CIFAR_10_wrapper/`：HLS 工程脚本与代码目录（入口：`run_cifar_10.tcl`）。
* `hw/base_project/`：顶层 Vivado 工程生成脚本目录（入口：`pynq_arch.tcl`）。
* `tools/PYNQ_BOARD_FILES/arty-z7-20/`：板卡文件源目录，要复制到 Vivado 的 `board_files/` 下。
* **SKETCHPAD**：图形化 Block Design 的示例工程名（通过 `make compile_graphical` 生成）。

---


