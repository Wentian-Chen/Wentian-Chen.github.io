# PyTorch 的深入理解

## 引言：PyTorch 是什么

PyTorch 是当前深度学习研究和工业落地中使用最广泛的框架之一。它由 Meta（原 Facebook）的人工智能团队开发和维护，核心设计理念是**动态计算图**（Dynamic Computation Graph）：网络在每次前向传播时都实时构建计算图，这意味着你可以用普通的 Python 控制流（if、for 循环）来编写模型结构，调试起来和在写普通 Python 程序一样自然。

与静态图框架（如 TensorFlow 1.x 时代的 Graph）相比，动态图的优势在于：

- **灵活**：模型结构可以在运行时动态变化，适合变长输入、循环结构等复杂场景。
- **易调试**：报错信息直接对应 Python 代码，配合 `print` 和断点就能定位问题。
- **接近 NumPy**：张量操作与 NumPy 几乎一致，从科学计算迁移到深度学习几乎没有门槛。

正因如此，PyTorch 在学术科研领域占据了绝对主流，大部分论文的开源代码（包括具身智能和 VLA 方向的大量工作）都是用 PyTorch 写的。对我这样在实验室里做研究的学生来说，PyTorch 几乎是一个默认选项。理解它的核心机制，能让你读代码、改代码、复现论文时都事半功倍。

下面我按一条主线来讲：**张量 → 自动求导 → 模型 → 训练 → 部署**，这也是从零构建一个深度学习项目必经的路径。

## 张量：Tensor

张量（Tensor）是 PyTorch 的基本数据结构，可以理解为支持 GPU 加速、自动求导的"NumPy 数组"。如果你熟悉 NumPy，学起来会非常快。

### 创建张量

```python
import torch

a = torch.tensor([1, 2, 3])          # 从列表创建
z = torch.zeros(2, 3)                # 全 0 张量
o = torch.ones(2, 3)                 # 全 1 张量
r = torch.randn(2, 3)                # 标准正态分布随机张量
ar = torch.arange(0, 10, 2)          # [0, 2, 4, 6, 8]
e = torch.eye(3)                     # 单位矩阵
```

### 形状操作

```python
x = torch.randn(2, 3)

y = x.reshape(3, 2)      # 重新调整形状（视图优先）
y2 = x.view(3, 2)        # 与 reshape 类似，但要求内存连续
y3 = x.t()               # 转置，得到形状 (3, 2)
y4 = x.transpose(0, 1)   # 指定维度转置

a = torch.randn(3, 1)
b = a.squeeze()          # 去掉长度为 1 的维度 -> (3,)
c = b.unsqueeze(0)       # 在第 0 维插入长度 1 的维度 -> (1, 3)
```

!!! warning "注意"
    `view` 要求张量在内存中连续，`transpose` 后的张量通常不再连续，此时用 `view` 会报错，`reshape` 则更宽容。不熟悉内存布局的话，直接用 `reshape` 更省心。

### 广播机制

当两个张量形状不一致时，PyTorch 会尝试**广播**（broadcasting）：把维度对齐，长度为 1 的维度自动扩展。例如形状为 `(3, 1)` 的张量可以和 `(1, 4)` 的张量直接相加，得到 `(3, 4)`。

```python
x = torch.randn(3, 1)
y = torch.randn(1, 4)
z = x + y        # 广播后结果形状为 (3, 4)
```

广播机制能让代码简洁很多，但前提是维度规则满足要求：从尾部对齐，要么长度相同，要么其中一个为 1，要么一方缺失该维度。不满足时不要硬写，先用 `unsqueeze` 显式扩展。

### device 与数据类型

张量有两大关键属性：**device**（所在设备）和 **dtype**（数据类型）。

```python
cpu_t = torch.tensor([1.0, 2.0])
gpu_t = cpu_t.to("cuda")          # 移到 GPU
back = gpu_t.to("cpu")            # 移回 CPU

half = gpu_t.half()               # 转 FP16，推理加速常用
f32 = cpu_t.float()               # 转 FP32
long_t = torch.tensor([0, 1]).long()   # 类别标签常用 int64
```

!!! tip "技巧"
    `.to()` 也可以接受 device 对象：`t.to(device)`。在代码里先定义好 `device = torch.device("cuda" if torch.cuda.is_available() else "cpu")`，再统一把模型和数据 `to(device)`，是避免设备不一致报错的好习惯。

## 自动求导：autograd

自动求导是深度学习的发动机。PyTorch 通过**动态计算图**记录每次张量运算，并在反向传播时自动计算梯度，省去了手动推导梯度的繁琐过程。

### 核心概念

- 创建张量时设置 `requires_grad=True`，之后所有基于它的运算都会被记录进计算图。
- 调用 `loss.backward()` 时，PyTorch 从 `loss` 出发沿计算图反向传播，把梯度累积到每个叶子张量的 `.grad` 属性中。
- `.grad` 保存的是梯度，`.zero_grad()` 用于清零梯度，防止多次反向传播时梯度累积。

### 最小示例

```python
import torch

x = torch.tensor(2.0, requires_grad=True)
y = x ** 2          # y = 4
y.backward()        # 反向传播
print(x.grad)       # tensor(4.)，即 dy/dx = 2x = 4
```

一个稍复杂一点的例子，验证梯度确实是解析值：

```python
import torch

x = torch.tensor(3.0, requires_grad=True)
z = (x - 1) ** 3
z.backward()
print(x.grad)       # tensor(12.)，即 3(x-1)^2 = 12
```

### 梯度清零

```python
x = torch.tensor(2.0, requires_grad=True)

for _ in range(2):
    y = x ** 2
    y.backward()
    print(x.grad)   # 第一次输出 tensor(4.)，第二次输出 tensor(8.)，因为梯度累积了
    x.grad.zero_()  # 手动清零后再看
```

!!! note "说明"
    PyTorch 的 `backward()` 默认是**累积梯度**而不是覆盖。在训练循环里，`optimizer.zero_grad()` 必须在每次 `backward()` 之前调用，否则梯度会不断叠加，模型行为完全错误。

## 构建模型：nn.Module

### 基本写法

PyTorch 中所有模型都继承自 `nn.Module`，需要实现 `__init__`（定义层）和 `forward`（定义前向计算逻辑）：

```python
import torch
import torch.nn as nn

class MLP(nn.Module):
    def __init__(self, in_dim, hidden_dim, out_dim):
        super().__init__()
        self.fc1 = nn.Linear(in_dim, hidden_dim)
        self.relu = nn.ReLU()
        self.fc2 = nn.Linear(hidden_dim, out_dim)

    def forward(self, x):
        x = self.fc1(x)
        x = self.relu(x)
        x = self.fc2(x)
        return x

model = MLP(4, 8, 2)
out = model(torch.randn(3, 4))   # 输入 (3, 4)，输出 (3, 2)
```

`forward` 里可以用任意 Python 控制流，这是动态图的精髓。注意**不要直接调用 `model.forward(x)`**，请使用 `model(x)`，这样 PyTorch 能正确管理钩子、模式切换等内部逻辑。

### nn.Sequential

如果模型就是简单的层序列，`nn.Sequential` 能让代码更紧凑：

```python
import torch.nn as nn

model = nn.Sequential(
    nn.Linear(4, 8),
    nn.ReLU(),
    nn.Linear(8, 2),
)
```

### 常用层一览

- `nn.Linear(in, out)`：全连接层。
- `nn.Conv2d(in, out, kernel_size)`：二维卷积层，是 CNN 的核心。
- `nn.ReLU()`：激活函数，默认选择。
- `nn.Dropout(p)`：随机丢弃一部分神经元，防止过拟合，只在训练时生效。
- `nn.BatchNorm1d / BatchNorm2d`：批归一化，加速收敛、稳定训练，在卷积网络中使用非常普遍。
- `nn.Embedding`：词嵌入 / 离散 token 嵌入，Transformer 系列模型的基础。

### 参数管理

模型定义好之后，可以用 API 遍历和管理参数：

```python
for name, param in model.named_parameters():
    print(name, param.shape)

total_params = sum(p.numel() for p in model.parameters())
print(f"总参数量: {total_params}")
```

`named_parameters()` 返回参数名和参数张量的配对，`parameters()` 只返回参数张量。按名称访问参数（如 `model.fc1.weight`）在初始化、冻结、检查点恢复时都很常用。

## 损失与优化

### 常用损失函数

| 损失 | 适用场景 |
| --- | --- |
| `nn.MSELoss` | 回归任务，预测连续数值（如坐标、速度） |
| `nn.L1Loss` | 回归任务，对异常值更鲁棒 |
| `nn.CrossEntropyLoss` | 多分类任务，内部已包含 Softmax，输入是原始 logits |
| `nn.BCEWithLogitsLoss` | 二分类或多标签分类 |
| `nn.CosineSimilarity` | 度量学习、对比学习，衡量向量方向一致性 |

!!! warning "注意"
    `nn.CrossEntropyLoss` 的输入是**未经 Softmax 的 logits**，标签是整数索引（不是 one-hot）。如果你先手动 Softmax 再传进去，训练时会出问题。

### 优化器

```python
import torch.optim as optim

optimizer = optim.SGD(model.parameters(), lr=0.01, momentum=0.9)
optimizer = optim.Adam(model.parameters(), lr=1e-3)
optimizer = optim.AdamW(model.parameters(), lr=1e-3, weight_decay=0.01)
```

怎么选？

- **SGD + momentum**：经典选择，收敛稳定，调好学习率后效果很好，常用于小模型。
- **Adam**：自适应学习率，开箱即用，绝大多数深度学习任务（尤其是 Transformer 和具身智能模型）的默认选择。
- **AdamW**：Adam 的修正版，正确实现了权重衰减，预训练大模型的事实标准。

### 学习率调度

`scheduler` 可以在训练过程中动态调整学习率，典型做法是余弦退火（CosineAnnealingLR）或按里程碑阶梯下降（StepLR）：

```python
scheduler = optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=epochs)
# 每个 epoch 结束后调用
scheduler.step()
```

一句话总结：**优化器负责更新方向，调度器负责更新节奏**，两者配合能让训练更快更稳。

## 数据管道

### Dataset 与 DataLoader

`torch.utils.data.Dataset` 负责定义"数据长什么样"，你需要继承它并实现两个方法：`__len__`（数据集大小）和 `__getitem__`（按索引取一条样本）。

```python
import torch
from torch.utils.data import Dataset, DataLoader

class MyDataset(Dataset):
    def __init__(self, xs, ys):
        self.xs = torch.tensor(xs, dtype=torch.float32)
        self.ys = torch.tensor(ys, dtype=torch.float32)

    def __len__(self):
        return len(self.xs)

    def __getitem__(self, idx):
        return self.xs[idx], self.ys[idx]

dataset = MyDataset([[0, 0], [1, 1], [2, 2]], [0, 2, 4])
loader = DataLoader(dataset, batch_size=2, shuffle=True, num_workers=0)
```

`DataLoader` 负责把样本组织成批次：

- `batch_size`：每批样本数。
- `shuffle`：是否打乱顺序，训练集一般开、验证集一般关。
- `num_workers`：数据加载的并行进程数，`0` 表示主进程加载（调试方便），Linux 下可以调到 4、8 等。

在数据加载环节，`torchvision.transforms` 提供了丰富的预处理工具（Resize、ToTensor、Normalize、RandomCrop、RandomHorizontalFlip 等），一句话：**transforms 在数据进入模型前完成形状归一化和数据增强**。如果你的输入不是图像而是向量，直接用 `torch.tensor` 转换即可。

## 训练循环范式

### 最小完整示例

下面的代码是一个完整的"线性回归 + SGD"训练循环，覆盖了训练的标准骨架：

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset

# 1. 数据
x = torch.randn(100, 1)
y = 2.0 * x + 1.0
loader = DataLoader(TensorDataset(x, y), batch_size=16, shuffle=True)

# 2. 模型、损失、优化器
model = nn.Linear(1, 1)
criterion = nn.MSELoss()
optimizer = optim.SGD(model.parameters(), lr=0.01)

# 3. 训练
epochs = 20
for epoch in range(epochs):
    model.train()                     # 进入训练模式（启用 Dropout/BatchNorm）
    total_loss = 0.0
    for xb, yb in loader:
        optimizer.zero_grad()         # 清空上一步梯度
        pred = model(xb)              # 前向
        loss = criterion(pred, yb)    # 计算损失
        loss.backward()               # 反向传播
        optimizer.step()              # 更新参数
        total_loss += loss.item()
    print(f"epoch {epoch + 1}: loss = {total_loss / len(loader):.4f}")
```

每一步的含义：

- `optimizer.zero_grad()`：清零梯度（防止累积）。
- `model(xb)`：前向传播。
- `loss.backward()`：反向传播，计算梯度。
- `optimizer.step()`：用梯度更新参数。

### train / eval 模式与 torch.no_grad()

```python
model.eval()                          # 验证/推理模式（关闭 Dropout、冻结 BatchNorm 统计量）
with torch.no_grad():                 # 不构建计算图，省内存、省时间
    val_pred = model(val_x)
```

- `model.train()` 和 `model.eval()` 切换的是**模型行为**（Dropout、BatchNorm 是否生效）。
- `torch.no_grad()` 关闭的是**梯度记录**，推理时一定要用，否则显存会被计算图撑爆。

!!! note "说明"
    一句话记住：**训练用 `model.train()` + backward，评估用 `model.eval()` + `no_grad()`**。很多新手报错"显存不足"或者"验证结果异常"，往往就是这两处的模式没切对。

## 模型保存与加载

### 保存和加载权重

最推荐的方式是保存 `state_dict()`（只含参数，不含模型结构）：

```python
torch.save(model.state_dict(), "model.pt")

model = nn.Linear(1, 1)               # 先重建模型结构
model.load_state_dict(torch.load("model.pt"))
model.eval()
```

### 保存检查点

训练中断后想续训，需要把优化器状态、epoch 一起保存：

```python
checkpoint = {
    "epoch": epoch,
    "model": model.state_dict(),
    "optimizer": optimizer.state_dict(),
    "scheduler": scheduler.state_dict(),
}
torch.save(checkpoint, "checkpoint.pt")

# 恢复
ckpt = torch.load("checkpoint.pt")
model.load_state_dict(ckpt["model"])
optimizer.load_state_dict(ckpt["optimizer"])
```

!!! warning "注意"
    `torch.load` 默认加载到保存时的设备，跨设备（GPU 训练、CPU 加载）时会报错，可以加 `map_location="cpu"` 或 `map_location="cuda"` 指定。

### 迁移学习与微调

用预训练权重做微调是省时省力的标准做法。做法通常是：替换分类头，然后冻结大部分参数，只训练头部或后几层。

```python
import torch
import torch.nn as nn

model = pretrained_model()
model.fc = nn.Linear(model.fc.in_features, num_classes)

for name, param in model.named_parameters():
    param.requires_grad = False      # 先冻结全部

for param in model.fc.parameters():
    param.requires_grad = True       # 只放开分类头

optimizer = torch.optim.Adam(filter(lambda p: p.requires_grad, model.parameters()), lr=1e-3)
```

`requires_grad=False` 的参数不会被优化器更新，也不会被计算梯度，显存占用更小、训练更快。这也是具身智能项目里微调视觉骨干网络的常用姿势。

## 分布式与加速简介

### 环境检查

```python
import torch

print(torch.cuda.is_available())     # GPU 是否可用
print(torch.cuda.device_count())     # GPU 数量
print(torch.cuda.get_device_name(0)) # 设备名
```

### AMP 混合精度

AMP（Automatic Mixed Precision）用 FP16 做计算、FP32 做参数存储，训练速度提升明显且精度损失极小。用 `torch.cuda.amp` 只需要改动训练循环的几行，值得尽早掌握。

### DDP 分布式训练

当单卡放不下模型、或者想多卡加速时，使用 `torch.distributed` 的 DistributedDataParallel（DDP），它通过多进程在每张卡上维护一个模型副本，同步梯度进行训练。一句话：**单卡能跑的代码，加一个 DDP 包装就能多卡并行**，是做大模型训练绕不开的组件。

## 常见问题

### CUDA out of memory

- 调小 `batch_size` 是最直接的手段。
- 训练时确认没有多余的张量被留在计算图上（例如把 batch 内的中间结果拼起来保存）。
- 推理时务必用 `torch.no_grad()`。
- 使用 AMP 混合精度可以把显存占用降低近一半。

### 设备不一致报错（Expected all tensors to be on the same device）

数据和模型不在同一个 device 上。解决方案：定义统一的 `device`，训练循环里把每个 batch 都 `.to(device)`，模型也 `.to(device)`。

```python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = model.to(device)
xb, yb = xb.to(device), yb.to(device)
```

### backward 后梯度累积

如果发现 loss 或参数更新行为异常，先检查是不是忘记调用 `optimizer.zero_grad()`。这个错误非常隐蔽，loss 会以"阶梯式"波动，但不会崩溃，排查时要格外留意。

## 结语

PyTorch 的核心其实并不复杂：**张量负责数据，autograd 负责梯度，nn.Module 负责组织模型，DataLoader 负责供数，训练循环把它们串起来**。把这条主线理解透了，再复杂的论文代码，读起来也只是在这套骨架上增加细节而已。具身智能方向的很多工作（包括 VLA 模型）都构建在 PyTorch 生态之上，先把地基打牢，后面的路会顺畅很多。

## 参考链接

- PyTorch 官方教程：[pytorch.org/tutorials](https://pytorch.org/tutorials)
- PyTorch 官方文档：[pytorch.org/docs](https://pytorch.org/docs/stable/index.html)
- torchvision transforms 文档：[pytorch.org/vision/stable/transforms.html](https://pytorch.org/vision/stable/transforms.html)
- PyTorch 官方 GitHub：[github.com/pytorch/pytorch](https://github.com/pytorch/pytorch)
