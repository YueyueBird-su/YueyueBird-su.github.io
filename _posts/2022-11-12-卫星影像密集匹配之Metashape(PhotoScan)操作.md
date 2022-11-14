---
layout:     post
title:      "卫星影像密集匹配之Metashape(PhotoScan)"
subtitle:   "DTM/DSM生成"
date:       2022-11-12 23:04:03
author:     "LpengSu"
header-img: https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114222355545.png
catalog: true
tags:
    - 个人提升
    - 学习记录 
    - InPho教程
    - 密集匹配
    - 总结
---

# PhotoScan卫星影像产品制作

> 测试影像：BJ3 三线阵卫星影像
>
> 版本号：version1.8.4
>
> CPU：AMD Ryzen 7 5800X
>
> 显卡：NVIDIA GeForce RTX 3080

## 一、预处理

1. 卫星影像预处理具体有以下步骤：

   - PanSharpen：多光谱与全色融合

   - Translate：将数据存储类型转为Uint8，加速处理

   - Gdaladdo：生成影像预览

     具体可参考之前发布的[GDAL预处理](https://yueyuebird-su.github.io/2022/11/08/%E5%8D%AB%E6%98%9F%E5%BD%B1%E5%83%8F%E5%AF%86%E9%9B%86%E5%8C%B9%E9%85%8D%E4%B9%8BGDAL%E9%A2%84%E5%A4%84%E7%90%86/)文章。

2. 在预处理之后，需要将卫星影像RPC参数与影像放在同一目录。

3. 勾选使用GPU加速处理

   ![image-20221114104858500](https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114104858500.png)

## 二、添加照片

PhotoScan在导入影像时能够自动读取影像RPC参数。

在菜单栏中选则【工作流程】，点击添加照片，选则需要处理的影像。

![image-20221114103746935](https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114103746935.png)

双击影像名可以查看影像预览：

<img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114110841789.png" alt="image-20221114110841789" style="zoom:50%;" />

如果只对一个区域进行测试，或者需要排除某些区域进行处理，可以在每一个影像中添加蒙版(mask)。

1. 在菜单栏中选则【照片】，点击选择图像，对需要处理的区域进行选取：

   ![image-20221114111806873](https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114111806873.png)

2. 选取结束后，右键菜单对Mask进行**添加**、**减去**、**反向**操作：

   <img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114112336947.png" alt="image-20221114112336947" style="zoom:80%;" />

3. 添加选择后，效果如下图所示，此时显示为排除情况，如果想只对该区域进行处理，可选择【倒置蒙版】的处理方式：

   <img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114125702272.png" alt="image-20221114125702272" style="zoom:80%;" />

4. 倒置蒙版的效果如图所示：

   <img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114202208463.png" alt="image-20221114202208463" style="zoom:80%;" />

5. 如果后续需要使用的话，可以右键当前影像导出mask：

   <img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114130124761.png" alt="image-20221114130124761" style="zoom:80%;" />

6. 为所有影像添加相应的mask。

## 三、对齐照片

在【工作流程】中选择【对齐照片】模块进行影像匹配，如果设置了mask，需要将参数设置为“key points”，其中各参数说明可参考该篇[文章](https://blog.csdn.net/rdw1246010462/article/details/125268486)

<img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114204535544.png" alt="image-20221114204535544" style="zoom:80%;" />

对齐效果如下：

<img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114205301216.png" alt="image-20221114205301216" style="zoom:80%;" />

## 四、建立密集点云

在菜单栏中选择【工作流程】，点击【建立密集点云】模块，设置密集点云参数，生成密集点云。

<img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114205442181.png" alt="image-20221114205442181" style="zoom:80%;" />

密集点云生成效果如图所示：

<img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114211943192.png" alt="image-20221114211943192" style="zoom:80%;" />

## 五、生成网格

在菜单栏中选择【工作流程】，点击【生成网格】模块，在模型中生成3D格网：

<img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114212453083.png" alt="image-20221114212453083" style="zoom:80%;" />

生成效果如图所示：

<img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114212653645.png" alt="image-20221114212653645" style="zoom:80%;" />

## 六、生成纹理

在菜单栏中选择【工作流程】，点击【生成纹理】模块，为格网添加纹理：

<img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114213713285.png" alt="image-20221114213713285" style="zoom:80%;" />

添加纹理后效果如图：

![image-20221114215836594](https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114215836594.png)

可通过以下方式修改3D模型的显示方式：

<img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114220027282.png" alt="image-20221114220027282" style="zoom: 80%;" />

## 七、生成平铺模型

Tield Model不需要生成网格和纹理即可得到，具体操作为在菜单栏中选择【工作流程】，点击【Build Tiled Model】模块，生成3D平铺模型。

<img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114213549665.png" alt="image-20221114213549665" style="zoom:80%;" />

生成效果如下图所示：

<img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114215803899.png" alt="image-20221114215803899" style="zoom:80%;" />

可通过以下方式选择Tield Model的显示方式：

<img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114215936736.png" alt="image-20221114215936736" style="zoom:80%;" />

## 八、生成DSM/DEM

在菜单栏中选择【工作流程】，点击【Build DEM】模块，修改相关投影参数，如果不想对密集点云进行插值，可以禁用，最后点击OK生成DEM。

<img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114220219876.png" alt="image-20221114220219876" style="zoom:80%;" />

DEM效果如图：

![image-20221114220422551](https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114220422551.png)

## 九、生成正射影像

在菜单栏中选择【工作流程】，点击【Build Orthomosaic】模块，设置相关参数，点击OK生成正射影像。

<img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114220652686.png" alt="image-20221114220652686" style="zoom:80%;" />

正射影像生成效果如图所示：

![image-20221114221548940](https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114221548940.png)

## 十、导出对应产品

在菜单栏【文件】中，点击【导出】模块，可以将对应的产品使用指定的格式导出到对应的路径中。

<img src="https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20221114221720881.png" alt="image-20221114221720881" style="zoom:80%;" />
