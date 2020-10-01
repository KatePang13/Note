# python 离线部署 oletools 

本文以 oletools 为例，记录如何在“老破小”的机器上离线部署一个python应用。



## Why? 为什么要离线部署

yum/ apt-get 用起来多爽呀，pip 用起来多爽呀，干嘛要折腾离线部署
当在一个生产环境很糟糕的情况下，yum, apt-get 可能都失效了， pip 远程拉取也失效了... 生活太艰难了



## Python 离线部署

### 准备
https://www.python.org/ 上选择合适的版本

这里使用 https://www.python.org/ftp/python/2.7/Python-2.7.tgz

````
wget https://www.python.org/ftp/python/2.7/Python-2.7.tgz

```


### 安装

```shell
tar -zxvf Python-2.7.tgz
cd Python-2.7/
make && make install


py27=/usr/local/bin/python2.7
py=/usr/bin/python

if [ ! -f $py27 ]; then
	ReportError "Failed to install python2.7"
	exit 1
fi

rm -rf $py
ln -s $py27 $py


#检查安装
python

#检查yum是否正常
yum
```

### 注意事项

1. 如何出现python对底层库的依赖问题，如果幸运的话，yum 安装相应的依赖，不然得搜索相应的源码包编译安装
2. yum 对 /usr/bin/python 有依赖，需要将 yum 脚本的第一行，改成原来的python版本，比如/usr/bin/python2.6




## setuptools 离线部署

### 准备

 https://pypi.org/

https://files.pythonhosted.org/packages/b0/f3/44da7482ac6da3f36f68e253cb04de37365b3dba9036a3c70773b778b485/setuptools-44.0.0.zip

### 安装

```shell
unzip setuptools-44.0.0.zip
cd setuptools-44.0.0
python  setup.py install
```



### 注意事项

setuptools-44.xx 以后，放弃对 python 2的支持，注意选择好版本



## pip 离线部署

### 准备

https://pypi.org/

https://files.pythonhosted.org/packages/5a/4a/39400ff9b36e719bdf8f31c99fe1fa7842a42fa77432e584f707a5080063/pip-20.2.2-py2.py3-none-any.whl

### 安装

```shell
python pip-20.2.2-py2.py3-none-any.whl/pip install --no-index pip-20.2.2-py2.py3-none-any.whl
echo "export PATH=/usr/local/bin:$PATH" >> /etc/profile
source /etc/profile

#检查安装
pip -V
```



## 应用服务离线部署

这里安装的是 [oletools](https://github.com/decalage2/oletools) ，一个用以探测 文件中恶意vba宏 的python工具，我这边应用在 恶意宏相关的垃圾邮件检查。

https://github.com/decalage2/oletools/wiki 常规的安装方法很方便，可惜条件不允许，哈哈。

### 准备

在打包机器上准备离线安装所需的材料

```shell
#获取资源
wget https://github.com/decalage2/oletools/archive/master.zip
unzip master
cd oletools-master

#根据 requirements.txt 下载依赖的离线安装包到指定路径
mkdir deps
pip download -r requirements.txt -d  deps

#将oletools 与 依赖一起打包压缩
tar -zxvf ../oletools.tgz ./*
```



### 安装

将打包了依赖的 **oletools.tgz** 拷贝到需要离线部署的机器上

```shell
#解压
tar -zxf oletools.tgz
cd oletools-master

#安装,从指定的本地路径获取依赖，而不是pip仓库远程拉取
pip install -r requirements.txt --no-index --find-links deps

#检查安装
mraptor test.doc
```




## 总结
部署一个小小的工具，这么折腾，这么多坑，充分凸显出了容器部署的优越性，任何技术的出现总是有它必然的原因。

