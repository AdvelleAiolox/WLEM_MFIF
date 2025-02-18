# WLEM_MFIF

## Multi-Focus Image Fusion based on Weighted Laplacian Edge Map 基于拉普拉斯边缘检测图权重分配的多聚焦图像融合



附加的代码是C++的，要用opencv包，不需要opencv_contrib包（因为我Cmake爆炸了装不上（嚎啕大哭））

It is mainly used to deal with the defocus spread effect(DSE), and it runs quite fast (compared to guided filtering, which is slower).

主要为了应付离焦扩散效应，而且跑得飞快（相比之下导向滤波就比较慢了）。

It was an ideal accidentally come up to me while I was testing various other algorithms, you can use it but don't plagiarize with your own name on it.

测试各种算法的时候意外搞出来的，可以拿去用但是别剽窃挂自己名。

The first step is to preprocess the image. The images I used for testing are not from the same camera position, so I first run ORB to adjust the image position and deform them, and then formally perform the fusion. If the images you use don't have this problem, you can ignore this.

首先是对图片预处理，我本身测试用图不是同一个相机位的，所以先跑ORB调整图片位置并变形，然后正式进行融合。如果你测试的图片不需要调整，请直接忽略。

The idea is pretty simple, with only three steps

思路很简单，分三步

### 1.First, detect the focus area of each images 首先进行聚焦区域检测

The specific method is first use the function Laplacian and keep the original edge map L1, then use the function Laplacian again after Gaussian blur and keep the edge map after blur as L2, store it as imgL (L1 has heavier noise so L2 is used), perform mean filtering on the difference of L1 and L2 and save it as imgdiff. 

具体方法为跑第一遍拉普拉斯记L1，然后高斯模糊后再跑一遍拉普拉斯记L2，存起来记为imgL（L1噪声比较重所以取L2），对差值进行均值滤波，记为imgdiff。

```
for (int i = 0; i < n; i++)
{
	img[i].convertTo(imgt, CV_32F, 1.0 / 255);        //归一化彩图 Normalize color img
	imgf.push_back(imgt.clone());						          
	cvtColor(imgt, gimgt, COLOR_BGR2GRAY);            //归一化彩图转灰度图 Change color img into grey img
	gimg.push_back(gimgt.clone());						        
	Laplacian(gimg[i], edge1, CV_32F, 3);             //第一遍拉普拉斯找原图边界 Use the function Laplacian to find the edge on the original img
	GaussianBlur(gimg[i], GBt, Size(7, 7), 5, 5);	  //灰度图模糊 Use Gussian Blur on the grey img
	Laplacian(GBt, edge2, CV_32F, 3);                 //第二遍拉普拉斯找高斯模糊后边界 Use the function Laplacian to find the edge on the img treated by Gussian Blur
	imgL.push_back(edge2.clone());
	absdiff(edge1, edge2, diff);                      //差值为聚焦比重 The difference of L1 and L2 is the weight of the focus
	blur(diff, diff, Size(7, 7));                     //模糊一下融合一下点 Blur to fuse some separate points
	imgdiff.push_back(diff.clone());
}
```

### 2.Then, allocate the weight 然后是权值分配

First, set a value as the limit to differ the two different treatement.

首先设置一个分割权值。

对于存在imgL的值大于分割权值的点，直接取用imgdiff中值最大的图的值，使得边缘保留足够的清晰度。

对于所有imgL的值均小于分割权值的点，需要进行加权叠加以模糊过渡边缘，具体操作为将所有图片的imgdiff乘一个权值（我采用的是值的五次方）并进行累加，然后叠加所有图，每个图的权重为加权后的imgdiff除以累加的和。

```
float pick = 0.9;                                //分割权值
for (int i = 0; i < rows; i++)
{
	flag = false;                                  //标记是否有图的imgL大于分割权值
	for (int j = 0; j < n; j++)
	{
		p.push_back(imgdiff[j].ptr<float>(i));
		pl.push_back(imgL[j].ptr<float>(i));
	}
	for (int j = 0; j < cols; j++)
	{
		nm = 0;
		sum = 0;
		max = *(p[0] + imgdiff[0].channels() * j);
		for (int k = 0; k < n; k++)
		{
			if (*(pl[k] + imgL[k].channels() * j) > pick)
			{
				flag = true;
			}
			if (*(p[k] + imgdiff[k].channels() * j) > max)           //记录最大值及其图片编号
			{
				nm = k;
				max = *(p[k] + imgdiff[k].channels() * j);
			}
			sum = sum + pow(*(p[k] + imgdiff[k].channels() * j), 5); //权值累加
		}
		if (flag)                                                  //设置imgdiff中值最大的图的值权值为1，其余均为0，保留图片边界的清晰度
		{
			for (int k = 0; k < n; k++)
			{
				*(p[k] + imgdiff[k].channels() * j) = 0;
			}
			*(p[nm] + imgdiff[nm].channels() * j) = 1.0;
		}
		else                                                       //分配权重，融合图片非边界部分，使得分割区过渡自然
		{
			for (int k = 0; k < n; k++)
			{
				*(p[k] + imgdiff[k].channels() * j) = pow(*(p[k] + imgdiff[k].channels() * j),5)/sum;
			}
		}
	}
	p.clear();
	pl.clear();
}
```

### 3.Fuse the images 图片融合

Using the weight img(imgdiff) from step 2 to fuse the images, it is not diffcult so I will not post my code here.

套用2中处理过的权重图（imgdiff）进行图片融合，具体代码不贴了，简单的.
