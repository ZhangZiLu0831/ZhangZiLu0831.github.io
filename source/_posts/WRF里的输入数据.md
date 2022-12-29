---
title: 关于WRF的输入与初始化——WRF的输入数据与如何修改
date: 2022-11-11 20:13:48
tags:
- WRF
- 数值模式
categories: 垃圾笔记
---
# 理想VS真实(Ideal VS Real)
WRF主要可以进行两种模拟，即理想（ideal）模拟与真实（real）模拟。
二者最大的差距在于是否经过WPS的前处理，对于理想模拟而言，只要将2D与3D气象数据便可进行理想模拟，而对于真实模拟而言，也就是我们通常使用的模拟而言，我们需要使用WPS进行前处理，再使用real.exe进行初始化。
# real的初始化
真实数据用例做一些额外的处理:
1. 从WRF预处理系统(WPS)读取气象和静态输入数据。
2. 准备模型中使用的下垫面(通常，垂直插值到指定地表方案所需的水平)，检查输入的静态数据是否一致。
3. 不同时间段的横向边界。
4. 三维边界数据
## 生成real的数据
1. WPS所需的三维气象数据:压力、u、v、温度、相对湿度、位势高度。
2. 可选的3-D水文数据可以在运行时提供给真正的程序，但这些字段不会在粗网格横向边界文件中使用。命名为:QR、QC、QS、QI、QG、QH、QNI(雨、云、雪、冰、霰、冰雹的混合比和数量浓度)的字段可以从metgrid输出文件中输入。
3. 三维土壤数据:土壤温度、土壤湿度、土壤液体。
4. 二维气象数据:海平面压力、地表压力、地表u和v、地表温度、地表相对湿度、初步猜测地形高程，可选二维气象数据：海面温度、物理雪深、雪水当量。
5. 静态地理数据:地形高程、土地利用分类、土壤质地分类、时间插值月数据、陆海掩膜、输入模型的地形高程。
6. 二维静态投影数据:地图因子，科里奥利，投影旋转，计算纬度。
7. 其他3D数据：排放、示踪等等。
## ERA5中的驱动数据
3D数据：ERA5 pressure levels data，温湿压和u v风、

2D数据：所有层的土壤数据、海平面压力、地表压力、地表u和v、地表温度、地表相对湿度、初步猜测地形高程，可选二维气象数据：海面温度、物理雪深、雪水当量。
## 需要改进的数据
1. 静态数据：反照率（与地面静态数据有关）。
2. 2D气象数据：海温、雪深、雪水当量、海冰场。

## Albedo与Snow albedo
**WRF的物理模型并不能输出Abledo，Albedo根据输入的静态的地理数据（土地利用类型）有关，并根据Vtable决定变化。**

静态数据有： albedo_modis /lai_modis_10m /maxsnowalb_modis /greenfrac_fpar_modis /lai_modis_30s /modis_landuse_20class_30s_with_lakes

修改时，应当设置namelist.input中：
```
&physics 
usemonalb = true, 
rdlai2d = true, 
sst_update=1 
```

在模式中单纯修改反照率，如果面积过大，就会造成地面平衡破坏太严重，模式易于崩溃。

由于模式会在计算过程中产生降雪，导致反射率发生变化。因此，我们需要将雪从模式中除去。方法就是每过一段时间重启一次，然后对wrfrst文件中的雪进行修改。

修改反射率与修改雪的范例代码如下所示：
```
f1	= addfile("wrfinput_d01","w") 
f2	= addfile("wrflowinp_d01","w")
fs		= systemfunc("ls -t wrfrst*")
f3		= addfile(fs(0),"w")

albedo1	= f1->ALBBCK
albedo2	= f2->ALBBCK
albedo3	= f3->ALBBCK
albedo3a	= f3->ALBEDO
albedo3b	= f3->SNOALB
snow1	= f3->ACSNOW
snow2	= f3->SNOW
snow3	= f3->SNOWH
snow4	= f3->SNOWC

albedo1(:,100:101,62:63)	=	0.305
albedo2(:,100:101,62:63)	=	0.305

f1->ALBBCK	= albedo1
f2->ALBBCK	= albedo2

albedo3(:,100:101,62:63)	=	0.305
albedo3a(:,100:101,62:63)	=	0.305
albedo3b(:,100:101,62:63)	=	0.305

snow1(:,100:101,62:63)	=	0
snow2(:,100:101,62:63)	=	0
snow3(:,100:101,62:63)	=	0
snow4(:,100:101,62:63)	=	0

f3->ALBBCK	= albedo3
f3->ALBDO	= albedo3a
f3->SNOALB	= albedo3b
f3->ACSNOW	= snow1
f3->SNOW	= snow2
f3->SNOWH	= snow3
f3->SNOWC	= snow4
```

---

If you map data to the 24-category USGS data, then albedo will be given at the model start time, based on the LANDUSE.TBL. Since the initial value is dependent on the landuse type, however, the Noah LSM (since V3.1) does not use the albedo (or any other land properties) from the LANDUSE.TBL. It uses values from the VEGPARM.TBL, and albedo is computed based on vegetation fraction, and boudned by the max and min values given in the VEGPARM.TBL (according to UCAR).

 Do you use the Noah scheme? The difference between the two variables will be important depending the type of soil, the time of the year and the ice/snow (or not) cover. Which area does your domain cover? The "albedo" variable would be an average number depending on the time of the year of your simulations. The background would be the actual one, for instance if you had snowfall during your run, you should have more albedo than without it. 

 ---
当使用Noah地面方案时，WRF **首先读取LANDUSE.TBL 表中规定的陆面属性参数值。如果VEGPARM.TBL表中有相同的参数, 则这些表中规定的陆面属性参数值将被VEGPARM.TBL表中的值代替。**

Albedo的计算依赖于土地利用（植被）类型和FVC, 可以通过FVC及其最大值、最小值来计算。
 
 Albedo的最大值、最小值在VEGPARM.TBL 表中进行查找。当usemonalb 选项设置为true时, 将读取来源于WPS geogrid程序的12个Albedo月值。LAI值一方面可以通过FVC和VEGPARM.TBL表中给定的其最大值、最小值来计算, 另一方面也可以通过读取外部的数据, 如MODIS LAI遥感数据。
 
 当读取外部LAI数据时, 将rdlai2d设置为true, 并将LAI数据格式转换为WPS metgrid.exe 要求的格式。例如, MODIS LAI遥感资料处理外部步骤包括：
 
 **对MODIS影像进行拼接与裁剪, 使其覆盖整个研究区域；采用单位网格区域平均的方法, 将遥感的分辨率处理到WRF模式水平网格分辨率；对模式网格点上的数据进行插补与修正。内部步骤为：先将处理后的MODIS LAI外部数据写入WPS中, 然后对WRF进行初始化, 并运行WRF。VEGPARM.TBL表中的FVC仍然通过1986~1991年的AVHRR NDVI数据得到, 空间分辨率为0.144º×0.144º, 时间分辨率为月。当sst_update选项等于1时, 将读取12个来源于WPS geogrid程序的MODIS/AVHRR的FVC月值数据。**

## 2D气象数据
即雪深、雪水当量、海冰场等等。

可以使用卫星数据进行修改，主要步骤为：将数据转为nc格式，并使用NCL将其转为WPS中间文件，修改MET.TBL文件即可读取。

# 难点评估与下一步
## 存在的难点与疑问：
1. 是否需要修改WPS下垫面？如何实现？修改什么？

——WPS下垫面的修改可分为直接修改静态数据与修改插入的gem文件，后者较为简单。

——WPS静态数据可以利用QGIS中的GIS4WRF插件直接转换实现。

——修改后还需配合数据修改index索引文件与GEOGRID.TBL文件，一个有用的示例：[WRF修改下垫面](https://blog.csdn.net/weixin_42181785/article/details/114178200?spm=1001.2014.3001.5506)

—— 考虑修改Albedo数据与max snow albedo数据，根据观测，对北极不同的区域修正。


2. 如何修改雪反照率？Noah中，雪反照率的两种计算方法？
   
——需要在namelist.input中进行设置对应参数。（见前文）

——修改VEGPARM.TBL

——是直接修改wrffrist？还是修改VEGPARM？还是两种一起。需要进行尝试模拟，进行结果。

## 下一步

下一步的重点应为对雪反照率的改变尝试。

先尝试对于静态数据的直接修改，再考虑修改wrffirst。

海温、雪深、海冰等二维数据的添加之前已做过类似，暂且不考虑。

具体安排：

1. 先绘制Domain区域土地利用图，查看下垫面分布。
2. 了解namelist.input中，三个设置的具体作用与实现。
3. 学习LANDUSE.TBL VEGPARM.TBL。
4. 动手尝试修改，敏感性实验。
   

使用工具：QGIS、NCL、python。







