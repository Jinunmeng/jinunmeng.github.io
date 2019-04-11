---
layout: post
title: "GDAL、OpenCV与C++中的数据类型对比"
subtitle: 'Comparison of data types in GDAL, OpenCV and C++'
author: "March"
#header-img: "img/post-bg-dreamer.jpg"
#header-mask: 0.4
header-style: text
tags:
  - GDAL
  - OpenCV
  - C++
---


在实际工作中会用到GDAL开源库进行影像读取，然后使用OpenCV中图像处理算法进行后续的处理，这过程涉及到的问题就是数据类型及数据格式转换。

GDAL\OpenCV\C++中的常用数据类型对比：

|**GDAL类型标识**|**OpenCV类型标识**|**C++中对应的类型**|**说明**|
|GDT_Unknown|| |未知数据类型|
|GDT_Byte|CV_8U|unsigned char|8bit正整型|
|GDT_UInt16 |CV_16U|unsigned short|16bit正整型|
|GDT_Int16 |CV_16S|short 或 short int|16bit整型|
|GDT_UInt32 ||unsigned long|32bit 正整型|
|GDT_Int32 |CV_32S|int 或 long 或 long int|32bit 整型|
|GDT_Float32 |CV_32F|float|32bit 浮点型|
|GDT_Float64|CV_64F|double|64bit 浮点型|


下面是使用GDAL读取影像，然后转OpenCV的Mat类型的简单例子（我读的是16位的影像，可以根据实际情况进行修改）：

```
Mat GdalToOpenCV(const char *pathFile)
{
	GDALAllRegister();          //GDAL所有操作都需要先注册格式
	CPLSetConfigOption("GDAL_FILENAME_IS_UTF8", "NO");

	GDALDataset *poDataset;
	poDataset = (GDALDataset*)GDALOpen(pathFile, GA_ReadOnly);
	//获取影像的宽高，波段数
	int width = poDataset->GetRasterXSize();
	int height = poDataset->GetRasterYSize();
	int nband = poDataset->GetRasterCount();
	
	float *buffer = new float[width*height];

	//读取一个波段，如果有多个波段需要遍历所有波段，然后再进行merge
	GDALRasterBand *bandData = poDataset->GetRasterBand(1);
	GDALDataType dataType = bandData->GetRasterDataType();
	bandData->RasterIO(GF_Read, 0, 0, width, height, buffer, width, height, dataType, 0, 0);
	Mat tempMat(height, width, CV_16U, buffer);
	Mat cvtMat(tempMat.size(), CV_8UC1);
	normalize(tempMat, cvtMat, 0, 255, CV_MINMAX);
	cvtMat.convertTo(cvtMat, CV_8U);
	//imwrite("re.tif", cvtMat);
	return cvtMat;
}

```

下面是多波段融合的例子：
```
vector<Mat> allMat;
//遍历所有波段
// OpenCV中存储彩色图像时，默认通道顺序为BGR，所以此处是逆着读取波段
for (int i = nband - 1; i > 0; i--)
{
    bandData = poDataset->GetRasterBand(i);
    GDALDataType dataType = bandData->GetRasterDataType();
    bandData->RasterIO(GF_Read, 0, 0, width, height, buffer, width, height, dataType, 0, 0);
    Mat tempMat(height, width, CV_16U, buffer);
    allMat.push_back(tempMat.clone());
}
//多通道融合
Mat mergeMat;
mergeMat.create(width, height, CV_16UC3);
merge(allMat, mergeMat);
```