---
layout: post
title: 基于Matlab的KNN+PCA语音二分类
date: 2021-05-12 23:06:47 +0800
categories: MATLAB
tag: MATLAB语音识别
---

* content
{:toc}

KNN+PCA实现电力系统正常/故障二分类处理

<!-- ![]({{ '/styles/article-image/20210512230647_1.jpg' | prepend: site.baseurl }}){:height='80%' width='80%'} -->

### PCA简要介绍

PCA也称“主元分析”，主要用于数据降维，类似**最优有损压缩**方法

🌰例子：

对一个二维数据，找到一条直线使所有数据距离线的距离之和最短，该线称为**最佳投影线**，此时该点到原点连成的线段在线上投影的距离“最长”（即到原点的距离一定，到直线距离最短，因此在直线上的投影最长），即可以保留最多的有效信息。

<img src="https://raw.githubusercontent.com/Yumik-xy/blogImage/main/img/20210512233141.png" alt="image-20210512233141030" style="zoom:67%;" />

### KNN简要介绍

KNN称“K邻近算法”，主要用于判据数据属于哪一分类

🌰例子：

 对于一组数据，现将其分成A、B两组。现在有一未知节点，为寻找其组别，算法会寻早其距离最近的K个节点，若K节点中A组节点数>B组节点数，则节点归为A组，反之同理。

<img src="https://raw.githubusercontent.com/Yumik-xy/blogImage/main/img/20210512233808.png" alt="image-20210512233808658" style="zoom:67%;" />

### 识别过程

#### 读入训练集数据和训练集标签

读取训练目录所有的声音数据和标签，由于提供的标签为$[1,2]$分布的值，故这里将其转化为$[-1,1]$分布便于进行二分类判决

```matlab
train_wav_path_list = dir(strcat(train_path,'*.wav')); % 读取所有 wav 文件
train_wav_length = length(train_wav_path_list); % 处理训练集数据
train_selection_wav = zeros(train_wav_length,171); % 训练集保存
train_lable = [1 1 ... 2 2 2 ... 1];
train_lable = (train_lable - 1.5)*2;
```

#### 训练集数据进行预处理

对所有的数据进行特征值选择，并保存在选择后的矩阵中。

```matlab
for i = 1:train_wav_length
    wav = audioread([train_path, num2str(i), '.wav'])'; % 读入数据
    wav = wav(1:max_audio);
    selection_wav = feature_selection(wav); % 特征选择函数
    selection_wav = selection_wav'; % 转置
    train_selection_wav(i,:) = selection_wav(:);
end
```

注意到函数`feature_selection(wav)`即为特征值选择函数。

函数首先对所有数据进行归一化操作，将数据进行0点对齐使得所有数据和为0，这样是为了更好的做PCA运算，归一化公式为
$$
x^*=\frac{x-\mu}{\sigma}
$$
函数如下：

```matlab
function new_data = normalization(data) % 数据归一化函数
    data_mean = mean(data); % 求均值μ
    data_norm = norm(data);
    new_data = (data - data_mean) / data_norm;
end
```

进行归一化后将8000bit语音信号进行分帧，分帧数通过公式
$$
fn=\frac{N-wlen}{inc}/+1
$$
进行计算。取分帧长度为800帧，帧移为400，计算可得需要分成19帧，帧与帧之间通过Hamming码进行连接，分帧函数如下：

```matlab
function f=enframe(data,win,inc)
    nx=length(data);
    nwin=length(win);
    if (nwin == 1)
       len = win;
    else
       len = nwin;
    end
    if (nargin < 3)
       inc = len;
    end
    nf = fix((nx-len+inc)/inc);
    f=zeros(nf,len);
    indf= inc*(0:(nf-1)).';
    inds = (1:len);
    f(:) = data(indf(:,ones(1,len))+inds(ones(nf,1),:));
    if (nwin > 1)
        w = win(:)';
        f = f .* w(ones(nf,1),:);
    end
end
```

分帧后的即对每一帧进行特征值选择。对分帧后数据进行傅里叶变换，计算得到子带能量和总的频带能量作为该帧的特征值，由此可以得到$9\times19$的特征值矩阵，特征值选择函数如下：

```matlab
function selection_data = feature_selection(data) % 特征选择函数
    enframe_data = enframe(data,hamming(800),400); % 声音分成19帧信号
    E = [];
    for i = 1:size(enframe_data,1)
        wav_std = normalization(enframe_data(i,:)); % 数据归一化
        wav_std_fft = abs(fft(wav_std,2048)); % 做2048的fft变换
        % 计算频带能量
        E0 = sum(wav_std_fft(1:1024).^2); % 总能量
        E1 = sum(wav_std_fft(1:120).^2); % 子带能量
        E2 = sum(wav_std_fft(121:250).^2); % 子带能量
        E3 = sum(wav_std_fft(251:360).^2); % 子带能量
        E4 = sum(wav_std_fft(361:480).^2); % 子带能量
        E5 = sum(wav_std_fft(481:600).^2); % 子带能量
        E6 = sum(wav_std_fft(601:730).^2); % 子带能量
        E7 = sum(wav_std_fft(731:850).^2); % 子带能量
        E8 = sum(wav_std_fft(851:1024).^2); % 子带能量    
        E = [E; E0 E1 E2 E3 E4 E5 E6 E7 E8];
    end
    selection_data = E;
end
```

#### 读入测试集数据和测试集标签

测试集的处理方式与训练集完全一致，这里不再过多赘述。若训练集较大，可以将训练得到的特征值变量写入为*.mat*文件进行保存，方便后期调用。

#### 进行KNN邻近算法，求解测试集分类



#### 计算测试集和训练集二者2l-范数距离

#### 取最小的k个值，判断正常/异常组

#### 计算错误率

