+++

title = "PyTorch基础 （二）Dataset和DataLoader"
date = 2023-01-07T11:13:45+08:00
slug = "pytorch-dataset"
description = "PyTorch基础，Dataset和DataLoader"
tags = ["Python", "深度学习"]
categories = ["Notes"]
image = "/p/pytorch-tensor/pytorch.webp"

+++

PyTorch provides two data primitives: `torch.utils.data.DataLoader`and `torch.utils.data.Dataset` that allow you to use pre-loaded datasets as well as your own data.
`Dataset`stores the samples and their corresponding labels, and `DataLoader` wraps an iterable around the `Dataset` to enable easy access to the samples.

## loading a dataset

load the **Fashion-MNIST** dataset from TorchVision.
The [torchvision](https://pytorch.org/vision/stable/index.html#module-torchvision) package consists of popular datasets, model architectures, and common image transformations for computer vision.

```python
%matplotlib inline
import torch
from torch.utils.data import Dataset
from torchvision import datasets
from torchvision.transforms import ToTensor, Lambda
import matplotlib.pyplot as plt

training_data = datasets.FashionMNIST(
    root="data", # data storage path
    train=True, # load training data
    download=True, # download data from internet to root if data don't found at root
    transform=ToTensor() # data to tensor
)

test_data = datasets.FashionMNIST(
    root="data",
    train=False, # load test data
    download=True,
    transform=ToTensor()
)
```

## Iterating and Visualizing the Dataset

```python
labels_map = {
    0: "T-Shirt",
    1: "Trouser",
    2: "Pullover",
    3: "Dress",
    4: "Coat",
    5: "Sandal",
    6: "Shirt",
    7: "Sneaker",
    8: "Bag",
    9: "Ankle Boot",
}
figure = plt.figure(figsize=(8, 8))
cols, rows = 3, 3
for i in range(1, cols * rows + 1):
    sample_idx = torch.randint(len(training_data), size=(1,)).item()
    img, label = training_data[sample_idx]
    figure.add_subplot(rows, cols, i)
    plt.title(labels_map[label])
    plt.axis("off")
    plt.imshow(img.squeeze(), cmap="gray")
plt.show()
```

## Training with DataLoaders

While training a model, we use `DataLoader` to pass samples in "**minibatches**", reshuffle the data at every **epoch** to reduce model overfitting, and use Python's multiprocessing to speed up data retrieval.
To use `DataLoader`, we need to set the followings paraments:

- **dataset**-dataset from which to load the data
- **batch_size**-how many samples per batch to load
- **shuffle**-set to True to have the data reshuffled at every epoch (default: False)

```python
from torch.utils.data import DataLoader

train_dataloader = DataLoader(training_data, batch_size=64, shuffle=True)
test_dataloader = DataLoader(test_data, batch_size=64, shuffle=True)
```

We have loaded that dataset into the `Dataloader` and can iterate through the dataset as needed. Each iteration below returns a batch of `train_features` and `train_labels`(containing `batch_size=64` features and labels respectively). Because we specified `shuffle=True`, after we iterate over all batches the data is shuffled (for finer-grained control over the data loading order.

```python
# Display image and label.
train_features, train_labels = next(iter(train_dataloader))
# 得到batch_size=64的训练数据
print(f"Feature batch shape: {train_features.size()}")
# output: Feature batch shape: torch.Size([64, 1, 28, 28])
# batch_size为64，训练数据的每一项是一个28x28的图片
print(f"Labels batch shape: {train_labels.size()}")
# 绘制训练数据batch中第一个图像
img = train_features[0].squeeze()
label = train_labels[0]
plt.imshow(img, cmap="gray")
plt.show()
print(f"Label: {label}")
```

`iter(object[, sentinel])` 用于生成迭代器，传入参数object必须为支持迭代的对象，`next()` 返回迭代器下一项。

## Normalizatioin

Normalization is a common data pre-processing technique that is applied to scale or transform the data to make sure there's an equal learning contribution from each feature.

### Transforms

We use **transforms** to perform some manipulation of the data and make it suitable for training.
`transform` to modify the features and `target_transform` to modify the labels.
`ToTensor` converts a PIL image or NumPy `ndarray` into a `FloatTensor` and scales the image's pixel intensity values in the range [0., 1.]
