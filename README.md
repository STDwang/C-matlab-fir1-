# C++实现matlab的fir1函数

## 导言

最近在进行Qt开发，涉及大量的matlab转C的工作，其中包括插值滤波等，但遗憾就求滤波系数的函数fir1而言，多数都是直接用的matlab生成的系数进行滤波，很少有用C生成的，有少数用C进行实现也和matlab生成的系数相差甚远，因此这里对matlab生成滤波系数的fir1函数进行了研究并用C++进行了重写，以备进行FIR滤波。
	经对比，结果与matlab的fir1函数生成的滤波系数完全一致。

## 函数需求分析

这里我对工作中调用matlab的fir1()函数的实例为例子进行讲解：
 1. **原matlab程序** ：由256（fir1()函数里进行了+1）阶，截止频率为0.25的低通滤波器（默认使用hamming窗）
 ```javascript
	lbn=255;
	lbf=fir1(lbn,0.25,'low');
```
 2. **fir1（）生成的结果** 
![matlab生成结果图](https://img-blog.csdnimg.cn/2482604f712a4052ab206be4ab061988.png#pic_center)
 3. **分析**
 可以看到，输入参数为阶数n，截止频率Wn，选择低通滤波（也是默认的选择），输出参数为一个一维数组，即滤波参数。 


 4.  **C++版fir1（）输入输出参数详情**：


参数名| 介绍
-------- | -----
n| 阶数，对应matlab的fir1里的阶数n
Wn| 对应matlab的fir1里的阶数Wn，但应注意传进来的数据应存在一个vector的double数组里。
h  | double类型的vector数组，里面存的是滤波器系数
  ```javascript
	vector <double> fir1(int numtaps, vector<double> cutoff)
	/*
		未写排错  检查输入有需要自己进行完善
		原matlab函数fir(n, wn)	【函数默认使用hamming】

		参数输入介绍：
			n：  对应matlab的fir1里的阶数n
			Wn:  对应matlab的fir1里的阶数Wn，但应注意传进
					 来的数据应存在一个vector的double数组里。

		参数输出介绍：
					vector <double>的一个数组，里面存的是长度
					为n的滤波器系数。
	*/;
```
5. **C++版fir1函数生成结果** ：结果表明，生成结果与matlab生成结果一致。
![在这里插入图片描述](https://img-blog.csdnimg.cn/b69f3f230d02466089aa151ea7703dac.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ3ODk4MTk4,size_16,color_FFFFFF,t_70#pic_center=30x30)



## 数学过程
数学小辣鸡的我，希望得到大家的指正

![在这里插入图片描述](https://img-blog.csdnimg.cn/271d673e39e74e0690d655e1e8b15b41.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ3ODk4MTk4,size_16,color_FFFFFF,t_70#pic_center)


## 源码

那么废话不多说，直接上代码
 ```javascript
 #include <stdio.h>
#include <math.h>
#include <vector>
#include <iostream>
using namespace std;
 #define PI acos(-1)
vector<double> sinc(vector<double> x)
{
	vector<double> y;
	for (int i = 0; i < x.size(); i++)
	{
		double temp = PI * x[i];
		if (temp == 0) {
			y.push_back(0.0);
		}
		else {
			y.push_back(sin(temp) / temp);
		}
	}
	return y;
}

vector <double> fir1(int n, vector<double> Wn)
{

	/*
		未写排错  检查输入有需要自己进行完善
		原matlab函数fir(n, wn)	【函数默认使用hamming】

		参数输入介绍：
			n：  对应matlab的fir1里的阶数n
			Wn:  对应matlab的fir1里的阶数Wn，但应注意传进
					 来的数据应存在一个vector的double数组里。

		参数输出介绍：
					vector <double>的一个数组，里面存的是长度
					为n的滤波器系数。
	*/
	
	//在截止点的左端插入0（在右边插入1也行）
	//使得截止点的长度为偶数，并且每一对截止点对应于通带。
	if (Wn.size() == 1 || Wn.size() % 2 != 0) {
		Wn.insert(Wn.begin(), 0.0);
	}

	/*
		‘ bands’是一个二维数组，每行给出一个 passband 的左右边缘。
		（即每2个元素组成一个区间）
	*/
	vector<vector <double>> bands;
	for (int i = 0; i < Wn.size();) {
		vector<double> temp = { Wn[i], Wn[i + 1] };
		bands.push_back(temp);
		i = i + 2;
	}

	// 建立系数
	/*
		m = [0-(n-1)/2,
			 1-(n-1)/2,
			 2-(n-1)/2,
			 ......
			 255-(n-1)/2]
		h = [0,0,0......,0]
	*/
	double alpha = 0.5 * (n - 1);
	vector<double> m;
	vector<double> h;
	for (int i = 0; i < n; i++) {
		m.push_back(i - alpha);
		h.push_back(0);
	}
	/*
		对于一组区间的h计算
		left:	一组区间的左边界
		right:  一组区间的右边界
	*/
	for (int i = 0; i < Wn.size();) {
		double left = Wn[i];
		double right = Wn[i+1];
		vector<double> R_sin, L_sin;
		for (int j = 0; j < m.size(); j++) {
			R_sin.push_back(right * m[j]);
			L_sin.push_back(left * m[j]);
		}
		for (int j = 0; j < R_sin.size(); j++) {
			h[j] += right * sinc(R_sin)[j];
			h[j] -= left * sinc(L_sin)[j];
		}

		i = i + 2;
	}

	// 应用窗口函数，这里和matlab一样
	// 默认使用hamming，要用别的窗可以去matlab查对应窗的公式。
	vector <double> Win;
	for (int i = 0; i < n; i++)
	{
		Win.push_back(0.54 - 0.46*cos(2.0 * PI * i / (n - 1)));	//hamming窗系数计算公式
		h[i] *= Win[i];
	}

	bool scale = TRUE;
	// 如果需要，现在可以处理缩放.
	if (scale) {
		double left = bands[0][0];
		double right = bands[0][1];
		double scale_frequency = 0.0;
		if (left == 0)
			scale_frequency = 0.0;
		else if (right == 1)
			scale_frequency = 1.0;
		else
			scale_frequency = 0.5 * (left + right);

		vector<double> c;
		for (int i = 0; i < m.size(); i++) {
			c.push_back(cos(PI * m[i] * scale_frequency));
		}
		double s = 0.0;
		for (int i = 0; i < h.size(); i++) {
			s += h[i] * c[i];
		}
		for (int i = 0; i < h.size(); i++) {
			h[i] /= s;
		}
	}
	return h;
}
```

## 注意
> 该文章仅个人学习使用，欢迎大家一起交流学习
