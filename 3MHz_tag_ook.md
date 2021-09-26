采样率为20Mhz的情况下

用1024个符号计算，每个符号点数为：$6.9$

实际速率为：$1/(6.9 * 1/20)= 2.9MHz$ 



帧同步



求信号部分的均值，减去均值，即去DC

将信号全部集中在实部

信号归一化（1，-1）



送去符号同步



> 为何原来的译码方案不行



> 目前方案的详尽报告





### 2 详尽接收端处理流程

#### 2.1 匹配滤波

- ​	7个点的滑动平均

##### 输出波形

![image-20210926204211959](/Users/wangwen/Library/Application Support/typora-user-images/image-20210926204211959.png)





#### 2.2 DC Block 模块

##### 作用

- 去除直流
- 用信号能量代替信号实部，虚部为0.

##### 实现步骤

1. 计算信号部分的能量的均值：

   $energe\_avg=\left(  \sum_{k=begin}^{k=end}{(x[k].real^2+x[k].imag^2)}\right)/(end-begin)$

2. 将能量集中在实部，对于每一个输入的采样点 $k$ ，输出为：

   $k\_real=(k\_real^2+k\_imag^2)-energe\_avg$

   $k\_imag=0$

##### 输出波形

![image-20210926204328651](/Users/wangwen/Library/Application Support/typora-user-images/image-20210926204328651.png)





#### 2.3 帧同步和AGC

##### 帧同步

- 帧同步使用14位preamble：（这里设计的preamble是下面的前12位，但是第一位会出错，故暂时借用数据位的两个1 ）

  `[1, -1,  1, -1, 1, -1,  1, -1, 1, -1,  1, -1,  1, 1]`

##### AGC

实现思路：

1. 计算信道估计值

2. 解决相位模糊问题（反射信号有时在载波之上，有时在载波之下）

```C
// 1. 信道估计值计算
float _channel_estimation(float * input,int index, int steps)
{
    float h = 0.0;
    for(int i = 0; i < 6; i ++){
        h += input[ index + i * steps*2 ];
    }
    return h / 6.0;
}		
int main(){
  	// index： 帧同步后的帧起始位置
  	// steps： 取值步长即 7 
  	float h_estm = _channel_estimation(input, index, steps);
  	// 2. 防止相位模糊
    if( h_estm > 0 ) { 
        for(int i = 0;  i < input_len; i++){
            input[i] = -input[i];
        }
    }
}		
```

3. 幅度调整 -- AGC

```C
		// 取信道估计值的绝对值
    h_estm = h_estm>0 ? h_estm : -h_estm;
    int out = 0;
    // 3. 幅度调整的目标值
    float agc_ampl_ref = 1.15;
    for(int i = index;  i < input_len; i++)
    {
        output[out++] =  input[i] / h_estm * agc_ampl_ref;  
    }
```

##### 输出波形

- 下面是经过帧同步 和 AGC ==之前==和==以后==：

![image-20210926195202175](/Users/wangwen/Library/Application Support/typora-user-images/image-20210926195202175.png)







#### 2.4 符号同步

初始时的符号同步周期为7

##### 输出波形

经过符号同步**==之前==**和**==之后==**的信号如图：

![image-20210926203110387](/Users/wangwen/Library/Application Support/typora-user-images/image-20210926203110387.png)



#### 2.5 译码

- 直接将符号同步之后的数据 $x[k]$ 用来判决译码：

```C
if x[k] < 0
	bit[k] = 0
else 
  bit[k] = 1
```









> 测试结果
