# YOLO 的全流程指南

## 引言：YOLO 是什么

YOLO（You Only Look Once）是一种经典的**单阶段目标检测**算法。与两阶段检测器（如 Faster R-CNN）先提候选区域、再逐区域分类不同，YOLO 把目标检测当作一个**端到端的回归问题**：把输入图像划分成网格，每个网格负责预测若干个边界框及其类别，网络"只看一眼"就能直接输出所有目标的位置和类别，因此推理速度极快，非常适合实时场景。

从 2016 年 Joseph Redmon 提出 YOLO 开始，这个家族经历了多次迭代：YOLOv5 让训练和部署的工程化体验大幅提升，YOLOv8 在精度与速度之间取得了很好的平衡并全面重构了代码结构，再到 YOLO11 继续在边缘设备上做优化。如今提到 YOLO，绝大多数情况下指的是 **Ultralytics** 维护的开源生态，它把数据集管理、训练、验证、导出、部署整合成了一套非常顺手的工具链。

为什么 YOLO 如此常用？总结下来大概有这几点：

- **速度快**：单阶段设计天然适合实时推理，在边缘设备上也能跑得动。
- **效果好**：在公开数据集（如 COCO）上的精度已经不输两阶段方法，且工程优化空间大。
- **生态成熟**：Ultralytics 提供统一的 CLI 和 Python API，数据格式统一，从训练到部署的链路非常顺滑。
- **上手成本低**：数据集格式简单，官方文档齐全，社区资料丰富。

在具身智能的日常研究中，YOLO 也常常被当作视觉感知的基础组件，例如给机械臂提供目标物体的检测框。下面我就从环境安装开始，带你完整走一遍 YOLO 的使用流程。

本文的整体路线如下：

```text
环境安装 -> 数据集准备 -> 训练 -> 验证评估 -> 预测推理 -> 模型导出 -> 常见问题
```

## 环境安装

### 安装 ultralytics

首先需要 Python 3.8 及以上的环境。推荐使用虚拟环境（如 `venv` 或 `conda`）隔离项目依赖，避免污染系统环境。

```bash
pip install ultralytics
```

安装完成后，在终端验证一下：

```bash
yolo --version
```

如果能正常输出版本号，说明安装成功。也可以在 Python 里验证：

```python
import ultralytics
ultralytics.checks()
```

### CPU / GPU 的考虑

- **CPU**：可以训练和推理小模型（如 YOLOv8n），但速度很慢，通常只用于验证流程是否跑通。
- **GPU**：强烈建议使用 NVIDIA GPU 进行训练。Ultralytics 会自动检测可用的 CUDA 设备，无需额外配置。

GPU 环境的关键是 **PyTorch 与 CUDA 版本要匹配**。`pip install ultralytics` 默认会安装对应的 PyTorch 版本，但如果你需要自定义 CUDA 版本，可以先去 PyTorch 官网按自己的 CUDA 环境选择安装命令，再单独安装 ultralytics。判断 CUDA 是否可用：

```python
import torch
print(torch.cuda.is_available())
```

如果输出 `True`，说明 GPU 环境正常。

!!! note "说明"
    关于 CUDA 版本匹配的细节，不同机器差异很大，最简单可靠的做法是：先查清本机 NVIDIA 驱动支持的 CUDA 版本，然后去 PyTorch 官方安装页选择对应组合。

### 安装验证

装好后可以跑一次最简单的训练命令，确认整条链路（PyTorch、GPU、数据加载）都是通的：

```bash
yolo train data=coco8.yaml model=yolov8n.pt epochs=1 imgsz=640
```

`coco8.yaml` 是 ultralytics 自带的微型示例数据集（只有 8 张图），用来验证环境非常合适。如果这条命令能顺利跑完，说明环境没有问题，可以开始准备自己的数据了。

!!! tip "技巧"
    新环境第一次跑训练时，官方会自动下载 `coco8.yaml` 对应的数据和 `yolov8n.pt` 权重。如果网络受限，可以提前用 `yolo download` 或手动下载权重文件放到指定目录，避免训练中断。

## 数据集准备

### 标注工具

训练目标检测模型，第一步是准备标注数据。我平时用得比较多的是 **LabelImg**，它是一个开源的图形化标注工具，支持直接导出 YOLO 格式的标注文件。

```bash
pip install labelimg
labelimg
```

在 LabelImg 里打开图片目录，用矩形框框出目标，选择类别标签，保存时选择 YOLO 格式即可。除了 LabelImg，也可以使用 LabelStudio、Roboflow 等工具，标注流程大同小异。

### YOLO 标注格式

YOLO 格式的标注是**一个图片对应一个 txt 文件**，txt 中每一行代表一个目标，格式如下：

```text
class_id x_center y_center width height
```

其中：

- `class_id`：类别编号，从 0 开始，对应 `dataset.yaml` 中 `names` 列表的下标。
- `x_center y_center`：目标中心点的横、纵坐标，**归一化到 [0, 1]**（除以图片宽/高）。
- `width height`：边界框的宽和高，同样归一化。

例如一张图片里有一个人和一辆车（人 = 0，车 = 1），标注文件内容可能是：

```text
0 0.5000 0.4500 0.1200 0.3000
1 0.7000 0.6000 0.1500 0.2500
```

!!! warning "注意"
    坐标必须是归一化的小数。如果直接填像素值，训练时边界框会严重错位，损失函数也无法收敛。

### 目录结构

Ultralytics 要求的目录结构非常固定，图片和标签分开存放，并且训练集、验证集分目录：

```text
dataset/
├── images/
│   ├── train/
│   │   ├── 001.jpg
│   │   └── 002.jpg
│   └── val/
│       ├── 003.jpg
│       └── 004.jpg
└── labels/
    ├── train/
    │   ├── 001.txt
    │   └── 002.txt
    └── val/
        ├── 003.txt
        └── 004.txt
```

关键点：**图片和标签的文件名必须一一对应**（扩展名不同没关系），并且每个标签文件中的类别编号必须在 `names` 范围内。

### dataset.yaml

数据集写好后，需要一个 yaml 文件描述数据集，Ultralytics 称之为 dataset.yaml：

```yaml
path: /path/to/dataset   # 数据集根目录
train: images/train      # 训练图片目录（相对 path）
val: images/val          # 验证图片目录（相对 path）

names:
  0: person
  1: car
```

字段含义：

- `path`：数据集根目录，可以是绝对路径或相对路径。
- `train` / `val`：训练集和验证集图片目录，相对 `path` 给出。
- `names`：类别名列表，`class_id` 与这里的下标一一对应。

!!! tip "技巧"
    建议在写完 yaml 后先用 `yolo train data=dataset.yaml model=yolov8n.pt epochs=1` 快速跑一个 epoch，确认数据能正常加载、图像和标签能正确配对，再正式训练。

## 训练

### 训练命令

训练是使用频率最高的操作，一条命令即可开始：

```bash
yolo train data=dataset.yaml model=yolov8n.pt epochs=100 imgsz=640 batch=16 device=0
```

- `data`：指向 dataset.yaml。
- `model`：预训练权重或模型配置文件。传 `.pt` 文件表示从该权重继续训练，也可以传 `yolov8n.yaml` 从零训练。
- 其余参数含义见下面的表格。

### 常用参数含义

| 参数 | 含义 |
| --- | --- |
| `epochs` | 训练的轮数，过少欠拟合，过多容易过拟合 |
| `imgsz` | 输入图像尺寸，默认 640，越大越慢但可能提升小目标精度 |
| `batch` | 每批样本数，受显存限制，显存不足就调小 |
| `device` | 训练设备，`0` 表示第一张 GPU，`cpu` 表示用 CPU |
| `workers` | 数据加载的进程数，越大加载越快，但占用内存 |
| `patience` | 早停耐心值，验证指标连续这么多轮不提升就提前停止 |
| `lr0` | 初始学习率，默认 0.01 |
| `lrf` | 最终学习率与初始学习率的比值，用于学习率衰减 |
| `optimizer` | 优化器，可选 SGD、Adam、AdamW 等 |
| `cache` | 是否缓存数据到内存（`True`）或磁盘（`disk`），可显著加速小数据集训练 |
| `pretrained` | 是否使用预训练权重，默认是 |
| `project` / `name` | 训练输出的保存路径，默认在 `runs/detect/` 下 |
| `resume` | 断点续训，传 `True` 或上次训练的输出目录路径 |
| `plots` | 是否保存训练过程的可视化图表，默认开启 |

!!! note "说明"
    `model=yolov8n.pt` 中的字母代表模型尺寸：n（nano）、s（small）、m（medium）、l（large）、x（xlarge）。精度依次提升，速度依次下降，小数据集从 n 或 s 开始比较合适。

### 从零训练与断点续训

大多数情况下我们使用 `model=yolov8n.pt` 的方式，它表示**在 COCO 预训练权重的基础上继续训练**，即迁移学习，收敛快、效果稳定。如果希望完全从随机初始化开始，可以传入模型配置文件：

```bash
yolo train data=dataset.yaml model=yolov8n.yaml epochs=100
```

两种方式的区别一句话就能说清：**`.pt` 是"接着练"，`.yaml` 是"从头练"**。小数据集几乎总是应该用预训练权重。

训练中途意外中断后，可以用 `resume` 接着跑，不需要重新开始：

```bash
yolo train resume=True
# 或指定目录
yolo train resume=runs/detect/train/
```

!!! tip "技巧"
    `resume` 会自动读取上次训练记录下的 `args.yaml`、epoch 和优化器状态，所以你只需要写 `yolo train resume=True` 一行，其余参数不用重复给出。

### 训练输出

训练过程中的输出保存在 `runs/detect/train/` 目录下（如果重复训练，会自动生成 `train2`、`train3` 等目录）。主要包括：

- `results.png`：训练曲线图，包含 loss、mAP、精确率、召回率随 epoch 的变化，是判断训练是否正常的最直观工具。
- `weights/best.pt`：验证集指标最好的权重，**上线部署和评估请用这个**。
- `weights/last.pt`：最后一个 epoch 的权重，用于断点续训。
- `args.yaml`：本次训练的全部参数，方便复现。
- `confusion_matrix.png` 等评估可视化文件。

!!! warning "注意"
    训练中后期看到 loss 略有波动是正常的，判断收敛不要只看 loss，要结合验证集的 mAP 和 `results.png` 综合判断。

## 验证与评估

### 验证命令

训练完成后，用验证集评估模型：

```bash
yolo val model=runs/detect/train/weights/best.pt data=dataset.yaml
```

Ultralytics 会在 `runs/detect/val/` 目录下输出混淆矩阵、PR 曲线以及各类别的详细指标。

### 核心指标

- **mAP50**：IoU 阈值取 0.5 时的平均精度均值（mAP），是目标检测最常用的指标之一。
- **mAP50-95**：IoU 阈值从 0.5 到 0.95（步长 0.05）共 10 个阈值下 mAP 的平均值，对边界框的精确度要求更严格，常用于论文汇报。
- **Precision（精确率）**：预测为正的样本中，实际为正的比例，即"检测出来的目标里有多少是对的"。
- **Recall（召回率）**：实际为正的样本中，被预测出来的比例，即"真正的目标被找回了多少"。
- **混淆矩阵**：展示每个类别之间互相误检的情况，能直观看出哪两类容易混淆。

!!! tip "技巧"
    如果你的场景对"宁缺毋滥"更敏感（比如检错比漏检更致命），可以适当调高置信度阈值；反之则调低，让召回率更高。具体可以通过 `yolo predict` 时的 `conf` 参数控制。

## 预测与推理

### 命令行方式

```bash
yolo predict model=runs/detect/train/weights/best.pt source=image.jpg
```

`source` 可以是单张图片、图片目录、视频文件，甚至摄像头（`source=0`）。预测结果默认保存在 `runs/detect/predict/` 目录下。

常用参数：

- `conf`：置信度阈值，默认 0.25，低于该值的检测框被过滤。
- `iou`：NMS 的 IoU 阈值，默认 0.7。
- `save`：是否保存结果图片，默认 `True`。
- `show`：直接显示结果窗口，需要图形界面。

### Python API 方式

在实际项目（尤其是具身智能的研究代码）里，更常用的是 Python API：

```python
from ultralytics import YOLO

model = YOLO("runs/detect/train/weights/best.pt")

results = model("image.jpg")

for r in results:
    boxes = r.boxes
    print(boxes.cls)        # 类别编号
    print(boxes.conf)       # 置信度
    print(boxes.xyxy)       # 边界框坐标 (x1, y1, x2, y2)
```

对单张图片，也可以直接取坐标：

```python
from ultralytics import YOLO

model = YOLO("best.pt")
result = model("image.jpg")[0]
xyxy = result.boxes.xyxy.cpu().numpy()
```

!!! note "说明"
    结果坐标默认在 GPU 上（如果可用），取数值前建议先 `.cpu()` 再转 numpy，避免设备不一致导致的报错。

## 模型导出

训练好的 PyTorch 权重往往还需要导出成其他格式，才能部署到特定平台。Ultralytics 一行命令即可完成导出：

```bash
yolo export model=runs/detect/train/weights/best.pt format=onnx
```

导出 ONNX 后，就可以配合 ONNX Runtime、TensorRT 等在推理服务器或嵌入式设备上部署。

其他常用格式：

```bash
yolo export model=best.pt format=tensorrt  # NVIDIA GPU 上的高性能推理
yolo export model=best.pt format=engine    # TensorRT 引擎，等价物
yolo export model=best.pt format=coreml    # Apple 设备
yolo export model=best.pt format=tflite    # 移动端 / 边缘设备
```

!!! tip "技巧"
    TensorRT 导出时通常会进行 INT8 或 FP16 量化，推理速度提升明显，但首次导出需要较长时间，并且建议在目标设备上执行。

## 常见问题

### 显存不足（CUDA out of memory）

这是最常遇到的报错之一。解决方法按优先级排列：

- 调小 `batch`（如从 16 调到 8 或 4）。
- 调小 `imgsz`（如从 640 调到 480）。
- 换更小的模型（如从 `yolov8m` 换到 `yolov8n`）。
- 设置 `cache=False`，减少数据缓存占用的显存。

### 训练不收敛

如果 loss 一直不下降或验证指标长期不动，可以从这几个方向排查：

- 检查数据集：标签是否归一化、类别编号是否对应、图片是否清晰。
- 学习率：初始学习率过大或过小都会影响收敛，可在 `lr0` 附近做几个小实验。
- 数据量：数据太少时模型很难收敛，优先考虑数据增强或迁移学习。
- 先跑一个 epoch 验证数据加载无误，再调参。

### 中文路径问题

路径中含中文（如 `数据集/...`）时，Windows 下容易出现编码或路径解析异常。**强烈建议数据集路径全部使用英文**，这是最省事的做法。

### 数据增强

Ultralytics 在训练时默认开启了一系列数据增强（Mosaic、随机翻转、色彩抖动等），小数据集尤其受益。如果发现训练集精度高、验证集低，可以适当降低增强强度（如调低 `hsv_h`、`hsv_s` 等参数），但一般建议优先保证数据量本身。

## 结语

从环境安装、数据准备，到训练、评估、推理和部署，YOLO 的全流程其实非常顺畅。得益于 Ultralytics 统一的设计，绝大部分环节都可以用一行命令完成，这也让它成为目标检测入门的首选。如果你打算把 YOLO 用在具身智能的项目里，我建议：先用小模型把流程跑通，再逐步在数据、增强和模型规模上做迭代。

## 参考链接

- Ultralytics 官方文档：[docs.ultralytics.com](https://docs.ultralytics.com)
- Ultralytics GitHub 仓库：[github.com/ultralytics/ultralytics](https://github.com/ultralytics/ultralytics)
- LabelImg 标注工具：[github.com/HumanSignal/labelImg](https://github.com/HumanSignal/labelImg)
- PyTorch 官方安装页：[pytorch.org/get-started/locally](https://pytorch.org/get-started/locally)
