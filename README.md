这是一个**在 PYNQ 平台（Zynq FPGA + Python）上快速原型化 CNN** 的开源工程（还在 Alpha 阶段）。它已经把常见深度学习框架（Caffe/Theano+Lasagne）的依赖、示例模型（LeNet、CIFAR-10）以及配套的硬件工程（Vivado HLS + Vivado Block Design）都准备好了，目标是让你尽快在 PYNQ 板子上跑起来。

工程支持两种“入口”：

1. **软件侧**：用作者准备好的 **PYNQ SD 卡镜像**，上电后直接在 Jupyter 上运行 demo（LeNet、CIFAR-10），模型可以用 **Caffe 或 Theano 语法** 的预训练权重。
2. **硬件侧**：如果你要**扩展/更改 CNN 模型的硬件实现**，就去 `hw/README.md` 按流程**先跑 Vivado HLS 生成 IP，再跑 Vivado 脚本生成顶层工程**。

---

# 仓库结构与你要做的事

## 1) PYNQ SD 卡镜像（最省事的跑通路径）

**它在说什么**
作者已经打包好一个 SD 卡镜像，里面预装了 Caffe、Theano（含 Lasagne）以及运行这些 demo 所需的依赖。建议直接用它，别自己手动装依赖（README 里明确写了“NOT RECOMMENDED”，因为问题多）。

**你要做什么（最小步骤）**

1. 准备一张 **≥16GB 的 microSD 卡**。
2. 用 Balena Etcher / Rufus 把镜像写进 SD 卡（镜像在 README 给的百度盘/Google Drive 链接）。
3. 把 SD 插到 **PYNQ 板**（如 PYNQ-Z1/Z2），用 USB/电源上电。
4. 把你的电脑网卡设到和板子同一网段（例如给电脑设一个 `192.168.2.x` 的静态地址，掩码 `255.255.255.0`）。
5. 通过浏览器访问 **`192.168.2.99`**（这是作者设置的 **Jupyter** 的静态 IP）。按镜像说明进入 Jupyter（密码/端口若镜像另有说明，以镜像为准）。
6. 在 Jupyter 里找到 demo（LeNet / CIFAR-10 “quick” 版本）并运行。通常会有 notebook 或脚本一步步演示推理/性能。

**为什么这么做**

* 省去了在 ARM Linux 上编译 Caffe/Theano 的痛点；
* 一上来就能跑通 demo，适合验证板卡/网络/驱动是否 OK。

**常见坑**

* **IP 冲突/连不上**：确认电脑网卡跟板子在同一网段（`192.168.2.*`），且没有别的设备占用 `192.168.2.99`。
* **镜像写卡失败**：换读卡器/换 SD 卡/校验镜像完整性（有校验值就对比）。
* **浏览器访问不到**：先 `ping 192.168.2.99`，不通就查网线/USB 网络共享、电脑是否开启了 VPN/安全软件拦截等。
* **手动装依赖**：除非你确有把握，否则不建议；真要装，按仓库 `MANUAL_INSTALL.md` 来（README 明说不推荐）。

---

## 2) Vivado 工程（要“实现更多 CNN 模型”或改硬件时）

**它在说什么**
“如何实现更多 CNN 模型？”——去看 **`hw/README.md`**。也就是硬件部分：先用 **Vivado HLS** 把你的 CNN 代码综合成 IP，再用 **Vivado** 的 Tcl 脚本自动生成完整的 **Block Design** 工程，把 HLS 的 CNN 模块接进系统里。

> 这一点和你之前发给我的另一段“Regenerating Vivado HLS and Vivado Projects”是一致的：**先 HLS，后 Vivado；没装 PYNQ 板卡文件要先拷进去。**

**你要做什么（典型命令流）**

1. **先跑 HLS**（以 CIFAR-10 为例）：

   ```bash
   cd PYNQ-Classification/hw/hls/CIFAR_10_wrapper
   vivado_hls -f run_cifar_10.tcl
   ```

   这会生成/更新 CNN 的 HLS IP。
2. **安装板卡文件**（如果 Vivado 里找不到 PYNQ/兼容板）：
   把 `tools/PYNQ_BOARD_FILES/arty-z7-20/` 复制到你 Vivado 安装目录下的
   `.../Vivado/<version>/data/boards/board_files/`，重启 Vivado。
3. **生成顶层 Vivado 工程**：

   ```bash
   cd PYNQ-Classification/hw/base_project
   vivado -source pynq_arch.tcl
   ```

   这会新建工程、导入 HLS IP、搭好 Block Design（Zynq PS、AXI、你的 CNN IP 等），并可能继续跑综合/实现（视脚本而定）。

**为什么这么做**

* 上层 Vivado 工程**依赖**你刚导出的 HLS IP；如果 IP 没更新，Vivado 里的设计不会变化；
* 模块化：CNN 的硬件实现与系统集成分离，便于替换/复用。

**常见坑**

* 没有 `vivado_hls`/`vivado` 命令：先 `source settings64.sh`（或用官方 Tcl Shell）。
* Vivado 里选不到板卡：板卡文件没放对目录或版本、放了但没重启。
* HLS 路径/头文件错误：修 `run_cifar_10.tcl` 的包含路径，避免中文/空格路径。
* 时序/资源红了：先降低频率目标、减少过度 pipeline/unroll，或改用更大器件。

---

## 3) Demo：LeNet & CIFAR-10（Caffe “quick” 版）

**它在说什么**
README 说明**第 3 步**会教你“如何下载并运行 LeNet 和 CIFAR-10（Caffe quick 版）”的 demo（通常在 Jupyter 里运行）。

> 你贴的这段只列了 1 和 2；第 3 步的具体命令一般在仓库的 notebooks 或 `demos/` 目录里，或视频里演示。

**你要做什么（一般套路）**

* 打开 ①里准备好的 Jupyter；
* 进入作者提供的 notebook/script；
* 选择 **Caffe** 或 **Theano+Lasagne** 的模型格式（这两种“语法”指的是网络定义/权重保存方式不同）；
* 运行单元格，一步步完成加载权重 → 在 PYNQ 上执行推理 → 查看正确率/速度等。

**为什么支持两种语法**

* 这方便拿你已有的 Caffe 或 Theano 项目做迁移与对比；
* 对同一 CNN，可能只需换加载器/权重文件，就能在同一平台上测试。

**常见坑**

* 权重/网络描述文件不匹配（prototxt ↔ caffemodel，或 Theano/Lasagne 的参数名不一致）；
* 缺少 Python 依赖：若用了镜像，通常已装好；如果你自己搭环境，按 `MANUAL_INSTALL.md` 补齐。

---

# 视频教程

README 给了一个 YouTube 教学视频链接（工程综述/上手演示）。建议先看一遍，心里有全局图，再照上面的“软件跑通路径”实际操作。

---

# 一页纸速查（TL;DR）

* **想快速跑通**：

  1. 下好 **SD 镜像** → 写卡 → 上电；
  2. 电脑设到同网段 → 打开浏览器访问 **`192.168.2.99`** → 进 **Jupyter**；
  3. 找到 **LeNet/CIFAR-10 demo** → 运行。

* **想改硬件/实现更多模型**：

  1. `vivado_hls -f run_cifar_10.tcl`（先生成 HLS IP）；
  2. 复制 **arty-z7-20** 板卡文件到 Vivado 的 `board_files` 并重启；
  3. `vivado -source pynq_arch.tcl`（生成顶层 Vivado 工程，查看/综合/实现）。

* **不建议**：自己从零装 Caffe/Theano（问题多）；真要装看 `MANUAL_INSTALL.md`。

## PROJECT NAME: 
PYNQ Classification - Python on Zynq FPGA for Convolutional Neural Networks (Alpha Release)

## BRIEF DESCRIPTION:
This repository presents a fast prototyping framework, which is an Open Source framework designed to enable fast deployment of embedded Convolutional Neural Network (CNN) applications on PYNQ platforms.

## CITATION:
If you make use of this code, please acknowledge us by citing [our article](https://spiral.imperial.ac.uk/handle/10044/1/57937):

	@inproceedings{Wang_FCCM18,
        author={E. Wang and J. J. Davis and P. Y. K. Cheung},
        booktitle={IEEE Symposium on Field-programmable Custom Computing Machines (FCCM)},
        title={{A PYNQ-based Framework for Rapid CNN Prototyping}},
        year={2018}
    }

## INSTRUCTIONS TO BUILD AND TEST THE PROJECT:

### Repository Organisation

The project demo accepts pre-trained CNN models in either Caffe or Theano syntax, hence the step 1 and 2 introduces how to install Caffe and Theano (with Lasagne) on PYNQ. Step 3 explains how to download and run the demos for LeNet and CIFAR-10 (Caffe "quick" version) models.

For a quick overview on the project please watch [my video tutorial](https://youtu.be/DoA8hKBltV4).

### 1. PYNQ SD Card Image

We have prepared a SD card image with pre-installed Caffe and Theano dependencies. A SD card with at least 16GB is needed. The static IP for the PYNQ Jupyter Notebook is 192.168.2.99

[Download Link (on Baidu Drive)](https://pan.baidu.com/s/1c2EmMvY)

[Download Link (on Google Drive)](https://drive.google.com/open?id=1A9MrN_zzbHYiIHJvnNQYFc3sXqWZBb6o)

If you wish to setup Caffe and Theano dependencies on your own, please see [MANUAL_INSTAL.md](MANUAL_INSTALL.md) for instructions. (NOT RECOMMENDED since multiple issues have been reported)

### 2. Vivado Project Setup - How to implement more CNN models?

Please go to hw/README.md for guide on regenerating the Vivado and Vivado HLS projects.

### References
    
### 1. BNN-PYNQ

If you find BNN-PYNQ useful, please cite the <a href="https://arxiv.org/abs/1612.07119" target="_blank">FINN paper</a>:

    @inproceedings{finn,
    author = {Umuroglu, Yaman and Fraser, Nicholas J. and Gambardella, Giulio and Blott, Michaela and Leong, Philip and Jahre, Magnus and Vissers, Kees},
    title = {FINN: A Framework for Fast, Scalable Binarized Neural Network Inference},
    booktitle = {Proceedings of the 2017 ACM/SIGDA International Symposium on Field-Programmable Gate Arrays},
    series = {FPGA '17},
    year = {2017},
    pages = {65--74},
    publisher = {ACM}
    }

### 2. Caffe

Caffe is released under the [BSD 2-Clause license](https://github.com/BVLC/caffe/blob/master/LICENSE).
The BAIR/BVLC reference models are released for unrestricted use.

Please cite Caffe in your publications if it helps your research:

    @article{jia2014caffe,
      Author = {Jia, Yangqing and Shelhamer, Evan and Donahue, Jeff and Karayev, Sergey and Long, Jonathan and Girshick, Ross and Guadarrama, Sergio and Darrell, Trevor},
      Journal = {arXiv preprint arXiv:1408.5093},
      Title = {Caffe: Convolutional Architecture for Fast Feature Embedding},
      Year = {2014}
    }
    
### 3. Theano

    @ARTICLE{2016arXiv160502688short,
       author = {{Theano Development Team}},
        title = "{Theano: A {Python} framework for fast computation of mathematical expressions}",
      journal = {arXiv e-prints},
       volume = {abs/1605.02688},
     primaryClass = "cs.SC",
     keywords = {Computer Science - Symbolic Computation, Computer Science - Learning, Computer Science - Mathematical Software},
         year = 2016,
        month = may,
          url = {http://arxiv.org/abs/1605.02688},
    }

### 4. Lasagne 

    @misc{lasagne,
      author       = {Sander Dieleman and
                      Jan Schlüter and
                      Colin Raffel and
                      Eben Olson and
                      Søren Kaae Sønderby and
                      Daniel Nouri and
                      others},
      title        = {Lasagne: First release.},
      month        = aug,
      year         = 2015,
      doi          = {10.5281/zenodo.27878},
      url          = {http://dx.doi.org/10.5281/zenodo.27878}
    }
