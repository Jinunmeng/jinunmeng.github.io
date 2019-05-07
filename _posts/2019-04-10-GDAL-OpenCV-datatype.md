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

**1、GDAL\OpenCV\C++中的常用数据类型对比**  

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


**2、OpenCV中type函数返回值与类型之间关系**  

||C1|C2|C3|C4|
|CV_8U|0|8|16|24|
|CV_8S|1|9|17|25|
|CV_16U|2|10|18|26|
|CV_16S|3|11|19|27|
|CV_32S|4|12|20|28|
|CV_32F|5|13|21|29|
|CV_64F|6|14|22|30|  

其中C1, C2, C3, C4 指的是通道(Channel)数

*3、Vec类的定义*
```
template<typename _Tp, int n> class Vec : public Matx<_Tp, n, 1> {...};

typedef Vec<uchar, 2> Vec2b;
typedef Vec<uchar, 3> Vec3b;
typedef Vec<uchar, 4> Vec4b;

typedef Vec<short, 2> Vec2s;
typedef Vec<short, 3> Vec3s;
typedef Vec<short, 4> Vec4s;

typedef Vec<int, 2> Vec2i;
typedef Vec<int, 3> Vec3i;
typedef Vec<int, 4> Vec4i;

typedef Vec<float, 2> Vec2f;
typedef Vec<float, 3> Vec3f;
typedef Vec<float, 4> Vec4f;
typedef Vec<float, 6> Vec6f;

typedef Vec<double, 2> Vec2d;
typedef Vec<double, 3> Vec3d;

typedef Vec<double, 4> Vec4d;
typedef Vec<double, 6> Vec6d;
```