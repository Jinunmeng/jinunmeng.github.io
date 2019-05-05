---
layout: post
title: "深度学习中数据预处理的一些技巧"
subtitle: 'Deep Learning Data Preprocessings'
author: "March"
#header-img: "img/post-bg-dreamer.jpg"
#header-mask: 0.4
header-style: text
tags:
  - 深度学习
  - Python
---

利用深度学习进行图像分类中需要进行模型的训练，模型训练中需要进行数据的预处理。分类结果的精度与数据的好坏有很大的关系。  
下面将数据预处理中用到的函数列出来。  
**1.numpy.rollaxis函数**  
原型：numpy.rollaxis(arr, axis, start)  
	• **arr**：输入数组  
	• **axis**：该轴向后滚动，其他轴的位置不会相对于彼此改变  
	• **start**：默认为零。滚动到指定的位置  
有些数据集的图片默认格式是[channel][height][width]，有些地方需要改变通道位置即[height][width][channel].此时可以使用该函数进行设置。
```
 a = np.ones((480, 860,3))
 x = np.rollaxis(a, -1)
 x.shape
 (3, 480, 860)
 x = np.rollaxis(a, 2)
 x.shape
 (3, 480, 860)

```

**2.numpy.transpose函数矩阵转置**  
该函数可以实现与numpy.rollaxis一样的功能。  
（1）对于二维矩阵： x.transpose((0,1)) 表示按照原坐标轴改变序列，也就是保持不变 而 x.transpose((1,0)) 表示交换 ‘0轴’ 和 ‘1轴’。  
（2）对于三维矩阵：x.transpose((0,1,2))表示x保持不变；x.transpose((1,0,2))表示0轴和1轴交换；x.transpose((1，2，0))可以进行操作[channel][height][width]变为[height][width][channel]，或者x.transpose((2，0，1))进行相反操作。


**3.tf.slice函数**  
原型：tf.slice(input_, begin, size, name=None)  
	• input_：图片的矩阵输入格式  
	• begin：开始截取的位置（通常是[x,y,z]形式）  
	• size：从开始截取点向各维度截取的距离  
	• name：该tensor的名字  
解读：  
（1）input是一个3维向量  
（2）第二个参数[1,0,0]是截取的起始点，这里就是第2行的第一个数字‘3’开始。  
（3）[1,1,3]是截取的距离，第一维度截取1个距离，第二个维度截取1个距离，第3维度截取3个距离。  
（4）在第三个参数中如果用-1，如[1，-1，-1]，表示第2，3维度从起点一直截取到最后。  
（5）多维向量不能理解为线，面，体之类的。**有多少层符号“[]”就有多少维，从外层向内层，维度依次增加。**  
```
input = tf.constant([[[1, 1, 1], [2, 2, 2]],
                     [[3, 3, 3], [4, 4, 4]],
                     [[5, 5, 5], [6, 6, 6]]])
data = tf.slice(input, [1, 0, 0], [1, 1, 3])
print(sess.run(data))
#[[[3 3 3]]]

data = tf.slice(input, [1, 0, 0], [1, 2, 3])
print(sess.run(data))
# [[[3 3 3]
#   [4 4 4]]]

data = tf.slice(input, [1, 0, 0], [2, 2, 2])
print(sess.run(data))
# [[[3 3]
#   [4 4]]
#
#  [[5 5]
#   [6 6]]]

```
**4.利用tifffile库进行读写tif影像**  

**（1）影像的读写**  
```
>>> # numpy数组写入tif文件中
>>> data = numpy.random.rand(4, 301, 219)
>>> imsave('temp.tif', data, photometric='minisblack')

>>> # 读取tif影像到numpy数组中
>>> image = imread('temp.tif')
>>> numpy.testing.assert_array_equal(image, data)

```
**（2）TiffWriter**  
TiffWriter的主要目的是将n维numpy数组写入到TIFF中，而不是创建任何可能的TIFF格式，不支持JPEG压缩，SubIFDs，ExifIFD或GPSIFD标记
```
>>> # successively append images to BigTIFF file
>>> data = numpy.random.rand(2, 5, 3, 301, 219)
>>> with TiffWriter('temp.tif', bigtiff=True) as tif:
...     for i in range(data.shape[0]):
...         tif.save(data[i], compress=6, photometric='minisblack')
# compress:取值0到9或者'LZMA','ZSTD'.从0到9的值控制zlib压缩的级别，默认为0表示不压缩。
# photometric:图像数据的颜色空间,取值{'MINISBLACK', 'MINISWHITE', 'RGB', 'PALETTE', 'CFA'}
``` 
**（3）TiffFile**  
从TIFF文件中读取图像和元数据  
```
>>> # read image array from TIFF file
>>> imsave('temp.tif', numpy.random.rand(5, 301, 219))
>>> with TiffFile('temp.tif') as tif:
...     data = tif.asarray()
>>> data.shape
(5, 301, 219)
```





**持续更新中...**