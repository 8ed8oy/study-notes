# 常用操作综合示例

> 来源：《动手学深度学习》（[d2l-zh](https://github.com/d2l-ai/d2l-zh)）第 5 章 `example`。
> 代码与公式来自教材（CC BY-SA 4.0 / MIT），文字说明与超参实验是我自己整理和补充的。

本Notebook展示了PyTorch中最核心和高频使用的功能，涵盖模型定义、参数操作、模型保存与加载以及设备管理。

## 1. 导入必要的库

```python
import torch
from torch import nn
from torch.nn import functional as F
import os

print(f"PyTorch version: {torch.__version__}")
```

## 2. 模型设计

展示如何使用 `nn.Sequential` 快速构建模型，以及如何继承 `nn.Module` 自定义层。

```python
# 2.1 使用 Sequential 快速构建
net_seq = nn.Sequential(
    nn.Linear(20, 256),
    nn.ReLU(),
    nn.Linear(256, 10)
)

X = torch.rand(2, 20)
print("Sequential Model Output Shape:", net_seq(X).shape)

# 2.2 自定义网络层
class MyLinear(nn.Module):
    def __init__(self, in_units, units):
        super().__init__()
        # 使用 nn.Parameter 注册参数，使其可被优化器识别
        self.weight = nn.Parameter(torch.randn(in_units, units))
        self.bias = nn.Parameter(torch.zeros(units))
    
    def forward(self, X):
        linear = torch.matmul(X, self.weight) + self.bias
        return F.relu(linear)

class MyModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.layer1 = MyLinear(20, 64)
        self.layer2 = nn.Linear(64, 10)
    
    def forward(self, X):
        return self.layer2(self.layer1(X))

net_custom = MyModel()
print("Custom Model Output Shape:", net_custom(X).shape)
```

## 3. 参数管理

展示如何访问、遍历和初始化参数。

```python
# 3.1 访问特定参数
print("\n--- Parameter Access ---")
print("First layer weight shape:", net_seq[0].weight.shape)

# 3.2 遍历所有参数 (name, param)
print("\n--- Named Parameters ---")
for name, param in net_custom.named_parameters():
    print(f"{name}: {param.shape}")

# 3.3 参数初始化
print("\n--- Parameter Initialization ---")
def init_weights(m):
    if isinstance(m, nn.Linear):
        # 对线性层应用初始化
        nn.init.xavier_uniform_(m.weight)
        print(f"Initialized Linear layer: {m}")

net_seq.apply(init_weights)
```

## 4. 模型持久化

展示如何保存和加载模型参数（推荐使用 state_dict）。

```python
# 4.1 保存模型参数
print("\n--- Saving Model ---")
save_path = 'model_params.pt'
torch.save(net_seq.state_dict(), save_path)
print(f"Model params saved to {save_path}")

# 4.2 加载模型参数
# 注意：必须先创建一个结构相同的模型实例
clone = nn.Sequential(
    nn.Linear(20, 256),
    nn.ReLU(),
    nn.Linear(256, 10)
)

# 加载参数
clone.load_state_dict(torch.load(save_path))
print("Model params loaded successfully.")

# 验证两个模型的第一层权重是否一致
is_same = torch.equal(net_seq[0].weight, clone[0].weight)
print(f"Weights are identical: {is_same}")

# 清理文件
os.remove(save_path)
```

## 5. 计算优化 (GPU 加速)

展示如何管理设备并将张量/模型迁移到 GPU。

```python
print("\n--- Device Management ---")

def try_gpu(i=0):
    """如果GPU可用则返回gpu(i)，否则返回cpu()"""
    if torch.cuda.device_count() >= i + 1:
        return torch.device(f'cuda:{i}')
    return torch.device('cpu')

device = try_gpu()
print(f"Using device: {device}")

# 5.1 张量迁移
x = torch.tensor([1, 2, 3])
x_gpu = x.to(device)
print(f"Tensor on device: {x_gpu.device}")

# 5.2 模型迁移
net_gpu = nn.Sequential(nn.Linear(3, 1))
net_gpu.to(device)
print(f"Model parameters on device: {net_gpu[0].weight.device}")

# 5.3 在同一设备上计算
# 输入数据也必须在同一设备上
X_input = torch.randn(2, 3, device=device)
output = net_gpu(X_input)
print(f"Output is on: {output.device}")
```
