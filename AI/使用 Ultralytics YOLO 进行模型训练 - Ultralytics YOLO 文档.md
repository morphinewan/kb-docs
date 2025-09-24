---
title: "训练"
url: "https\://docs.ultralytics.com/zh/modes/train/\#train-settings"
description: "通过关于设置、数据增强和硬件利用的全面说明，了解如何使用 YOLO11 有效地训练目标检测模型。"
author: "Ultralytics"
site_name: "\<null\>"
language: "zh"
image: "https\://github.com/ultralytics/docs/releases/download/0/ultralytics-yolov8-ecosystem-integrations.avif"
tags: "Ultralytics, YOLO11, model training, deep learning, object detection, GPU training, dataset augmentation, hyperparameter tuning, model performance, apple silicon training"
generated_at: "2025-09-01T08:39:57.519Z"
---

[](https://github.com/ultralytics/ultralytics/tree/main/docs/en/modes/train.md "编辑此页面")[](javascript:void\(0\) "以 Markdown 格式复制页面")

使用 Ultralytics YOLO 进行模型训练
==========================

![Ultralytics YOLO 生态系统和集成](https://github.com/ultralytics/docs/releases/download/0/ultralytics-yolov8-ecosystem-integrations.avif)

简介
--

训练[深度学习](https://www.ultralytics.com/glossary/deep-learning-dl)模型包括向其输入数据并调整其参数，使其能够做出准确的预测。Ultralytics YOLO11 中的训练模式专为有效训练目标检测模型而设计，充分利用了现代硬件功能。本指南旨在涵盖您使用 YOLO11 强大的功能集开始训练您自己的模型所需的所有详细信息。

  
  
**观看：** 如何在 Google Colab 上训练 YOLO 模型以用于您的自定义数据集。

为什么选择 Ultralytics YOLO 进行训练？
----------------------------

以下是选择 YOLO11 训练模式的一些令人信服的理由：

*   **效率：** 充分利用您的硬件，无论您是在单 GPU 设置上还是在多个 GPU 上进行扩展。
*   **多功能性：** 除了 COCO、VOC 和 ImageNet 等现成的数据集外，还可以在自定义数据集上进行训练。
*   **用户友好：** 简单而强大的 CLI 和 python 接口，可提供直接的训练体验。
*   **超参数灵活性：** 范围广泛的可自定义超参数，可用于微调模型性能。

### 训练模式的主要功能

以下是 YOLO11 训练模式的一些值得注意的功能：

*   **自动数据集下载：** 首次使用时会自动下载 COCO、VOC 和 ImageNet 等标准数据集。
*   **多 GPU 支持：** 在多个 GPU 上无缝扩展您的训练工作，以加快流程。
*   **超参数配置：** 可以选择通过 YAML 配置文件或 CLI 参数修改超参数。
*   **可视化和监控：** 实时跟踪训练指标并可视化学习过程，以获得更好的见解。

提示

*   YOLO11 数据集（如 COCO、VOC、ImageNet 和许多其他数据集）在首次使用时会自动下载，即 `yolo train data=coco.yaml`

使用示例
----

在 COCO8 数据集上训练 YOLO11n 100 [个 epoch](https://www.ultralytics.com/glossary/epoch) ，图像大小为 640。可以使用以下命令指定训练设备 `device` 参数。如果未传递任何参数，则 GPU `device=0` 如果可用，将使用，否则 `device='cpu'` 将被使用。有关训练参数的完整列表，请参见下面的“参数”部分。

Windows多进程处理错误

在 Windows 上，您可能会收到 `RuntimeError` 以脚本形式启动训练时。添加一个 `if __name__ == "__main__":` 在您的训练代码之前添加代码块以解决它。

单 GPU 和 CPU 训练示例

设备是自动确定的。如果有 GPU 可用，则将使用 GPU（默认 CUDA 设备 0），否则将在 CPU 上开始训练。

`from ultralytics import YOLO # Load a model model = YOLO("yolo11n.yaml")  # build a new model from YAML model = YOLO("yolo11n.pt")  # load a pretrained model (recommended for training) model = YOLO("yolo11n.yaml").load("yolo11n.pt")  # build from YAML and transfer weights # Train the model results = model.train(data="coco8.yaml", epochs=100, imgsz=640)`

`# Build a new model from YAML and start training from scratch yolo detect train data=coco8.yaml model=yolo11n.yaml epochs=100 imgsz=640 # Start training from a pretrained *.pt model yolo detect train data=coco8.yaml model=yolo11n.pt epochs=100 imgsz=640 # Build a new model from YAML, transfer pretrained weights to it and start training yolo detect train data=coco8.yaml model=yolo11n.yaml pretrained=yolo11n.pt epochs=100 imgsz=640`

### 多 GPU 训练

通过将训练负载分配到多个 GPU 上，多 GPU 训练可以更有效地利用可用的硬件资源。此功能可通过 python API 和命令行界面使用。要启用多 GPU 训练，请指定您希望使用的 GPU 设备 ID。

多 GPU 训练示例

要使用 2 个 GPU（CUDA 设备 0 和 1）进行训练，请使用以下命令。根据需要扩展到其他 GPU。

`from ultralytics import YOLO # Load a model model = YOLO("yolo11n.pt")  # load a pretrained model (recommended for training) # Train the model with 2 GPUs results = model.train(data="coco8.yaml", epochs=100, imgsz=640, device=[0, 1]) # Train the model with the two most idle GPUs results = model.train(data="coco8.yaml", epochs=100, imgsz=640, device=[-1, -1])`

`# Start training from a pretrained *.pt model using GPUs 0 and 1 yolo detect train data=coco8.yaml model=yolo11n.pt epochs=100 imgsz=640 device=0,1 # Use the two most idle GPUs yolo detect train data=coco8.yaml model=yolo11n.pt epochs=100 imgsz=640 device=-1,-1`

### 空闲 GPU 训练

空闲 GPU 训练支持自动选择多 GPU 系统中利用率最低的 GPU，从而优化资源使用，而无需手动选择 GPU。此功能根据利用率指标和 VRAM 可用性识别可用的 GPU。

空闲 GPU 训练示例

要自动选择和使用最空闲的 GPU 进行训练，请使用 `-1` device 参数。这在共享计算环境或具有多个用户的服务器中特别有用。

`from ultralytics import YOLO # Load a model model = YOLO("yolov8n.pt")  # load a pretrained model (recommended for training) # Train using the single most idle GPU results = model.train(data="coco8.yaml", epochs=100, imgsz=640, device=-1) # Train using the two most idle GPUs results = model.train(data="coco8.yaml", epochs=100, imgsz=640, device=[-1, -1])`

`# Start training using the single most idle GPU yolo detect train data=coco8.yaml model=yolov8n.pt epochs=100 imgsz=640 device=-1 # Start training using the two most idle GPUs yolo detect train data=coco8.yaml model=yolov8n.pt epochs=100 imgsz=640 device=-1,-1`

自动选择算法优先考虑具有以下特征的 GPU：

1.  更低的电流利用率
2.  更高的可用内存（可用 VRAM）。
3.  更低的温度和功耗

此功能在共享计算环境或跨不同模型运行多个训练作业时尤其有价值。它可以自动适应不断变化的系统条件，确保最佳的资源分配，而无需手动干预。

### Apple Silicon MPS 训练

通过 Ultralytics YOLO 模型中集成的对 Apple 芯片的支持，现在可以在利用强大的 Metal Performance Shaders (MPS) 框架的设备上训练您的模型。MPS 提供了一种在 Apple 的定制芯片上执行计算和图像处理任务的高性能方法。

为了在 Apple 芯片上启用训练，您应该在启动训练过程时将 'mps' 指定为您的设备。以下是如何在 python 中以及通过命令行执行此操作的示例：

MPS 训练示例

`from ultralytics import YOLO # Load a model model = YOLO("yolo11n.pt")  # load a pretrained model (recommended for training) # Train the model with MPS results = model.train(data="coco8.yaml", epochs=100, imgsz=640, device="mps")`

`# Start training from a pretrained *.pt model using MPS yolo detect train data=coco8.yaml model=yolo11n.pt epochs=100 imgsz=640 device=mps`

在利用 Apple 芯片的计算能力的同时，这可以更有效地处理训练任务。如需更详细的指导和高级配置选项，请参阅 [PyTorch MPS 文档](https://docs.pytorch.org/docs/stable/notes/mps.html)。

### 恢复中断的训练

从先前保存的状态恢复训练是使用深度学习模型时的一项关键功能。这在各种情况下都非常有用，例如当训练过程意外中断时，或者当您希望使用新数据或更多 epoch 继续训练模型时。

当训练恢复时，Ultralytics YOLO 会从上次保存的模型加载权重，还会恢复优化器状态、[学习率](https://www.ultralytics.com/glossary/learning-rate)调度器和 epoch 编号。这使您可以从中断的地方无缝地继续训练过程。

您可以通过设置以下参数，在 Ultralytics YOLO 中轻松恢复训练 `resume` 参数为 `True` 在调用 `train` 方法时，并指定包含部分训练模型权重的 `.pt` 文件的路径。

以下是如何使用 python 和通过命令行恢复中断训练的示例：

恢复训练示例

`from ultralytics import YOLO # Load a model model = YOLO("path/to/last.pt")  # load a partially trained model # Resume training results = model.train(resume=True)`

`# Resume an interrupted training yolo train resume model=path/to/last.pt`

通过设置 `resume=True`， `train` 函数将从上次停止的地方继续训练，使用存储在 'path\\/to\\/last.pt' 文件中的状态。如果省略 `resume` 参数或将其设置为 `False`， `train` ，该函数将启动新的训练会话。

请记住，默认情况下，检查点会在每个 epoch 结束时保存，或者使用 `save_period` 参数按固定间隔保存，因此您必须至少完成 1 个 epoch 才能恢复训练运行。

训练设置
----

YOLO 模型的训练设置包括训练过程中使用的各种超参数和配置。这些设置会影响模型的性能、速度和[准确性](https://www.ultralytics.com/glossary/accuracy)。关键训练设置包括批量大小、学习率、动量和权重衰减。此外，优化器的选择、[损失函数](https://www.ultralytics.com/glossary/loss-function)和训练数据集组成会影响训练过程。仔细调整和试验这些设置对于优化性能至关重要。

参数

类型

默认值

描述

`weight_decay`

`float`

`0.0005`

L2 [正则化](https://www.ultralytics.com/glossary/regularization)项，惩罚大权重以防止过拟合。

`momentum`

`float`

`0.937`

SGD 的动量因子或 [Adam 优化器](https://www.ultralytics.com/glossary/adam-optimizer)的 beta1，影响当前更新中过去梯度的合并。

`resume`

`bool`

`False`

从上次保存的检查点恢复训练。自动加载模型权重、优化器状态和 epoch 计数，无缝继续训练。

`cos_lr`

`bool`

`False`

使用余弦[学习率](https://www.ultralytics.com/glossary/learning-rate)调度器，在 epochs 上按照余弦曲线调整学习率。有助于管理学习率，从而实现更好的收敛。

`save_period`

`int`

`-1`

保存模型检查点的频率，以 epoch 为单位指定。值为 -1 时禁用此功能。适用于在长时间训练期间保存临时模型。

`freeze`

`int` 或 `list`

`None`

冻结模型的前 N 层或按索引指定的层，从而减少可训练参数的数量。适用于微调或[迁移学习](https://www.ultralytics.com/glossary/transfer-learning)。

`mask_ratio`

`int`

`4`

分割掩码的下采样率，影响训练期间使用的掩码分辨率。

`dfl`

`float`

`1.5`

分布焦点损失的权重，在某些 YOLO 版本中用于细粒度分类。

`dropout`

`float`

`0.0`

分类任务中用于正则化的 Dropout 率，通过在训练期间随机省略单元来防止过拟合。

`cls`

`float`

`0.5`

分类损失在总损失函数中的权重，影响正确类别预测相对于其他成分的重要性。

`lr0`

`float`

`0.01`

初始学习率（即 `SGD=1E-2`, `Adam=1E-3`)。调整此值对于优化过程至关重要，它会影响模型权重更新的速度。

`save`

`bool`

`True`

启用保存训练检查点和最终模型权重。可用于恢复训练或[模型部署](https://www.ultralytics.com/glossary/model-deployment)。

`cache`

`bool`

`False`

启用在内存中缓存数据集图像（`True`/`ram`），在磁盘上缓存（`disk`），或禁用缓存（`False`）。通过减少磁盘 I/O 来提高训练速度，但会增加内存使用量。

`rect`

`bool`

`False`

启用最小填充策略——批量中的图像被最小程度地填充以达到一个共同的大小，最长边等于 `imgsz`。可以提高效率和速度，但可能会影响模型精度。

`amp`

`bool`

`True`

启用自动[混合精度](https://www.ultralytics.com/glossary/mixed-precision)（AMP）训练，减少内存使用，并可能在对准确性影响最小的情况下加快训练速度。

`pose`

`float`

`12.0`

在为姿势估计训练的模型中，姿势损失的权重会影响对准确预测姿势关键点的强调。

`single_cls`

`bool`

`False`

在多类别数据集中，将所有类别视为单个类别进行训练。适用于二元分类任务或侧重于对象是否存在而非分类时。

`close_mosaic`

`int`

`10`

在最后 N 个 epochs 中禁用 mosaic [数据增强](https://www.ultralytics.com/glossary/data-augmentation)，以在完成前稳定训练。设置为 0 可禁用此功能。

`profile`

`bool`

`False`

在训练期间启用 ONNX 和 TensorRT 速度的分析，有助于优化模型部署。

`val`

`bool`

`True`

在训练期间启用验证，从而可以定期评估模型在单独数据集上的性能。

`patience`

`int`

`100`

在验证指标没有改善的情况下，等待多少个epoch后提前停止训练。通过在性能停滞时停止训练，有助于防止[过拟合](https://www.ultralytics.com/glossary/overfitting)。

`exist_ok`

`bool`

`False`

如果为 True，则允许覆盖现有的 project/name 目录。适用于迭代实验，无需手动清除之前的输出。

`kobj`

`float`

`2.0`

姿势估计模型中关键点对象性损失的权重，用于平衡检测置信度和姿势准确性。

`warmup_epochs`

`float`

`3.0`

学习率预热的 epochs 数，将学习率从低值逐渐增加到初始学习率，以在早期稳定训练。

`deterministic`

`bool`

`True`

强制使用确定性算法，确保可重复性，但由于限制了非确定性算法，可能会影响性能和速度。

`batch`

`int` 或 `float`

`16`

[批次大小](https://www.ultralytics.com/glossary/batch-size)，具有三种模式：设置为整数（例如， `batch=16`），自动模式，GPU 内存利用率为 60%（`batch=-1`），或具有指定利用率分数的自动模式（`batch=0.70`）。

`fraction`

`float`

`1.0`

指定用于训练的数据集比例。允许在完整数据集的子集上进行训练，这在实验或资源有限时非常有用。

`model`

`str`

`None`

指定用于训练的模型文件。接受指向 `.pt` 预训练模型或 `.yaml` 配置文件的路径。对于定义模型结构或初始化权重至关重要。

`device`

`int` 或 `str` 或 `list`

`None`

指定用于训练的计算设备：单个 GPU（`device=0`），多个 GPU（`device=[0,1]`），CPU（`device=cpu`），适用于 Apple 芯片的 MPS（`device=mps`），或自动选择最空闲的 GPU（`device=-1`）或多个空闲 GPU （`device=[-1,-1]`)

`classes`

`list[int]`

`None`

指定要训练的类 ID 列表。可用于在训练期间过滤掉并仅关注某些类。

`box`

`float`

`7.5`

[损失函数](https://www.ultralytics.com/glossary/loss-function)中框损失分量的权重，影响对准确预测[边界框](https://www.ultralytics.com/glossary/bounding-box)坐标的重视程度。

`data`

`str`

`None`

数据集配置文件的路径（例如， `coco8.yaml`）。此文件包含数据集特定的参数，包括训练和 [验证数据的路径](https://www.ultralytics.com/glossary/validation-data)，类别名称和类别数量。

`lrf`

`float`

`0.01`

最终学习率作为初始速率的一部分 = (`lr0 * lrf`），与调度器结合使用以随时间调整学习率。

`time`

`float`

`None`

最长训练时间（以小时为单位）。如果设置此参数，它将覆盖 `epochs` 参数，允许训练在指定时长后自动停止。适用于时间受限的训练场景。

`plots`

`bool`

`False`

生成并保存训练和验证指标的图表，以及预测示例，从而提供对模型性能和学习进度的可视化见解。

`nbs`

`int`

`64`

用于损失归一化的标称批量大小。

`workers`

`int`

`8`

用于数据加载的工作线程数（每个 `RANK` ，如果是多 GPU 训练）。影响数据预处理和输入模型的速度，在多 GPU 设置中尤其有用。

`imgsz`

`int`

`640`

用于训练的目标图像大小。图像被调整为边长等于指定值的正方形（如果 `rect=False`），为 YOLO 模型保留宽高比，但不为 RTDETR 保留。影响模型 [准确性](https://www.ultralytics.com/glossary/accuracy) 和计算复杂度。

`pretrained`

`bool` 或 `str`

`True`

确定是否从预训练模型开始训练。可以是一个布尔值，也可以是加载权重的特定模型的字符串路径。增强训练效率和模型性能。

`overlap_mask`

`bool`

`True`

确定是否应将对象掩码合并为单个掩码以进行训练，还是为每个对象保持分离。如果发生重叠，则在合并期间，较小的掩码会覆盖在较大的掩码之上。

`optimizer`

`str`

`'auto'`

训练优化器的选择。选项包括 `SGD`, `Adam`, `AdamW`, `NAdam`, `RAdam`, `RMSProp` 等等，或者 `auto` 用于基于模型配置自动选择。影响收敛速度和稳定性。

`epochs`

`int`

`100`

训练的总轮数。每个[epoch](https://www.ultralytics.com/glossary/epoch)代表对整个数据集的一次完整遍历。调整此值会影响训练时长和模型性能。

`name`

`str`

`None`

训练运行的名称。用于在项目文件夹中创建一个子目录，训练日志和输出存储在该子目录中。

`seed`

`int`

`0`

设置训练的随机种子，确保在相同配置下运行结果的可重复性。

`multi_scale`

`bool`

`False`

通过增加/减少来启用多尺度训练 `imgsz` 高达 `0.5` 在训练期间。训练模型，使其在多次迭代中更加准确 `imgsz` 在推理过程中。

`project`

`str`

`None`

项目目录的名称，训练输出保存在此目录中。允许有组织地存储不同的实验。

`warmup_bias_lr`

`float`

`0.1`

预热阶段偏差参数的学习率，有助于稳定初始 epochs 中的模型训练。

`warmup_momentum`

`float`

`0.8`

预热阶段的初始动量，在预热期间逐渐调整到设定的动量。

关于批量大小设置的说明

字段 `batch` 参数可以通过三种方式配置：

*   **固定 [批量大小](https://www.ultralytics.com/glossary/batch-size)**：设置一个整数值（例如， `batch=16`），直接指定每个批次的图像数量。
*   **自动模式（60% GPU 内存）**：使用 `batch=-1` 自动调整批量大小，以达到大约 60% 的 CUDA 内存利用率。
*   **具有利用率分数的自动模式**：设置一个分数值（例如， `batch=0.70`），以根据指定的 GPU 内存使用率分数调整批量大小。

增强设置和超参数
--------

数据增强技术对于提高 YOLO 模型的鲁棒性和性能至关重要，它通过在[训练数据](https://www.ultralytics.com/glossary/training-data)中引入变异性，帮助模型更好地泛化到未见过的数据。下表概述了每个数据增强参数的目的和效果：

参数

类型

默认值

范围

描述

[`hsv_h`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#hue-adjustment-hsv_h)

`float`

`0.015`

`0.0 - 1.0`

通过色轮的一小部分调整图像的色调，从而引入颜色变化。帮助模型在不同的光照条件下进行泛化。

[`hsv_s`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#saturation-adjustment-hsv_s)

`float`

`0.7`

`0.0 - 1.0`

通过一小部分改变图像的饱和度，从而影响颜色的强度。可用于模拟不同的环境条件。

[`hsv_v`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#brightness-adjustment-hsv_v)

`float`

`0.4`

`0.0 - 1.0`

通过一小部分修改图像的明度（亮度），帮助模型在各种光照条件下表现良好。

[`degrees`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#rotation-degrees)

`float`

`0.0`

`0.0 - 180`

在指定的角度范围内随机旋转图像，提高模型识别各种方向物体的能力。

[`translate`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#translation-translate)

`float`

`0.1`

`0.0 - 1.0`

通过图像尺寸的一小部分在水平和垂直方向上平移图像，帮助学习检测部分可见的物体。

[`scale`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#scale-scale)

`float`

`0.5`

`>=0.0`

通过增益因子缩放图像，模拟物体与相机的不同距离。

[`shear`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#shear-shear)

`float`

`0.0`

`-180 - +180`

按指定的角度错切图像，模仿从不同角度观察物体的效果。

[`perspective`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#perspective-perspective)

`float`

`0.0`

`0.0 - 0.001`

对图像应用随机透视变换，增强模型理解 3D 空间中物体的能力。

[`flipud`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#flip-up-down-flipud)

`float`

`0.0`

`0.0 - 1.0`

以指定的概率将图像上下翻转，增加数据变化，而不影响物体的特征。

[`fliplr`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#flip-left-right-fliplr)

`float`

`0.5`

`0.0 - 1.0`

以指定的概率将图像左右翻转，有助于学习对称物体并增加数据集的多样性。

[`bgr`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#bgr-channel-swap-bgr)

`float`

`0.0`

`0.0 - 1.0`

以指定的概率将图像通道从 RGB 翻转到 BGR，有助于提高对不正确通道排序的鲁棒性。

[`mosaic`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#mosaic-mosaic)

`float`

`1.0`

`0.0 - 1.0`

将四个训练图像组合成一个，模拟不同的场景组成和物体交互。对于复杂的场景理解非常有效。

[`mixup`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#mixup-mixup)

`float`

`0.0`

`0.0 - 1.0`

混合两个图像及其标签，创建一个合成图像。通过引入标签噪声和视觉变化，增强模型的泛化能力。

[`cutmix`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#cutmix-cutmix)

`float`

`0.0`

`0.0 - 1.0`

组合两张图像的部分区域，创建局部混合，同时保持清晰的区域。通过创建遮挡场景来增强模型的鲁棒性。

[`copy_paste`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#copy-paste-copy_paste)

`float`

`0.0`

`0.0 - 1.0`

_仅分割_。在图像中复制和粘贴对象，以增加对象实例。

[`copy_paste_mode`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#copy-paste-mode-copy_paste_mode)

`str`

`flip`

\-

_仅分割_。指定了 `copy-paste` 要使用的策略。选项包括 `'flip'` 和 `'mixup'`.

[`auto_augment`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#auto-augment-auto_augment)

`str`

`randaugment`

\-

_仅分类_。应用预定义的增强策略（`'randaugment'`, `'autoaugment'`或 `'augmix'`）通过视觉多样性来增强模型性能。

[`erasing`](https://docs.ultralytics.com/zh/guides/yolo-data-augmentation/#random-erasing-erasing)

`float`

`0.4`

`0.0 - 0.9`

_仅分类_。在训练过程中随机擦除图像的区域，以鼓励模型关注不那么明显的特征。

可以调整这些设置以满足数据集和手头任务的特定要求。尝试不同的值可以帮助找到最佳的数据增强策略，从而获得最佳的模型性能。

信息

有关训练增强操作的更多信息，请参阅[参考部分](https://docs.ultralytics.com/zh/reference/data/augment/)。

日志记录
----

在训练 YOLO11 模型时，您可能会发现跟踪模型随时间变化的性能很有价值。这就是日志记录的用武之地。Ultralytics YOLO 支持三种类型的日志记录器 - [Comet](https://docs.ultralytics.com/zh/integrations/comet/)、[ClearML](https://docs.ultralytics.com/zh/integrations/clearml/) 和 [TensorBoard](https://docs.ultralytics.com/zh/integrations/tensorboard/)。

要使用记录器，请从上面的代码片段中的下拉菜单中选择它并运行它。将安装并初始化所选的记录器。

### Comet

[Comet](https://docs.ultralytics.com/zh/integrations/comet/) 是一个平台，允许数据科学家和开发人员跟踪、比较、解释和优化实验和模型。它提供诸如实时指标、代码差异和超参数跟踪等功能。

要使用 Comet：

示例

`# pip install comet_ml import comet_ml comet_ml.init()`

请务必先在其网站上登录您的 Comet 帐户，并获取您的 API 密钥。您需要将其添加到您的环境变量或脚本中，才能记录您的实验。

### ClearML

[ClearML](https://clear.ml/) 是一个开源平台，可以自动跟踪实验，并有助于高效地共享资源。它旨在帮助团队更有效地管理、执行和重现其 ML 工作。

要使用 ClearML：

示例

`# pip install clearml import clearml clearml.browser_login()`

运行此脚本后，您需要在浏览器上登录您的 ClearML 帐户并验证您的会话。

### TensorBoard

[TensorBoard](https://www.tensorflow.org/tensorboard) 是一个用于 [TensorFlow](https://www.ultralytics.com/glossary/tensorflow) 的可视化工具包。它允许您可视化您的 TensorFlow 图，绘制关于图执行情况的定量指标，并显示通过它的图像等附加数据。

要在 [Google Colab](https://colab.research.google.com/github/ultralytics/ultralytics/blob/main/examples/tutorial.ipynb) 中使用 TensorBoard：

示例

`load_ext tensorboard tensorboard --logdir ultralytics/runs # replace with 'runs' directory`

要在本地使用 TensorBoard，请运行以下命令并在以下位置查看结果 `http://localhost:6006/`.

示例

`tensorboard --logdir ultralytics/runs # replace with 'runs' directory`

这将加载 TensorBoard 并将其定向到保存训练日志的目录。

设置好您的记录器后，您可以继续进行模型训练。所有训练指标将自动记录在您选择的平台上，您可以访问这些日志来监控模型随时间的性能、比较不同的模型并确定需要改进的领域。

常见问题
----

### 如何使用 Ultralytics YOLO11 训练 [目标检测](https://www.ultralytics.com/glossary/object-detection) 模型？

要使用 Ultralytics YOLO11 训练目标检测模型，您可以使用 python API 或 CLI。以下是两者的示例：

单 GPU 和 CPU 训练示例

`from ultralytics import YOLO # Load a model model = YOLO("yolo11n.pt")  # load a pretrained model (recommended for training) # Train the model results = model.train(data="coco8.yaml", epochs=100, imgsz=640)`

`yolo detect train data=coco8.yaml model=yolo11n.pt epochs=100 imgsz=640`

有关更多详细信息，请参阅[训练设置](https://docs.ultralytics.com/zh/modes/train/#train-settings)部分。

### Ultralytics YOLO11 训练模式的主要功能是什么？

Ultralytics YOLO11 训练模式的主要功能包括：

*   **自动数据集下载：** 自动下载标准数据集，如 COCO、VOC 和 ImageNet。
*   **多 GPU 支持：** 在多个 GPU 上扩展训练，以加快处理速度。
*   **超参数配置：** 通过 YAML 文件或 CLI 参数自定义超参数。
*   **可视化和监控：** 实时跟踪训练指标，以获得更好的见解。

这些功能使训练高效且可根据您的需求进行定制。有关更多详细信息，请参见[训练模式的主要功能](https://docs.ultralytics.com/zh/modes/train/#key-features-of-train-mode)部分。

### 如何在 Ultralytics YOLO11 中从中断的会话恢复训练？

要从中断的会话恢复训练，请设置 `resume` 参数为 `True` 并指定上次保存的检查点的路径。

恢复训练示例

`from ultralytics import YOLO # Load the partially trained model model = YOLO("path/to/last.pt") # Resume training results = model.train(resume=True)`

`yolo train resume model=path/to/last.pt`

有关更多信息，请查看关于[恢复中断的训练](https://docs.ultralytics.com/zh/modes/train/#resuming-interrupted-trainings)的部分。

### 我可以在 Apple 芯片上训练 YOLO11 模型吗？

是的，Ultralytics YOLO11 支持在利用 Metal Performance Shaders (MPS) 框架的 Apple 芯片上进行训练。将 'mps' 指定为您的训练设备。

MPS 训练示例

`from ultralytics import YOLO # Load a pretrained model model = YOLO("yolo11n.pt") # Train the model on Apple silicon chip (M1/M2/M3/M4) results = model.train(data="coco8.yaml", epochs=100, imgsz=640, device="mps")`

`yolo detect train data=coco8.yaml model=yolo11n.pt epochs=100 imgsz=640 device=mps`

有关更多详细信息，请参阅[Apple Silicon MPS 训练](https://docs.ultralytics.com/zh/modes/train/#apple-silicon-mps-training)部分。

### 常见的训练设置有哪些，如何配置它们？

Ultralytics YOLO11 允许您通过参数配置各种训练设置，例如批量大小、学习率、epochs 等。这是一个简要概述：

参数

默认值

描述

`model`

`None`

用于训练的模型文件路径。

`data`

`None`

数据集配置文件的路径（例如， `coco8.yaml`）。

`epochs`

`100`

训练的总轮数。

`batch`

`16`

批处理大小，可调整为整数或自动模式。

`imgsz`

`640`

训练的目标图像大小。

`device`

`None`

用于训练的计算设备，例如 `cpu`, `0`, `0,1`或 `mps`.

`save`

`True`

启用保存训练检查点和最终模型权重。

有关训练设置的深入指南，请查看[训练设置](https://docs.ultralytics.com/zh/modes/train/#train-settings)部分。

  
  
📅 创建于 1 年前 ✏️ 更新于 2 个月前 [![glenn-jocher](https://avatars.githubusercontent.com/u/26833433?v=4&s=96)](https://github.com/glenn-jocher "glenn-jocher（21 次更改）")[![Laughing-q](https://avatars.githubusercontent.com/u/61612323?v=4&s=96) ](https://github.com/Laughing-q "Laughing-q（3 次更改）")[![UltralyticsAssistant](https://avatars.githubusercontent.com/u/135830346?v=4&s=96) ](https://github.com/UltralyticsAssistant "UltralyticsAssistant (2 处更改)")[![MatthewNoyce](https://avatars.githubusercontent.com/u/131261051?v=4&s=96) ](https://github.com/MatthewNoyce "MatthewNoyce（2 次更改）")[![Y-T-G](https://avatars.githubusercontent.com/u/32206511?v=4&s=96) ](https://github.com/Y-T-G "Y-T-G (1 处更改)")[![JairajJangle](https://avatars.githubusercontent.com/u/25704330?v=4&s=96) ](https://github.com/JairajJangle "JairajJangle（1 处更改）")[![jk4e](https://avatars.githubusercontent.com/u/116908874?v=4&s=96) ](https://github.com/jk4e "jk4e (1 次更改)")[![RizwanMunawar](https://avatars.githubusercontent.com/u/62513924?v=4&s=96) ](https://github.com/RizwanMunawar "RizwanMunawar (1 项更改)")[![dependabot](https://avatars.githubusercontent.com/u/27347476?v=4&s=96) ](https://github.com/dependabot "dependabot（1 处更改）")[![fcakyon](https://avatars.githubusercontent.com/u/34196005?v=4&s=96) ](https://github.com/fcakyon "fcakyon (1 处更改)")[![Burhan-Q](https://avatars.githubusercontent.com/u/62214284?v=4&s=96)](https://github.com/Burhan-Q "Burhan-Q (1 处更改)")   

评论
--