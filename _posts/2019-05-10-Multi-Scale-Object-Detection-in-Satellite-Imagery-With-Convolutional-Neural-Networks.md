---
layout: post
title: "卫星图像进行快速目标识别——SIMRDWN的运行过程"
subtitle: 'The Satellite Imagery Multiscale Rapid Detection with Windowed Networks(SIMRDWN)'
author: "March"
#header-img: "img/post-bg-dreamer.jpg"
#header-mask: 0.4
header-style: text
tags:
  - GDAL
  - OpenCV
  - C++
---
翻译来自于[该文](https://github.com/CosmiQ/simrdwn)，如有错误欢迎提出。  
**The Satellite Imagery Multiscale Rapid Detection with Windowed Networks (SIMRDWN)**  
运行SIMRDWN的一些操作过程：  
**0、安装**  
(1) clone[源码](https://github.com/CosmiQ/simrdwn)  
(2) 安装[nvidia-docker](https://github.com/NVIDIA/nvidia-docker)  
(3) 构建docker文件  
```
cd /simrdwn/docker
nvidia-docker build --no-cache -t simrdwn .
```
(4) Spin up the docker container (see the docker docs for options)    
```
nvidia-docker run -it -v /simrdwn:/simrdwn --name simrdwn_container0 simrdwn
```
(5) 编译Darknet程序  
```
 cd /simrdwn/yolt2
 make
 cd /simrdwn/yolt3
 make
```
**1、准备数据**  
1A、创建YOLT结构数据
训练数据前需要将数据转换为YOLO结构："images"文件夹存放训练的原始图像，"lables"文件夹存放边框标签数据。比如"images/ex0.png"图像对应"labels/ex0.txt"标签。标签的格式为：  
```
<object-class> <x> <y> <width> <height>
```
其中，x,y,width,height表示图像的宽和高。执行/simrdwn/core/parse_cowc.py脚本可以从大型标记图像数据集[COWC](https://gdo152.llnl.gov/cowc/)中提取适当大小的训练窗口(范围通常为416或544像素)。脚本将对应于这些窗口的标签转换为正确的格式，并在/simdwn/data/training_list.txt中创建所有训练输入图像的列表,类名在YOLT中从0索引值开始的，在tensorflow中是从1开始的。我们还需要使用.pbtxt文件定义对象类，例如/simrdwn/data/class_labels_car.pbtxt。  
1B、创建.tfrecord格式数据(可选)  
如果使用tensorflow的目标检测api进行处理，需要将训练数据存储为.tfrecord格式。你可以使用simrdwn/core/preprocess_tfrecords.py脚本来完成。  
```
python /simrdwn/core/preprocess_tfrecords.py \
    --image_list_file /simrdwn/data/cowc_labels_car_list.txt \
    --pbtxt_filename /simrdwn/data/class_labels_car.pbtxt \
    --outfile /simrdwn/data/cowc_labels_car_train.tfrecord \
    --outfile_val /simrdwn/data/cowc_labels_car_val.tfrecord \
    --val_frac 0.1
```
 
**2、训练**  
我们既可以训练成YOLT模型也可以训练成tensoflow目标检测模型。如果使用tensorflow，需要修改/simrdwn/configs文件夹中的config文件（参考[配置文件](https://github.com/tensorflow/models/tree/master/research/object_detection/samples/configs)）。  
执行训练数据命令：  
```
# SSD vehicle search
python /simrdwn/core/simrdwn.py \
	--framework ssd \
	--mode train \
	--outname inception_v2_3class_vehicles \
	--label_map_path /simrdwn/data/class_labels_airplane_boat_car.pbtxt \
	--tf_cfg_train_file /simrdwn/configs/_altered_v0/ssd_inception_v2_simrdwn.config \
	--train_tf_record /simrdwn/data/labels_airplane_boat_car_train.tfrecord \
	--max_batches 30000 \
	--batch_size 16 \
	--gpu 0
	
# YOLT vechicle search
python /simrdwn/core/simrdwn.py \
	--framework yolt2 \
	--mode train \
	--outname dense_3class_vehicles \
	--yolt_cfg_file ave_dense.cfg  \
	--weight_dir /simrdwn/yolt2/input_weights \
	--weight_file yolo.weights \
	--yolt_train_images_list_file labels_airplane_boat_car_list.txt \
	--label_map_path /simrdwn/data/class_labels_airplane_boat_car.pbtxt \
	--nbands 3 \
	--max_batches 30000 \
	--batch_size 64 \
	--subdivisions 16 \
	--gpu 0
```
训练过程中会在/simrdwn/results文件夹中创建以[framework]+[outname]+[date]命名的结果目录。因为YOLT不能使用Tensorboard工具，所以可以执行/simrdwn/core/yolt_plot_loss.py和/simrdwn/core/tf_plot_loss.py脚本来查看训练的收敛情况。  
**3、测试**  
在测试阶段，可以处理任意大小的输入图像。处理过程如下：  
（1）将测试图像切片成训练使用的窗口大小；  
（2）然后对每个切片窗口进行预；  
（3）将所有窗口拼接成原始测试图像的大小；  
（4）对重叠预测的区域使用非最大抑制法进一步处理；  
（5）生产预测图（可选）。  





