+++

title = "在Atlas 200 DK中部署深度学习模型"
date = 2023-09-26T12:08:08+08:00
slug = "deploy-deep-learning-model-on-atlas"
description = "本文记录了将Tensorflow模型部署到Atlas 200 DK的过程，首先将SavedModel转换成ONNX模型，然后将ONNX模型转换为om格式，以及模型转换中遇到的一些问题"
tags = ["Atlas", "深度学习"]
categories = ["Tech"]
image = ""

+++

## 环境准备

本文部署的模型是使用TensorFlow2训练的ResNet网络，用于干扰信号识别，原本的模型文件将权重保存在`ResNet.h5`中。

从模型文件中读取权重，并使用测试数据进行推理

```python
import ResNet as ResNet
import numpy as np

# 干扰类型
class_names = ['CSNJ', 'CW', 'LFM', 'MTJ', 'PBNJ', 'PPNJ']

# 权重文件的存储路径
model_path = "ResNet.h5"

data_dimension = (256, 256, 1)
model = ResNet.ResNet50(data_dimension)
# 导入网络模型
model.load_weights(model_path)

# 读取测试数据
test_img = np.load('test_img.npy')

# 转换输入数据的维度
test_img = np.array(test_img)  # (256, 256)
test_img = np.expand_dims(test_img, axis=2)  # (256, 256, 1)
test_img = np.expand_dims(test_img, 0)  # (1, 256, 256, 1)

test_output = model.predict(test_img)

# 模型输出的干扰类型
pred = class_names[test_output.argmax()]
print(pred)
```

Atlas 200 DK的环境部署采用“开发环境与运行环境分设”的方案，开发环境使用Windows下Linux子系统中的Ubuntu 22.04.3 LTS，CANN版本为6.0.RC1.alpha005。

参考官方教程[开发环境与运行环境分设](https://www.hiascend.com/document/detail/zh/Atlas200DKDeveloperKit/1013/environment/atlased_04_0018.html)完成环境配置。

## cann-toolkit 配置

在Ubuntu中安装完Ascend-cann-toolkit后，运行模型转换工具`atc`时可能会出现报错
```bash
error while loading shared libraries: libascend_hal.so: cannot open shared object file: No such file or directory
```

这是由于无法找到库文件`libascend_hal.so`，此时可以进入Ascend-cann-toolkit的安装路径，寻找`libascend_hal.so`

```bash
xu@DESKTOP-9B4N33I:~$ cd Ascend/ascend-toolkit/latest/
xu@DESKTOP-9B4N33I:~/Ascend/ascend-toolkit/latest$ find . -name libascend_hal.so
./x86_64-linux/devlib/libascend_hal.so
```

将`libascend_hal.so`复制到任意路径，并将该路径添加到Ascend-cann-toolkit环境变量中的`LD_LIBRARY_PATH`。

例如将`libascend_hal.so`复制到`~/Ascend/missing_lib`
```bash
xu@DESKTOP-9B4N33I:~$ mkdir ~/Ascend/missing_lib
xu@DESKTOP-9B4N33I:~$ cp ~/Ascend/ascend-toolkit/latestx86_64-linux/devlib/libascend_hal.so ~/Ascend/missing_lib
```
在`~/.bashrc`中更改环境变量
```shell
export LD_LIBRARY_PATH=${ASCEND_TOOLKIT_HOME}/lib64:${ASCEND_TOOLKIT_HOME}/lib64/plugin/opskernel:${ASCEND_TOOLKIT_HOME}/lib64/plugin/nnengine:/home/xu/Ascend/missing_lib:$LD_LIBRARY_PATH
```

在终端输入`source ~/.bashrc`并重启终端，此时`atc`可以正常运行。

## 模型转换

由于TensorFlow2不再支持导出模型为FrozenGraphDef格式，而`atc`转换TensorFlow模型时只能使用FrozenGraphDef格式，所以本文采用的转换流程是，先将TensorFlow模型导出为SavedModel格式，再将模型转换为[ONNX](https://onnx.ai)格式，最后使用`atc`将ONNX模型转换为.om格式。

### 将TensorFlow模型导出为SavedModel

```python
import tensorflow as tf
import ResNet as ResNet
import os

model_path = 'ResNet.h5'  # 权重文件路径

data_dimension = (256, 256, 1)
model = ResNet.ResNet50(data_dimension)

# 读取网络模型
model.load_weights(model_path)

# export model to savedmodel
mobilenet_save_path = os.path.join("./model")
tf.saved_model.save(model, mobilenet_save_path)
```
### 将SavedModel转换为ONNX模型

使用[tf2onnx](https://github.com/onnx/tensorflow-onnx)将SavedModel转换为.onnx格式的模型。

```bash
python -m tf2onnx.convert --saved-model tensorflow-model-path --output model.onnx
```  
由于`atc`不支持ONNX的高版本算子，转换时`tf2onnx`的--opset 参数值需使用默认值15。

导出的ONNX模型可以使用[Netron](https://netron.app)查看网络结构。使用ONNX模型完成推理以验证模型

```python
import onnxruntime as onnxrt
import numpy as np

class_names = ['CSNJ', 'CW', 'LFM', 'MTJ', 'PBNJ', 'PPNJ']

# load model
model = onnxrt.InferenceSession("resnet.onnx",
                providers=['AzureExecutionProvider', 'CPUExecutionProvider'])
print("loading model success!")

# input data
test_img = np.load('test_img.npy')
test_img = np.array(test_img, dtype=np.float32)  # (256, 256)
test_img = np.expand_dims(test_img, axis=2)  # (256, 256, 1)
test_img = np.expand_dims(test_img, 0)  # (1, 256, 256, 1)

# input
sf_input = {model.get_inputs()[0].name:test_img}

# output
output = model.run(None, sf_input)
print("get output of spatial filter success!")
output = np.array(output)

pred = class_names[output.argmax()]
print(pred)
```

### 将ONNX模型转换为om格式

#### `domain_version`报错

在Ubuntu终端中运行`atc`模型转换工具

```bash
atc --model=resnet.onnx --framework=5 --output=resnet --soc_version=Ascend310
```

可能会出现报错

```bash
E16005: The model has [2] [--domain_version] fields, but only one is allowed.
```
这是由于在将模型转换为ONNX是产生了两个`(domain, version)`（domain_version的概念参考[ONNX Concepts](https://onnx.ai/onnx/intro/concepts.html)）。

获取ONNX模型中的`(domain,version)`

```python
import onnx

model = onnx.load("resnet.onnx")
print(model.opset_import)
```
得到输出
```bash
[domain: ""
version: 15
, domain: "ai.onnx.ml"
version: 2
]
```
此时只需去除多余的`(domain,version)`，保留一个即可
```python
import onnx

model = onnx.load("resnet.onnx")

# atc 模型转换只支持一个domain_version
if len(model.opset_import) > 1:
    # 删除多余的domain_version
    model.opset_import.pop()

# 将删除domain_version后的模型保存
onnx.save(model, "./resnet.onnx")
```

#### 模型输入维度报错

再次运行`atc`，出现报错

```bash
E10001: Value [-1] for parameter [input_2] is invalid. Reason: maybe you should set input_shape to specify its shape.
```
使用Netron查看网络结构，发现`input_2`是输入节点，Netron中显示的该节点信息为

```toml
name: input_2
tensor: float32[unk__618,256,256,1]
```

输入数据为四维张量，每一个维度分别表示N（数量）H（高）W（宽）C（通道数），例如前面模型推理时使用的数据维度为`(1, 256, 256, 1)`表示1个高和宽都为256，通道数为1的图像。

在导出的ONNX模型中维度N的数值为指定，需要在模型转换时使用`--input-shape`参数指定输入节点的维度

```bash
atc --model=resnet.onnx --framework=5 --output=resnet --input-shape="input_2:1,256,256,1" --soc_version=Ascend310
```

## 在Atlas 200 DK中进行模型推理

模型推理时使用的是[pyACL](https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/70RC1alpha003/infacldevg/aclpythondevg/aclpythondevg_0001.html)接口，并使用了第三库[acllite](https://gitee.com/ascend/samples/tree/master/python/common/acllite)。

```python
import numpy as np
import sys
import os

# 将acllite路径添加到python环境变量
path = os.path.dirname(os.path.abspath(__file__))
sys.path.append(os.path.join(path, ".."))
sys.path.append(os.path.join(path, "../../common"))
sys.path.append(os.path.join(path, "../../common/acllite"))

from acllite_model import AclLiteModel
from acllite_resource import AclLiteResource

MODEL_PATH = "../model/resnet.om"

# init
acl_resource = AclLiteResource()
acl_resource.init()
# load model
model = AclLiteModel(model_path=MODEL_PATH)

test_img = np.load("../data/test_img.npy")
test_img = np.array(test_img)  # (256, 256)
test_img = np.expand_dims(test_img, axis=2)  # (256, 256, 1)
test_img = np.expand_dims(test_img, 0)  # (1, 256, 256, 1)
test_img = test_img.astype(np.float32)

# get output of model
output = model.execute(test_img)

output = np.squeeze(output)
print(output)

class_names = ['CSNJ', 'CW', 'LFM', 'MTJ', 'PBNJ', 'PPNJ']
pred = class_names[output.argmax()]
print(pred)
```