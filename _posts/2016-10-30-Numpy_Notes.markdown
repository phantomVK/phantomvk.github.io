---
layout:     post
title:      "NumPy学习手记"
date:       2016-10-30
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - NumPy
    - Python
---

Python原生提供多种数据类型的支持，包括列表、元组、字典、集合等。不过对于数据统计、数据挖掘、机器学习来说，这些支持不够用，而NumPy则是数学运算库的不二之选。


# 一、安装或升级

没有`pip`的请先安装Python包管理工具。

安装NumPy:

```bash
$ pip install numpy
```

升级NumPy:

```bash
$ pip install --upgrade numpy
```

# 二、导入NumPy
请自行在文件头导入NumPy库。如没有特殊情况，后文将省略这段代码。

导入numpy所有模块，请注意不要和其他库冲突

```python
from numpy import *
```


# 三、功能使用

## 3.1 mat() 数组转矩阵

使用random函数随机生成 4x4 的数组，然后mat()把数组转换为矩阵。为了方便学习，接下来的几个例子都用这里生成的变量randMat。

```python
>>> randMat = mat(random.rand(4,4))
>>> print randMat
[[ 0.63795383  0.16175714  0.47359276  0.97462684]
 [ 0.45715522  0.77251449  0.66504488  0.07133683]
 [ 0.61471841  0.23323568  0.95002458  0.38985276]
 [ 0.26085195  0.42547952  0.84814459  0.35398749]]
```


## 3.2 shape() 查看维度

使用上个例子的randMat变量，查看这个数组的维度。

```python
>>> print shape(randMat)
(4, 4)
```

矩阵行

```python
>>> print shape(randMat)[0]
4
```

矩阵的列

```python
>>> print shape(randMat)[1]
4
```

## 3.3 size 元素个数

总共16个元素

```python
>>> print randMat.size
16
```

## 3.4 逆矩阵

注意变量后面的参数

```python
>>> print randMat.I
[[ 0.29673207  1.05925085  1.8179092  -3.03254655]
 [ 0.36225232  1.41847049 -1.66390977  0.54925607]
 [-0.74644908 -0.81502459  0.82412699  1.31180452]
 [ 1.13439787 -0.53272813 -1.31423984  1.25639622]]
```
 
## 3.5 tile() 函数

定义一个比较简单的1x2矩阵

```python
>>> a = [1, 2]
```

____

输入矩阵和参数值2

```python
>>> print tile(a, 2)
[1 2 1 2]

>>> print tile(a, 4)
[1 2 1 2 1 2 1 2]
```

____

把参数改为元组(1,2)，就变成了1x4的矩阵

```python
>>> print tile(a, (1,2))
[[1 2 1 2]]
```

____

参数改为(2, 2)，变成2x4的矩阵，行和列都是原来的2倍

```python
>>> print tile(a, (2,2))
[[1 2 1 2]
 [1 2 1 2]]
```

____

把a在行上拓展

```python
>>> print tile(a, (4,1))
[[1 2]
 [1 2]
 [1 2]
 [1 2]]
```

## 3.6 sum() 总计

按列总计

```python
>>> print sum([[0,1,2],[2,1,3]],axis=0)
[2 2 5]
```

____

按行总计

```python
>>> print sum([[0,1,2],[2,1,3]],axis=1)
[3 6]
```


## 3.7 argsort()

返回按升序排列的下标值

```python
>>> x = array([1, 2, 3])
>>> argsort(x)
array([0, 1, 2])
```

```python
>>> x = array([3, 2, 1])
>>> argsort(x)
array([2, 1, 0])
```


```python
>>> x = array([3, 1, 2])
>>> argsort(x)
array([1, 2, 0])
```

多维矩阵的排序，使用axis参数指定排序的条件

```python
>>> x = array([[0, 3], [2, 2]])
>>> argsort(x, axis=0)
array([[0, 1],
       [1, 0]])
```

```python
>>> argsort(x, axis=1)
array([[0, 1],
       [0, 1]])
```

按降序排列

```python
>>> x = array([1,2,3])
>>> argsort(-x)
array([2, 1, 0])
```

## 3.8 创建0矩阵

1x4浮点型0矩阵：

```python
>>> print zeros(4)
[0. 0. 0. 0.]
```
____

创建一个4x4的浮点型0矩阵:

```python
>>> print zeros([4,4])
[[ 0.  0.  0.  0.]
 [ 0.  0.  0.  0.]
 [ 0.  0.  0.  0.]
 [ 0.  0.  0.  0.]]
```

在`zeros()`里面，注意填写的`[4,4]`，分别是行和列。

____

整形的0矩阵：

```python
>>> print zeros([4,4],int16)
[[0 0 0 0]
 [0 0 0 0]
 [0 0 0 0]
 [0 0 0 0]]
```


## 3.9 全1矩阵

2x3的全1矩阵

```python
>>> print ones([2,3])
[[ 1.  1.  1.]
 [ 1.  1.  1.]]
```

____

生成两个2x3全1矩阵

```python
>>> print ones([2,2,3])
[[[ 1.  1.  1.]
  [ 1.  1.  1.]]

 [[ 1.  1.  1.]
  [ 1.  1.  1.]]]
```

____

生成两个整形2x3全1矩阵

```python
>>> print ones([2,2,3],int16)
[[[1 1 1]
  [1 1 1]]

 [[1 1 1]
  [1 1 1]]]
```

