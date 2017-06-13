---
layout:     post
title:      "Install Dashboard component of Open-Falcon"
subtitle:   ""
date:       2017-06-12 10:16:32
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Open-Falcon
---
上周五在QA机器上安装Open-Falcon的[Dashboard](https://github.com/open-falcon/dashboard)组件失败了，当时没能解决，今天拿过来仔细看一下。

Dashboard是一个Python项目。

问题出在执行这条命令时
```
./env/bin/pip install -r pip_requirements.txt -i https://pypi.douban.com/simple
```
它的意思是使用当前虚拟环境下的pip来install pip_requirements.txt里定义的相关package，`-i`用于指定镜像源，国内一般选择豆瓣[[http://pypi.douban.com/simple/ ](http://pypi.douban.com/simple/ )]或阿里云[[http://mirrors.aliyun.com/pypi/simple](http://mirrors.aliyun.com/pypi/simple)]的源，用Python的官方源pypi.python.org/pypi的话速度不理想。

pip_requirements.txt里的内容很简单
```
Flask==0.10.1
Jinja2==2.7.2
Werkzeug==0.9.4
gunicorn==18.0
python-dateutil==2.2
requests==2.3.0
mysql-python
python-ldap
```
使用这种[requirements files](https://pip.pypa.io/en/latest/user_guide/#requirements-files)的方式要比手动一个个install方便的多。

命令执行后的结果
![pip-install-result](/img/in-post/install-dashboard-component-of-open-falcon/pip-install-result.png)

原因应该是这个
```
Could not fetch URL https://pypi.douban.com/simple/flask/: There was a problem confirming the ssl certificate: hostname 'pypi.doubanio.com' doesn't match '*.ucdl.pp.uc.cn' - skipping
```
`*.ucdl.pp.uc.cn`不就是那个阿里的UC吗？为什么`ssl certificate`是它？奇怪~

上图中还有两个SSL相关的Warning，但是暂时不能通过`upgrade to a newer version of Python to solve this`来避免。不清楚Dashboard项目是否兼容Python2和3。

既然看起来像是SSL的问题，那我先不用它好了。注意现在我只是install Flask来做测试。
```
[root@localhost dashboard]# ./env/bin/pip install Flask -i http://pypi.douban.com/simple
Collecting Flask
  The repository located at pypi.douban.com is not a trusted or secure host and is being ignored. If this repository is available via HTTPS it is recommended to use HTTPS instead, otherwise you may silence this warning and allow it anyways with '--trusted-host pypi.douban.com'.
  Could not find a version that satisfies the requirement Flask (from versions: )
No matching distribution found for Flask

```
还是不行，但是提示信息很明了。Try again！
```
[root@localhost dashboard]# ./env/bin/pip install Flask -i http://pypi.douban.com/simple --trusted-host pypi.douban.com
Collecting Flask
  Downloading http://pypi.doubanio.com/packages/77/32/e3597cb19ffffe724ad4bf0beca4153419918e7fa4ba6a34b04ee4da3371/Flask-0.12.2-py2.py3-none-any.whl (83kB)
...
...
Installing collected packages: click, Flask
Successfully installed Flask-0.12.2 click-6.7
```
Done.

那如果用pip默认的配置来做会是什么结果呢？
```
[root@localhost dashboard]# ./env/bin/pip install Flask
Collecting Flask
...
  SNIMissingWarning
...
  InsecurePlatformWarning
  Downloading Flask-0.12.2-py2.py3-none-any.whl (83kB)
Installing collected packages: Flask
Successfully installed Flask-0.12.2
```
看起来也可以的，就是不知道用的哪个源。无语了，确实有点儿搞不懂。先放在这，看后面能否找到真正的原因。
![cry](/img/cry.png)


[《Pythoh guide - en》](http://docs.python-guide.org/en/latest/)

[《Pythoh guide - zh 》](https://pythonguidecn.readthedocs.io/zh/latest/)

