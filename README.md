# rasa ARM64编译过程

> * \*~~~~国内用户请科学上网~~

###前言：

此次编译是在[Rasa](https://rasa.com/) v2.7.0版本的基础上进行的，
为了适配armv8架构的CPU，我们需要对rasa进行一些修改，
这次编译的难点在于，国内网络环境下，要反复切换不同的网络模式， 网络模式的切换也会影响到编译速度，
所以我们需要对rasa进行一些修改，以便在国内环境下， 可以反复切换不同的网络模式，从而提高编译速度。
还有一点需要注意的是，底层依赖的包，如果安装不成功需要重新编译单独的包，有时候在网上找不到现成，编译好的包。

主要修改如下：


### 1> 下载conda环境，下载适合ARM64的安装文件

> https://github.com/conda-forge/miniforge

### 2> 创建conda环境 ,指定python版本

创建conda环境，并且指定python版本为3.7

`conda create -n python37 python=3.7`

`conda activate python37`

### 3> 编译rasa需要条件为三部分：

#### 1.poetry包管理器

此处很重要，一定调整好科学上网，否则，会导致：
1、下载失败
2、下载中断重新安装
3、安装中断

> * \*  curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python
>  source $HOME/.poetry/env （此处可选添加环境变量）



#### 2.pip 依赖
pip依赖分为三个部分：
1、可直接通过poetry包管理器安装,虽然安装过程中可能会断开，但是poetry包管理器可以缓存。

2、不可直接通过poetry包管理器，但是可以通过pip install或者conda install 安装，

3、需要重新编译，或者需要在网上找编译好的包。

numpy,h5py网上都有编译好的包
经过查找，只有psycopg2是需要重新编译

##### 编译psycopg2

> https://www.psycopg.org/docs/install.html#debug-build

可能过程中需要安装jdk8,jdk8后续安装pip依然会用到，

`apt-get install openjdk-8-jdk -y`

如果遇到问题 请尝试安装libpq-dev

`apt-get install rsync`

`apt-get install libpq-dev`                      `

`pip install psycopg2`

此处如果不成功 请下载源代码包进行编译：

`python setup.py build`

`python setup.py install`

##### 3.tensorflow部分

> TensorFlow是一个开源软件库，用于各种感知和语言理解任务的机器学习。目前被50个团队用于研究和生产许多Google商业产品,如语音识别、Gmail、Google 相册和搜索，其中许多产品曾使用过其前任软件DistBelief
> TensorFlow最初由谷歌大脑团队开发，用于Google的研究和生产，于2015年11月9日在Apache 2.0开源许可证下发布。
> -引用自维基百科

tensorflow部分分为三个组件，其中包括tensorflow本体，和插件

tensorflow组件部分的前提是需要安装bazel编译工具

tensorflow版本对应固定版本bazel 此处tensorflow选择版本为2.3.1 bazel版本为3.1.0
###### 1. tensorflow本体

 > https://github.com/Qengineering/TensorFlow-Raspberry-Pi_64-bit
 
 此次本体未编译，找到了编译好的包，自行编译中间遇到底层运行库错误，此处使用了2.3.1版本

如果修改tensorflow版本请在此处修改

![](https://small-e.oss-cn-beijing.aliyuncs.com/1-md.png)


###### 2. tensorflow-addon


tensorflow-addon注意编译版本此处选择的版本为0.12.1，最好不要随便修改版本，版本poetry锁已经定好了。
具体参照了国外的学术网站

> https://qengineering.eu/install-tensorflow-addons-on-raspberry-64-os.html

首先下载tensorflow-addon 0.12.1版本

> https://github.com/tensorflow/addons/tags

###### 3. tensorflow-text

这边尝试过用和tensorflow-addons,同样的方法编译tensorflow-text，但是编译未成功

tensorflow-text目前编译不通过，官方也没有退出对应的arm64安装包 可以从poetry锁中删除，如果要使用请查看github上的issues自行编译

请在poetry.lock和pyproject.toml删除tensorflow-text相关信息

poetry.lock删除信息
![](https://small-e.oss-cn-beijing.aliyuncs.com/2-md.png)

pyproject.toml删除信息
![](https://small-e.oss-cn-beijing.aliyuncs.com/3-md.png)
### 4> rasa源代码安装
>  make install

请注意，此处rasa源代码是程序本身，不能删除，即使pip安装上依然不能删除。

编译结束后如果无法启动可能报如下错误

![](https://small-e.oss-cn-beijing.aliyuncs.com/4-md.jpg)


请导入环境变量

`export LD_PRELOAD=$LD_PRELOAD:/opt/conda/envs/py37/lib/python3.7/site-packages/scikit_learn.libs/libgomp-d22c30c5.so.1.0.0`

### 5> 编译rasa镜像

在docker内部编译rasa镜像，参考的是这篇文章：

> https://github.com/khalo-sa/rasa-apple-silicon

需要注意一下几点

1、docker内部的网络和外部不完全一样，需要注意网络环境

2、镜像封装的大小

接下来说说具体编译的步骤：

###### 1.准备基础镜像

1.condaforge/miniforge3 选择的是conda的镜像

*`docker pull condaforge/miniforge3`

**后续的操作均在镜像内进行**

###### 2.安装依赖
>apt-get install vim -y
apt-get install rsync -y 
apt-get install openjdk-8-jdk -y
apt-get install libpq-dev -y

请注意为了保证镜像重量级 后面一定要把编译过程中无用的依赖删除

###### 3.拷贝预编译文件

在自行编译的过程中，使用的是通过源码封装的镜像的中间件
使用的文件为：
>**addons - whl
 tensorflow - whl
 tensorflow_addons - whl
 h5py - whl 
 numpy - whl
 bazel - 二进制文件
 \*psycopg - 可选
 rasa源码**
 
###### 4.基本编译指令
```
#-------------------------install  bazel--------------------------------------
mv  bazel /usr/bin/bazel
chmod +x bazel
#-------------------------安装依赖--------------------------------------
apt-get install openjdk-8-jdk -y
apt-get install vim
apt-get install rsync -y 
apt-get install libpq-dev
conda create -n python37 python=3.7
conda activate py37
 sudo ln -s /opt/conda/envs/python37/lib/python3.7/site-packages/tensorflow/python/_pywrap_tensorflow_internal.so /usr/lib/lib_pywrap_tensorflow_internal.so
#-------------------------install  poetry--------------------------------------
curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python
source $HOME/.poetry/env 
apt-get install openssl
apt-get install privoxy
apt-get install libpq-dev
#pip安装上诗歌安装不上的包
pip install scikit-learn==0.24.2
pip install mypy==0.8.12
注：rasa编译包不可以删除 就是程序本身
*export LD_PRELOAD=$LD_PRELOAD:/opt/conda/envs/py37/lib/python3.7/site-packages/scikit_learn.libs/libgomp-d22c30c5.so.1.0.0

# rasa run --enable-api && rasa run actions

```
###### 5.镜像封装
 
此指令请在主机上运行：

>  docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
> OPTIONS说明：
-a :提交的镜像作者；
-c :使用Dockerfile指令来创建镜像；
-m :提交时的说明文字；
-p :在commit时，将容器暂停。


可关注“起硕智能科技”公众号，给我们留言
nanoha@tutanota.com
