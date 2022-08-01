---
layout:     post
title:      "《数字图像处理》常用函数"
subtitle:   "影像几何校正"
date:       2020-11-17 12:00:00
author:     "LpengSu"
header-img: "https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/20201117.jpg"
catalog: true
tags:
    - 个人提升
    - 总结
    - 学习记录 
    - GDAL
    - Eigen
    - 专业知识
    - RS
    - 常用个人代码
---


#### 一、图像处理

##### 读取影像数据

```c
/*读取影像数据*/
MatrixXd readImgData(const char* path) {
	//注册文件格式
	GDALAllRegister();
	//设置支持中文路径
	CPLSetConfigOption("GDAL_FILENAME_IS_UTF8", "NO");
	//图像读取
	GDALDataset* pathDataset = (GDALDataset*)GDALOpen(path, GA_ReadOnly);
	if (pathDataset == NULL) {
		std::cout << path << "图像打开失败" << std::endl;
		exit(0);
	}
	else {
		std::cout << "图像打开成功！\n\n";
	}
	//图像属性设置
	const int bandNum = pathDataset->GetRasterCount();//波段数
	const int xNum = pathDataset->GetRasterXSize();//宽度
	const int yNum = pathDataset->GetRasterYSize();//高度
	GDALDataType dataType_i = pathDataset->GetRasterBand(1)->GetRasterDataType();
	//输出图像大小和波段信息
	std::cout << "图像大小为(" << yNum << "," << xNum << ")\n" << "数据个数为：" << xNum * yNum << "\n波段数为：" << bandNum << "\n\n";
	//初始化矩阵大小
	MatrixXd imgData(yNum, xNum);
	//读取图像的第一个波段的数据
	GDALRasterBand* pathBand_i = pathDataset->GetRasterBand(1);
	unsigned char* puData = new unsigned char[xNum * yNum];//暂存矩阵
	pathBand_i->RasterIO(GF_Read, 0, 0, xNum, yNum, puData, xNum, yNum, dataType_i, 0, 0);
	//将一维数组中的数据传入二维数组
	for (int i = 0; i < yNum; i++) {
		for (int j = 0; j < xNum; j++) {
			imgData(j, i) = (double)puData[i * xNum + j];
		}
	}
	GDALClose(pathDataset);
	return imgData;
}
```

##### 读取参考影像和待校正影像同名点

```c
/*从参考影像和待校正影像中读取同名点*/
MatrixXd* computeData(MatrixXd refeImg, MatrixXd falseImg) {
	MatrixXd* falseTemps = new MatrixXd[36];//存储待校正影像中36个模板
	MatrixXd tempCenterMatrix(36, 3);	//存储模板中心点
	MatrixXd refeSamePoint(36, 3);//存储同名点位置和对应的NCC值
	cout << "###############################\n";
	cout << "待校正影像模板中心点为：\n";
	int num = 0;
	for (int x = 90; x < falseImg.rows(); x = x + 61) {//以（90，90）为第一个中心点开始查找同名点
		for (int y = 90; y < falseImg.cols(); y = y + 61) {
			if (x == 90 || x == 90 + 61 * 9 || y == 90 || y == 90 + 61 * 9) {
				tempCenterMatrix(num, 0) = x;
				tempCenterMatrix(num, 1) = y;
				falseTemps[num] = falseImg.block(x - 30, y - 30, 21, 21);
				num++;
				printf("%d\t(%d,%d)\n", num, x, y);
			}
			else {
				continue;
			}
		}
	}

	//在参考影像中查找同名点
	cout << "###############################\n";
	cout << "参考影像同名点为：\n(大概需要等待20分钟)\n";
	for (int k = 0; k < 36; k++) {
		//设置临时参数
		double max_NCC = 0;
		double max_i = 0;
		double max_j = 0;
		//查找   k   模板的同名点
		for (int i = 30; i < refeImg.rows() - 21; i++) {//中心点从（30，30）开始查找
			for (int j = 30; j < refeImg.cols() - 21; j++) {
				//MatrixXd refeTemple = find_t_in_i(refeImg, i, j, 61, 61);
				MatrixXd refeTemple = refeImg.block(i - 30, j - 30, 21, 21);//从i，j取61x61的矩阵大小
				double ncc = 0;
				ncc = runNCC(refeTemple, falseTemps[k]);//计算ncc
				float location = (k * ((refeImg.rows() - 21) * (refeImg.cols() - 21)) + i * (refeImg.cols() - 21) + j) * 100 / (36 * float(refeImg.rows() - 21) * (refeImg.cols() - 21));
				printf("%.2f%%\b\b\b\b\b\b\b\b\b\b\b\b", location);

				if (ncc > max_NCC) {//记录NCC最大值和模板中心点坐标
					max_NCC = ncc;
					max_i = i;
					max_j = j;
				}
			}
		}
		//cout << k << "\t" << max_i << "\t" << max_j << "\t" << max_NCC << endl;
		refeSamePoint(k, 0) = max_i;
		refeSamePoint(k, 1) = max_j;
		refeSamePoint(k, 2) = max_NCC;
	}
	cout << "同名点查找完成！\n";
	cout << refeSamePoint << endl;

	MatrixXd* outData = new MatrixXd[2];
	outData[0] = tempCenterMatrix;//返回待校正影像同名点
	outData[1] = refeSamePoint;//返回参考影像同名点
	return outData;
}
```

##### 将数据存到txt文件 

```c
/*将同名点存储到txt文件*/
int saveData2txt(const char* path, MatrixXd tempCenterMatrix, MatrixXd refeSamePoint) {
	FILE* fp;//
	fp = fopen(path, "w");
	int num = 0;
	for (int i = 0; i < 36; i++) {
		if(refeSamePoint(i, 2) > 0.7) {
			num++;
			double x_f = tempCenterMatrix(i, 0);
			double y_f = tempCenterMatrix(i, 1);
			double x_t = refeSamePoint(i, 0);
			double y_t = refeSamePoint(i, 1);
			double ncc = refeSamePoint(i, 2);
			//string str = (string)x_f + '\t' + y_f + '\t';
			fprintf(fp, "%.0f  %.0f %.0f %.0f %.2f \n", x_f, y_f, x_t, y_t, ncc);
		}
	}

	printf("\n已成功将%d组数据输出到./Data/同名点.txt\n",num);
	fclose(fp);
	return num;//返回ncc>0.7的数量;
}
```

##### 从txt文件中读取同名点

```c
/*从已知txt文件读取数据*/
int readPathData(const char* path,MatrixXd *saveData) {
	//MatrixXd* saveData = new MatrixXd[2];
	saveData[0] = MatrixXd::Zero(36, 3);
	saveData[1] = MatrixXd::Zero(36, 3);

	FILE* fpr = fopen(path, "r");

	int i = 0;
	while (!feof(fpr))//当文件没有读完时
	{
		char* data1 = new char[100];
		char* data2 = new char[100];
		char* data3 = new char[100];
		char* data4 = new char[100];
		char* data5 = new char[100];
		fscanf(fpr, "%s %s %s %s %s ", data1, data2, data3, data4, data5);
		if (atof(data5) > 0.7) {
			saveData[0](i, 0) = atof(data1);
			saveData[0](i, 1) = atof(data2);
			saveData[1](i, 0) = atof(data3);
			saveData[1](i, 1) = atof(data4);
			saveData[1](i, 2) = atof(data5);
			i++;
		}
	}
	printf("\n##############################\n");
	printf("\n已成功读入%d组数据\n", i);
	//关闭文件 
	fclose(fpr);
	return i;
}
```

##### 保存计算数据为.tif格式 

```c
/*保存影像为tif格式*/
int saveImg(unsigned char* outUData, const char* path, int row, int col) {
	//获取驱动
	const char* pszFormat = "GTiff";
	GDALDriver* poDriver = GetGDALDriverManager()->GetDriverByName(pszFormat);
	if (!poDriver)
	{
		fprintf(stderr, "get driver by name failed\n");
		return -1;
	}


	//创建保存影像数据集
	char** papszOptions = nullptr;
	papszOptions = CSLSetNameValue(papszOptions, "INTERLEAVE", "BAND");
	const char* pszDstFilename = path;
	GDALDataset* poDstDS = poDriver->Create(pszDstFilename, col, row, 1, GDT_Byte, papszOptions);
	if (!poDstDS)
	{
		std::cout << "create Nosie_result.tif failed" << std::endl;
		return -1;
	}

	poDstDS->RasterIO(GF_Write, 0, 0, col, row, outUData, col, row, GDT_Byte, 1, 0, 0, 0, 0);
	cout << "影像输出完成！\n";
	cout << "###############################\n\n";
}
```

#### 二、几何影像匹配

##### 从待配准影像计算配准后影像的四个角点

```c
/*从待配准影像计算配准后影像的四个角点*/
void findTrueImgSize(MatrixXd falseImg, MatrixXd xf2r_Matrix, int *min_x, int* min_y, int* max_x , int* max_y) {
	int falseRow = falseImg.rows();
	int falseCol = falseImg.cols();
	RowVector3d* falseData = new RowVector3d[4];
	falseData[0] << 1, 0, 0;
	falseData[1] << 1, 0, falseCol;
	falseData[2] << 1, falseRow, 0;
	falseData[3] << 1, falseRow, falseCol;//对四个角点赋值
	double* x = new double[4];
	double* y = new double[4];	
	for (int i = 0; i < 4; i++) {
		x[i] = falseData[i] * xf2r_Matrix.col(0);
		y[i] = falseData[i] * xf2r_Matrix.col(1);
		if (x[i] < *min_x) {
			*min_x = int(x[i]);
		}
		else if (x[i] > *max_x) {
			*max_x = int(x[i]);
		}
		if (y[i] < *min_y) {
			*min_y = int(y[i]);
		}
		else if (y[i] > *max_y) {
			*max_y = int(y[i]);
		}
	}
}
```

##### 计算两个模板的NCC值

```c
/*计算两个大小相同的影像的NCC值*/
double runNCC(MatrixXd refeT, MatrixXd falseT) {
	//定义分子
	double num1 = 0;
	//定义分母
	double num2_refe = 0;
	double num2_f = 0;
	num1 = ((refeT.array() - refeT.mean()) * (falseT.array() - falseT.mean())).sum();
	num2_refe = ((refeT.array() - refeT.mean()).cwiseAbs2()).sum();
	num2_f = (falseT.array() - falseT.mean()).cwiseAbs2().sum();
	double ncc = num1 / (sqrt(num2_refe * num2_f));
	if (ncc <= 1) {
		return ncc;
	}
	else {
		std::cout << ncc << "\n";
		std::cout << "ncc绝对值大于1，请检查算法！！！\n\n";
		//exit(0);
	}
}
```

##### 