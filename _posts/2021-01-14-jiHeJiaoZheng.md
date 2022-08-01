---
layout:     post
title:      "INPHO软件卫星影像密集匹配教程"
subtitle:   "DTM/DSM生成"
date:       2022-08-01 21:00:00
author:     "LpengSu"
header-img: "img/posts_img/2020-11-17-JiHeJiaoZheng/bg.jpg"
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

> 更新时间 2021-01-14 于期末考试结束后

#### 一、GDAL读取/存储影像

##### 1.1 读取影像数据

通过GDAL数据类型的特点，使用switch自动设置读取的影像数据存储格式，再使用模板<template>自动适应影像格式，最终返回正确的数据格式。

```c
enum GDType {
	Unknown, Byte, UInt16, Int16
};
```



```c
/*读取影像数据*/
template <typename T>
MatrixXd* readImgData(const char* path, int* bandNum, T* outData) {
	//注册文件格式
	GDALAllRegister();
	//设置支持中文路径
	CPLSetConfigOption("GDAL_FILENAME_IS_UTF8", "NO");
	//图像读取
	GDALDataset* pathDataset = (GDALDataset*)GDALOpen(path, GA_ReadOnly);
	if (pathDataset == NULL) {
		cout << path << "图像打开失败" << std::endl;
		exit(0);
	}
	else {
		cout << "图像打开成功！\n\n";
	}
	//图像属性设置
	*bandNum = pathDataset->GetRasterCount();//波段数	
	const int xNum = pathDataset->GetRasterXSize();//宽度
	const int yNum = pathDataset->GetRasterYSize();//高度
	GDALDataType dataType_i = pathDataset->GetRasterBand(1)->GetRasterDataType();
	//输出图像大小、波段数、数据类型
	std::cout << "图像大小为(" << yNum << "," << xNum << ")\n" << "数据个数为：" << xNum * yNum * *bandNum << "\n波段数为：" << *bandNum << "\n";
	switch (dataType_i) {
	case GDType::Unknown:
		printf("该影像数据类型为GDT_Unknown\n\n"); break;
	case GDType::Byte:
		printf("该影像数据类型为GDT_Byte，对应unsigned char类型\n\n"); break;
	case GDType::UInt16:
		printf("该影像数据类型为GDT_UInt16，对应unsigned short类型\n\n"); break;
		/*有时间再补充*/
	default:
		break;
	}
	//初始化矩阵大小
	const int band = *bandNum;
	MatrixXd* imgData = new MatrixXd[band];
	int num = 0, max = 0;
	for (int i = 0; i < band; i++) {
		outData = new T[xNum * yNum]{ 0 };
		imgData[i] = MatrixXd::Zero(yNum, xNum);
		GDALRasterBand* pathBand_i = pathDataset->GetRasterBand(i + 1);
		pathBand_i->RasterIO(GF_Read, 0, 0, xNum, yNum, outData, xNum, yNum, dataType_i, 0, 0);
		for (int j = 0; j < yNum; j++) {
			for (int k = 0; k < xNum; k++) {
				imgData[i](j, k) = outData[j * xNum + k];
				//if (imgData[i](j, k) == 0 || imgData[i](j, k) == NULL) {
				//	num++;
				//}
				//if (imgData[i](j, k) > max) {
				//	max = imgData[i](j, k);
				//}
			}
		}
		cout << "已成功读取第  " << i + 1 << "  个波段\n\n";
	}
	//printf("有%d个值为0的像素，像素的最大值为：%d\n", num, max);
	GDALClose(pathDataset);
	return imgData;
	delete[]imgData;
}
```

##### 1.2 存储图像为tif格式

这里的第一个参数RGB不一定需要三个参数，只是函数名

```
/*将图像存到tif文件中*/
template <typename T>
void saveImgData(MatrixXd* RGB, int bandNum,T* outUData,GDALDataType datatype,int* bandMap ,const char* path) {
	int width = RGB[0].cols();
	int height = RGB[0].rows();
	//将矩阵值转到一维矩阵
	outUData = new T[width * height * bandNum];
	for (int band = 0; band < bandNum; band++) {
		for (int i = 0; i < height; i++) {
			for (int j = 0; j < width; j++) {
				outUData[band * height * width + i * width + j] = (T)RGB[band](i, j);
			}
		}
	}

	//获取驱动
	const char* pszFormat = "GTiff";
	GDALDriver* poDriver = GetGDALDriverManager()->GetDriverByName(pszFormat);
	if (!poDriver)
	{
		fprintf(stderr, "get driver by name failed\n");
		exit(0);
	}
	//创建保存影像数据集
	char** papszOptions = nullptr;
	papszOptions = CSLSetNameValue(papszOptions, "INTERLEAVE", "BAND");
	const char* pszDstFilename = path;
	GDALDataset* poDstDS = poDriver->Create(pszDstFilename, width, height, bandNum, datatype, papszOptions);
	if (!poDstDS)
	{
		std::cout << "create Nosie_result.tif failed" << std::endl;
		exit(0);
	}
	poDstDS->RasterIO(GF_Write, 0, 0, width, height, outUData, width, height, datatype, bandNum, bandMap, 0, 0, 0);
}
```



#### 二、几何影像匹配

##### 2.1读取参考影像和待校正影像同名点

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

##### 2.2 将数据存到txt文件 

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

##### 2.3 从txt文件中读取同名点

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

##### 

##### 2.4 从待配准影像计算配准后影像的四个角点

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

##### 2.5计算两个模板的NCC值

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

#### 三、影像融合

##### 3.1 PCA融合

```c
#define ColorLen 255

/*将数组转为主成分矩阵*/
MatrixXd transMatrix(MatrixXd* matrix,int band) {
	int Num = matrix[0].rows() * matrix[0].cols();
	MatrixXd outMatrix = MatrixXd::Zero(Num, band);
	for (int i = 0; i < band; i++) {
		for (int j = 0; j < matrix[0].rows(); j++) {
			for (int k = 0; k < matrix[0].cols(); k++) {
				outMatrix(j * matrix[0].cols() + k, i) = matrix[i](j, k);
			}
		}
	}
	return outMatrix;
}


/*样本中心化 将样本均值从mean变到0*/
void centralization(MatrixXd*& matrix) {
	MatrixXd meanval = (*matrix).colwise().mean();
	//std::cout << meanval<<std::endl;
	RowVectorXd meanvecRow = meanval;
	//样本均值化为0
	(*matrix).rowwise() -= meanvecRow;
}

/*计算样本的协方差矩阵*/
MatrixXd computeCov(MatrixXd* X,int band)
{	
	//协方差矩阵C = XTX / n-1;
	MatrixXd C(band, band);
	C = (*X).adjoint() * (*X);
	C = C.array() / ((*X).rows() - 1);
	return C;
}

/*计算特征值和特征向量*/
void computeEig(MatrixXd& cov, MatrixXd& vec, MatrixXd& val)
{
	//使用selfadjont按照对阵矩阵的算法去计算，可以让产生的vec和val按照有序排列（默认从大到小）
	/*自动排序的特征向量*/
	//SelfAdjointEigenSolver<MatrixXd> eig(cov);
	//vec = eig.eigenvectors();//特征向量
	//val = eig.eigenvalues();//特征值
	/*非自动排序*/
	EigenSolver<MatrixXd> eig(cov); 
	vec = eig.pseudoEigenvectors();
	val = eig.pseudoEigenvalueMatrix();
}

/*主成分变换*/
MatrixXd RGB2PCA(MatrixXd* matrix, MatrixXd vec) {
	//Matrix3d Vec;
	//Vec.col(0) = vec.inverse().col(0);
	//Vec.col(1) = vec.inverse().col(1);
	//Vec.col(2) = vec.inverse().col(2);
	 MatrixXd result = (*matrix) * vec;
	return result;
}

/*使用高分影像替换主成分*/
MatrixXd subsitutionPCA(MatrixXd* pca, MatrixXd* grayData, MatrixXd vec) {
	//(*grayData) = (*grayData) * vec(0,0);
	MatrixXd PCA_Data = MatrixXd::Zero((*grayData).rows(), (*pca).cols());
	PCA_Data.col(0) = (*grayData);//替换最大的主成分
	//PCA_Data.col(0) = (*pca).col(0);
	PCA_Data.col(1) = (*pca).col(1);
	PCA_Data.col(2) = (*pca).col(2);
	return PCA_Data;
}

/*线性拉伸*/
void linearStretch(MatrixXd*& PCA_Data) {
	//统计极值
	double* maxNum = new double[(*PCA_Data).cols()]{ 0 };
	double* minNum = new double[(*PCA_Data).cols()]{ 1000 };
	for (int i = 0; i < (*PCA_Data).cols(); i++) {
		maxNum[i] = (*PCA_Data).col(i).maxCoeff();
		minNum[i] = (*PCA_Data).col(i).minCoeff();
	}
	for (int i = 0; i < (*PCA_Data).cols(); i++) {
		if ((maxNum[i] - minNum[i]) < ColorLen) {
			(*PCA_Data).col(i) = (*PCA_Data).col(i).array() - minNum[i];
		}
		else {
			(*PCA_Data).col(i) = ((*PCA_Data).col(i).array() - minNum[i]) * ColorLen / (maxNum[i] - minNum[i]);
		}
	}
}

/*逆变换*/
MatrixXd PCA2RGB(MatrixXd* pcaData, MatrixXd vec) {
	//求特征向量的逆矩阵
	Matrix3d reVec = vec.inverse();
	//reVec.col(0) = vec.inverse().col(0);
	//reVec.col(1) = vec.inverse().col(1);
	//reVec.col(2) = vec.inverse().col(2);
	MatrixXd rgbData = (*pcaData) * reVec;
	return rgbData;
}
```

##### 3.2 HSI融合

```c
#define PI acos(-1)

/*将三波段RGB图像转到HSI图像*/
MatrixXd* RGB2HSI(MatrixXd* imageData) {
	/*使用几何推导法计算HSI*/
	MatrixXd* HSI = new MatrixXd[3];
	HSI[0] = MatrixXd::Zero(imageData[0].rows(), imageData[0].cols());
	HSI[1] = MatrixXd::Zero(imageData[0].rows(), imageData[0].cols());
	HSI[2] = MatrixXd::Zero(imageData[0].rows(), imageData[0].cols());
	for (int i = 0; i < imageData[0].rows(); i++) {
		for (int j = 0; j < imageData[0].cols(); j++) {
			double R = imageData[0](i, j);
			double G = imageData[1](i, j);
			double B = imageData[2](i, j);
			double theta = acos((R - B + R - G) / (2 * sqrt((R - G) * (R - G) + (R - B) * (G - B))));
			if (G >= B) {
				HSI[0](i, j) = theta;
			}
			else {
				HSI[0](i, j) = 2 * PI - theta;
			}
			double min = (R < G) ? R : G;
			min = (min < B) ? min : B;
			HSI[1](i, j) = 1 - 3 * min / (R + G + B);
			HSI[2](i, j) = (R + G + B) / 3;
		}
	}
	return HSI;
}

/*将HSI图像转到RGB图像(H为弧度)*/
MatrixXd* HSI2RGB(MatrixXd* HSI) {
	int width = HSI[0].cols();
	int height = HSI[0].rows();
	MatrixXd* RGB = new MatrixXd[3];
	RGB[0] = MatrixXd::Zero(height, width);
	RGB[1] = MatrixXd::Zero(height, width);
	RGB[2] = MatrixXd::Zero(height, width);
	for (int i = 0; i < height; i++) {
		for (int j = 0; j < width; j++) {
			double H = HSI[0](i, j);
			double S = HSI[1](i, j);
			double I = HSI[2](i, j);
			if (H >= 0 && H < 2 * PI / 3) {
				RGB[0](i, j) = I * (1 + S * cos(H) / cos(PI / 3 - H)) + 0.5;
				RGB[2](i, j) = I * (1 - S) + 0.5;
				RGB[1](i, j) = 3 * I - (RGB[0](i, j) + RGB[2](i, j)) + 0.5;
			}
			if (H >= 2 * PI / 3 && H < H < 4 * PI / 3) {
				H = H - 2 * PI / 3;
				RGB[1](i, j) = I * (1 + S * cos(H) / cos(PI / 3 - H)) + 0.5;
				RGB[0](i, j) = I * (1 - S) + 0.5;
				RGB[2](i, j) = 3 * I - (RGB[0](i, j) + RGB[1](i, j)) + 0.5;
			}if (H >= 4 * PI / 3 && H <= 2 * PI) {
				H = H - 4 * PI / 3;
				RGB[2](i, j) = I * (1 + S * cos(H) / cos(PI / 3 - H)) + 0.5;
				RGB[1](i, j) = I * (1 - S) + 0.5;
				RGB[0](i, j) = 3 * I - (RGB[2](i, j) + RGB[1](i, j)) + 0.5;
			}

		}
	}
	return RGB;
}

/*从Int16转到Int8*/
MatrixXd* UInt2Byte(MatrixXd* imageData, int bandNum) {
	MatrixXd* imageData8 = new MatrixXd[bandNum];//创建返回矩阵的存储空间
	for (int band = 0; band < bandNum; band++) {
		printf("正在处理第  %d  个波段\n", band + 1);
		//定义16位影像像素的最大值和最小值
		double pMax = imageData[band].maxCoeff();//找到该波段最大值
		double pMin = imageData[band].minCoeff();//找到该波段最小值
		cout << pMax << "\t" << pMin << endl;
		//创建和统计像素直方图
		double* num_pix = new double[pMax + 1]{ 0 };
		for (int i = 0; i < imageData[band].rows(); i++) {
			for (int j = 0; j < imageData[band].cols(); j++) {
				int pixel = imageData[band](i, j);
				num_pix[pixel] += 1;//记录每个像素的个数
			}
		}

		//计算各像素出现概率
		double* pix_val = new double[pMax + 1]{ 0 };
		for (int i = 0; i < pMax; i++) {
			pix_val[i] = num_pix[i] / (imageData[0].cols() * imageData[0].rows());
		}
		//计算累计频率值
		double* accumlt_pix_val = new double[pMax + 1]{ 0 };
		for (int i = 0; i < pMax; i++) {
			for (int j = 0; j < i; j++) {
				accumlt_pix_val[i] += pix_val[j];
			}
		}
		//计算像素两端阶段值
		int minVal = 0, maxVal = 0;
		for (int i = 1; i < pMax; i++) {
			double pixelVal0 = pix_val[0];
			double pixelVal = pix_val[i];
			if ((pixelVal - pixelVal0) > 0.002) {
				minVal = i; break;
			}
		}
		for (int i = pMax - 1; i > 0; i--) {
			double pixelValmax = pix_val[(int)pMax];
			double pixelVal = pix_val[i];
			if (pixelVal < (pixelValmax - 0.0001)) {
				maxVal = i; break;
			}
		}
		//拉伸
		int min = 255, max = 0, num = 0;
		imageData8[band] = MatrixXd::Zero(imageData[band].rows(), imageData[band].cols());//初始化该层数组大小
		for (int i = 0; i < imageData[band].rows(); i++) {
			for (int j = 0; j < imageData[band].cols(); j++) {
				double pixelData16 = imageData[band](i, j);//获取原始数据
				double pixelData8 = (pixelData16 - minVal) / (double(maxVal) - minVal);
				if (pixelData16 < minVal) {
					imageData8[band](i, j) = int(pixelData16 * (20.0 / minVal));
				}
				else if (pixelData16 > maxVal) {
					pixelData8 = (pixelData16 - pMin) / (pMax - pMin);
					imageData8[band](i, j) = int(pow(pixelData8, 0.7) * 255);
					//imageData8[band](i, j) = 254;
				}
				else {
					imageData8[band](i, j) = int(pow(pixelData8, 0.7) * 255);
				}
				if (imageData8[band](i, j) > max) {
					max = imageData8[band](i, j);
				}
				if (imageData8[band](i, j) < min) {
					min = imageData8[band](i, j);
				}
				if (imageData8[band](i, j) == 0 || imageData8[band](i, j) == NULL) {
					num++;
				}
			}
		}
		printf("处理后的第%d波段的最小像素值为：%d，最大像素值为：%d，并且有  %d  个像素值为0的像素\n\n", band + 1, min, max, num);
	}
	return imageData8;
}
```

#### 四、影像变化检测

##### 4.1 CVA变化检测

```c
/*计算两影像的光谱变化量*/
MatrixXd getDiference(MatrixXd* time1, MatrixXd* time2, int bandNum) {
	int width = time1[0].cols();
	int height = time1[0].rows();
	MatrixXd defeMat = MatrixXd::Zero(height, width);
	for (int i = 0; i < bandNum; i++) {
		MatrixXd matrix0;
		matrix0 = (time1[i].array() - time2[i].array()).array().square();
		defeMat += matrix0;
	}
	defeMat = defeMat.cwiseSqrt();
	return defeMat;
}

```

##### 4.2 自适应阈值（OSTU）

```c
#define MAXSIZE 255
#define BEGIN 250

/*统计灰度直方图*/
double* grayHistogram(MatrixXd defeMat) {
	double* gray = new double[MAXSIZE] {0};
	for (int m = 0; m < defeMat.rows(); m++) {
		for (int n = 0; n < defeMat.cols(); n++) {
			int count = int(defeMat(m, n)) - 1;
			if (count < 0) {
				gray[0] += 1;
			}
			else if (count > MAXSIZE - 1) {
				gray[MAXSIZE - 1] += 1;
			}
			else {
				gray[count] += 1;
			}
		}
	}
	return gray;
}

/*OSTU方法查找自适应阈值*/
double getLimitInOSTU(MatrixXd matrix) {
	int sizeImg = matrix.rows() * matrix.cols();
	double limit = 0;
	VectorXd RMSE = VectorXd::Zero(MAXSIZE);
	double* gray = new double[MAXSIZE] {0};
	gray = grayHistogram(matrix);
	cout << gray[1] << "\t" << gray[100] << endl;
	for (int i = BEGIN; i < MAXSIZE; i++) {
		double num_0 = (matrix.array() <= i).count();
		double num_1 = (matrix.array() > i).count();
		double sum_min = 0;
		double sum_max = 0;
		for (int j = 0; j < MAXSIZE; j++) {
			if (j <= i) {
				sum_min += (j + 1) * gray[j];
			}
			if (j > i) {
				sum_max += (j + 1) * gray[j];
			}
			double index = (i * MAXSIZE + j)*100 / (MAXSIZE * MAXSIZE);
			printf("%.f%%\b\b\b\b", index);
		}
		double mean_0 = sum_min / sizeImg;
		double mean_1 = sum_max / sizeImg;
		double w0 = num_0 / sizeImg;
		double w1 = num_1 / sizeImg;
		RMSE(i) = w0 * w1 * (mean_0 - mean_1) * (mean_0 - mean_1);
	}
	RMSE.maxCoeff(&limit);
	printf("方差最大值为：%f，自适应阈值为：%f\n", RMSE.maxCoeff(), limit);
	return limit;
}

/*中值滤波*/
MatrixXd medianFilter(MatrixXd matrix) {
	Matrix3d blockMat;
	double median = 0;
	MatrixXd newMat = matrix;
	for (int i = 0; i < matrix.rows()-3; i++) {
		for (int j =0; j < matrix.cols()-3; j++) {
			blockMat = matrix.block(i, j, 3, 3);
			if ((blockMat.array() > 0).count() > 4) {
				median = 255;
			}
			else {
				median = 0;
			}
			newMat(i + 1, j + 1) = median;
		}
	}
	return newMat;
}
```

##### 4.3 马尔可夫随机去噪

通过修改tempsize可以修改马尔可夫阶数，但并不是递增的阶数变化

```c
#define tempSize 2

MatrixXd denoising_MRF(MatrixXd matrix) {
	int width = matrix.cols();
	int height = matrix.rows();
	//将等于255的位置值赋为1,等于0的赋值为-1
	MatrixXd y = (matrix.array() != 0).cast<double>() - (matrix.array() == 0).cast<double>();
	MatrixXd x = y;
	double h = 0.1, beta = 3.2, eta = 0.1;
	int num = 0;
	while (num < 10) {
		int index = 0;//记录变化的像素数目，不变时退出循环
		for (int i = tempSize; i < height - tempSize; i++) {
			for (int j = tempSize; j < width - tempSize; j++) {
				double temp = x(i, j);
				double rela = 0;
				for (int m = -tempSize; m <= tempSize; m++) {
					for (int n = -tempSize; n <= tempSize; n++) {
						if (m != 0 && n != 0) {
							rela = rela + beta / sqrt(m * m + n * n) * x(i - m, j - n);
							//rela = rela + beta * x(i - m, j - n);
						}
					}
				}
				x(i, j) = -1;
				//double  E1 = h * x(i, j) - beta * x(i, j) * (x(i - 1, j) + x(i + 1, j) + x(i, j - 1) + x(i, j + 1)) - eta * x(i, j) * y(i, j);
				double E1 = h * x(i, j) - x(i, j) * rela - eta * x(i, j) * y(i, j);
				x(i, j) = 1;
				//double E2 = h * x(i, j) - beta * x(i, j) * (x(i - 1, j) + x(i + 1, j) + x(i, j - 1) + x(i, j + 1)) - eta * x(i, j) * y(i, j);
				double E2 = h * x(i, j) - x(i, j) * rela - eta * x(i, j) * y(i, j);
				if (E1 < E2) {
					x(i, j) = -1;
				}
				else {
					x(i, j) = 1;
				}
				if (temp != x(i, j)) {
					index++;
				}
			}
		}
		num++;
		printf("--->已完成第 %d 次循环,有 %d 个像素点像素值改变\n", num, index);
		if (index < 1)
			break;
	}
	MatrixXd trueImg = (x.array() != -1).cast<double>().array() * 255;
	return trueImg;
}
```

##### 4.4 变化检测精度评价

```c
double* GetConfusionMatrix(MatrixXd myImg, MatrixXd refeImg) {
	int width = myImg.cols();
	int height = myImg.rows();
	double* confusionMatrix = new double[4]{ 0 };
	MatrixXd addMatrix = myImg + refeImg;
	MatrixXd subMatrix = refeImg - myImg;
	confusionMatrix[0] = (addMatrix.array() == 510).count();
	confusionMatrix[3] = (addMatrix.array() == 0).count();
	confusionMatrix[1] = (subMatrix.array() == -255).count();
	confusionMatrix[2] = (subMatrix.array() == 255).count();
	return confusionMatrix;
}

double* GetAccuracyEvaluation(double* confusionMatrix) {
	double* accuracy = new double[4];
	//漏检率
	accuracy[0] = confusionMatrix[2] / (confusionMatrix[2] + confusionMatrix[0]);
	//虚警率
	accuracy[1] = confusionMatrix[1] / (confusionMatrix[0] + confusionMatrix[1]);
	//总分类精度
	accuracy[2] = (confusionMatrix[0] + confusionMatrix[3]) / (confusionMatrix[0] + confusionMatrix[1] + confusionMatrix[2] + confusionMatrix[3]);
	double num0 = ((confusionMatrix[0] + confusionMatrix[1] + confusionMatrix[2] + confusionMatrix[3]) * (confusionMatrix[0] + confusionMatrix[3]));
	double num1 = (confusionMatrix[0] * confusionMatrix[2] + confusionMatrix[1] * confusionMatrix[3]);
	double num2 = pow((confusionMatrix[0] + confusionMatrix[1] + confusionMatrix[2] + confusionMatrix[3]), 2);
	//kappa系数
	accuracy[3] = (num0 - num1) / (num2 - num1);
	return accuracy;
}

int writeAccuracy(const char* path, double* accuracy,const char* imgPath) {
	string name[4] = { "漏检率","虚警率","总分类精度","kappa系数" };
	FILE* fpWrite = fopen(path, "a");
	if (fpWrite == NULL){
		return 0;
	}
	fprintf(fpWrite, imgPath);
	fprintf(fpWrite, "\n");
	fprintf(fpWrite, "漏检率：    %f\n", accuracy[0]);
	fprintf(fpWrite, "虚警率：    %f\n", accuracy[1]);
	fprintf(fpWrite, "总分类精度：%f\n", accuracy[2]);
	fprintf(fpWrite, "kappa系数： %f\n", accuracy[3]);
	fclose(fpWrite);
}
```

#### 五、K-Means 非监督分类

##### 5.1 不调用其他内嵌函数

```c
/*将Eigen矩阵转为double矩阵*/
double* Eigen2Double(MatrixXd matrix) {
	int width = matrix.cols();
	int height = matrix.rows();
	double* data = new double[width * height];
	for (int i = 0; i < height; i++) {
		for (int j = 0; j < width; j++) {
			data[i * width + j] = matrix(i, j);
		}
	}
	return data;
}

/*将double矩阵转为Eigen矩阵*/
MatrixXd Double2Eigen(double* data,int width,int height) {
	MatrixXd matrix;
	for (int i = 0; i < height; i++) {
		for (int j = 0; j < width; j++) {
			matrix(i, j) = data[i * width + j];
		}
	}
	return matrix;
}

/*K-means聚类*/
MatrixXd clustering_GDAL(MatrixXd matrix, int ClusterNum) {
	/*存储原始矩阵大小*/
	int width = matrix.cols();
	int height = matrix.rows();
	int dataNum = width * height;
	/*获取矩阵内最小像素和最大像素*/
	int Size = 255 / (ClusterNum - 1);
	int minPixel = matrix.minCoeff();
	int maxPixel = matrix.maxCoeff();
	double* data = Eigen2Double(matrix);
	MatrixXd ClusterMatrix;
	MatrixXd locationMatrix = MatrixXd::Zero(dataNum, ClusterNum);
	VectorXd colorMat = VectorXd::Zero(ClusterNum);
	/*初始化聚类中心*/
	double* p0 = new double[ClusterNum];
	for (int i = 0; i < ClusterNum; i++) {
		p0[i] = minPixel+20 + i * 20;
		colorMat(i) = Size * i;
	}
	while (true) {
		/*初始化矩阵*/
		ClusterMatrix = MatrixXd::Zero(dataNum, ClusterNum);
		double* sum = new double[ClusterNum] {0};
		int* num = new int[ClusterNum] {0};
		for (int i = 0; i < dataNum; i++) {
			for (int j = 0; j < ClusterNum; j++) {
				ClusterMatrix(i, j) = abs(p0[j] - data[i]);
			}
			int r = -1;
			int c = -1;
			ClusterMatrix.row(i).minCoeff(&r, &c);
			sum[c] = sum[c] + data[i];
			num[c] = num[c] + 1;
			locationMatrix(i, c) = 1;
		}
		double* p1 = new double[ClusterNum];
		int change = 0;
		for (int i = 0; i < ClusterNum; i++) {	
			p1[i] = sum[i] / num[i];
			if (abs(p1[i] - p0[i]) > 0.02) {
				cout << p1[i] << "\t" << p0[i]<<"\t";
				p0[i] = p1[i];
				change++;				
			}
		}
		cout << "\t" << change<<endl;
		cout << p0[0] << "\t" << p0[1] << endl;
		cout << "\n-------------------------------------------------------------\n" << endl;
		if (change == 0){
			cout << "Center:\n";
			for (int i = 0; i < ClusterNum; i++) {
				cout << "\t" << p0[i];
			}
			cout << endl;
			break;
		}
		
	}
	VectorXd vector = locationMatrix * colorMat;
	MatrixXd outMatrix = MatrixXd::Zero(height, width);
	for (int i = 0; i < height; i++) {
		for (int j = 0; j < width; j++) {
			outMatrix(i, j) = vector(i * width + j);
		}
	}
	return outMatrix;
}
```

##### 5.2 使用OpenCV内嵌函数

```c
Mat clustering_OpenCv(Mat src,int ClusterNum)
{
	int row = src.rows;
	int col = src.cols;
	unsigned long int size = row * col;

	Mat clusters(size, 1, CV_32SC1);	//clustering Mat, save class label at every location;

	//convert src Mat to sample srcPoint.
	Mat srcPoint(size, 1, CV_32FC3);

	Vec3f* srcPoint_p = (Vec3f*)srcPoint.data;//
	Vec3f* src_p = (Vec3f*)src.data;
	unsigned long int i;

	for (i = 0; i < size; i++)
	{
		*srcPoint_p = *src_p;
		srcPoint_p++;
		src_p++;
	}
	Mat center(ClusterNum, 1, CV_32FC3);
	double compactness;//compactness to measure the clustering center dist sum by different flag
	compactness = kmeans(srcPoint, ClusterNum, clusters,
		TermCriteria(TermCriteria::COUNT + TermCriteria::EPS, 10, 0.1), ClusterNum,
		KMEANS_PP_CENTERS, center);

	cout << "center row:" << center.rows << " col:" << center.cols << endl;
	for (int y = 0; y < center.rows; y++)
	{
		Vec3f* imgData = center.ptr<Vec3f>(y);
		for (int x = 0; x < center.cols; x++)
		{
			cout << imgData[x].val[0] << " " << imgData[x].val[1] << " " << imgData[x].val[2] << endl;
		}
		cout << endl;
	}


	double minH, maxH;
	minMaxLoc(clusters, &minH, &maxH);			//remember must use "&"
	cout << "H-channel min:" << minH << " max:" << maxH << endl;

	int* clusters_p = (int*)clusters.data;
	//show label mat
	Mat label(src.size(), CV_32SC1);
	int* label_p = (int*)label.data;
	//assign the clusters to Mat label
	for (i = 0; i < size; i++)
	{
		*label_p = *clusters_p;
		label_p++;
		clusters_p++;
	}

	Mat label_show;
	label.convertTo(label_show, CV_8UC1);
	normalize(label_show, label_show, 255, 0, NORM_MINMAX);
	imshow("label", label_show);



	map<int, int> count;		//map<id,num>
	map<int, Vec3f> avg;		//map<id,color>

	//compute average color value of one label
	for (int y = 0; y < row; y++)
	{
		const Vec3f* imgData = src.ptr<Vec3f>(y);
		int* idx = label.ptr<int>(y);
		for (int x = 0; x < col; x++)
		{

			avg[idx[x]] += imgData[x];
			count[idx[x]] ++;
		}
	}
	//output the average value (clustering center)
	//计算所得的聚类中心与kmean函数中center的第一列一致，
	//以后可以省去后面这些繁复的计算，直接利用center,
	//但是仍然不理解center的除第一列以外的其他列所代表的意思
	for (i = 0; i < ClusterNum; i++)
	{
		avg[i] /= count[i];
		if (avg[i].val[0] > 0 && avg[i].val[1] > 0 && avg[i].val[2] > 0)
		{
			cout << i << ": " << avg[i].val[0] << " " << avg[i].val[1] << " " << avg[i].val[2] << " count:" << count[i] << endl;

		}
	}
	//show the clustering img;
	Mat showImg(src.size(), CV_32FC3);
	for (int y = 0; y < row; y++)
	{
		Vec3f* imgData = showImg.ptr<Vec3f>(y);
		int* idx = label.ptr<int>(y);
		for (int x = 0; x < col; x++)
		{
			int id = idx[x];
			imgData[x].val[0] = avg[id].val[0];
			imgData[x].val[1] = avg[id].val[1];
			imgData[x].val[2] = avg[id].val[2];
		}
	}
	normalize(showImg, showImg, 1, 0, NORM_MINMAX);
	imshow("show", showImg);
	waitKey();
	return label;
}
```