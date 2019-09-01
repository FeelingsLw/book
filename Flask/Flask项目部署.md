### 项目部署

#### 项目开发完成

1.测试排除项目存在的BUG
 2.用pip freeze > requirements.txt将当前环境的包导出到requirements.txt文件中，方便在部署的时候安装

#### 服务器上准备

1.安装virtualenv以及virutalenvwrapper。并创建虚拟环境。

```shell
pip3 install virtualenv
pip3 install virtualenvwrapper
vi ~/.bashrc 进入文件中，填入以下两行代码：
  export WORKON_HOME=$HOME/.virtualenvs #设置创建的虚拟环境在哪
  source /usr/local/bin/virtualenvwrapper.sh  #通过which virtualenvwrapper.sh找路径

```

注意上面一句容易Bug,如果你的环境是python3，重新加载后，可能会出现这样类似的错误

```shell
/usr/bin/python: No module named virtualenvwrapper
virtualenvwrapper.sh: There was a problem running the initialization hooks. 
If Python could not import the module virtualenvwrapper.hook_loader,
check that virtualenvwrapper has been installed for
VIRTUALENVWRAPPER_PYTHON=/usr/bin/python and that PATH is
set properly
```

因为脚本会默认使用python2环境，但是virtualenvwrapper装在了python3环境中，所以会有上面的报错

直接将VIRTUALENVWRAPPER_PYTHON默认值修改为$PYTHON_HOME/python3即可

```shell

VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3
if [ "$VIRTUALENVWRAPPER_PYTHON" = "" ] then
    VIRTUALENVWRAPPER_PYTHON="$(command \which python)"
fi
```



2.创建虚拟环境

```shell
mkvirtualenv --python=/usr/bin/python3 flask-env
```

3.安装MySQL服务器和客户端：

```shell
yum install mysql-server mysql-client
yum install libmysqld-dev
```

4.进入虚拟环境中，然后进入到项目所在目录，执行命令：`pip install -r requirements.txt`安装好相应的包。

5.在mysql数据库中，创建相应的数据库。

#### 安装uwsgi

1.通过`pip install uwsgi` 安装 uwsgi (uwsgi必须安装在系统级别的Python环境中，不要安装到虚拟环境中)。
 2.使用命令`uwsgi --http :8000 --wsgi-file programdir/program_name.py -callable app -H 虚拟环境路径。`用uwsgi启动项目，如果能够在浏览器中访问到这个页面，说明uwsgi可以加载项目了。

#### 通过配置文件运行uwsgi

在项目的路径下面，创建一个文件叫做program_name_uwsgi.ini的文件，然后填写以下代码：

```shell
//文件名：program_name_uwsgi.ini
[uwsgi]
master = true  # 主进程
processes = 4     # 内核数
chdir = /srv/flask_demo # 项目的路径
callable = app       # 注意，python文件内的app需要作为全局变量引出，不然会找不到
home = /opt/.virtutalenvs/flask-env # 虚拟环境的路径
http =:8000 # 监听端口
wsgi-file=/opt/flask_demo/manager.py    # 配置启动应用的文件名
buffer-size = 65536
daemonize=/data/logs/flask.log   # pkill -f -9 uwsgi


#http = 0.0.0.0:8000     //测试uwsgi的时候使用这个注释掉socket
socket  = /home/tmp/programname.sock   //socket 文件，用于和nginx通信，推荐不要放在项目目录下，不然可能会报错，提示找不到该文件
chmod-socket    = 666
vacuum = true #退出uwsgi时是否清空socket文件

```

然后使用命令uwsgi --ini program_name_uwsgi.ini，看下是否还能启动这个项目。(如果选择socket可能会运行失败，提示没有sock文件，这时候可以先进行下一步nginx的部署，nginx会自动生成sock文件)

#### 安装nginx：

nginx是一个web服务器。用来加载静态文件和接收http请求的。通过命令`sudo apt install nginx`即可安装。
 nginx常用命令：

```shell
启动nginx：service nginx start
关闭nginx：service nginx stop
重启nginx：service nginx restart
测试nginx：service nginx configtest
差错nginx：nginx -t                  //查错，如有错会在终端报错
```

#### 编写nginx配置文件：

在/etc/nginx/conf.d目录下，新建一个文件，叫做program_name.conf，然后将以下代码粘贴进去：

```shell
upstream flask_demo{
    server unix:///tmp/program_name.sock; 
    #该sock文件与上面uwsgi配置文件内的sock文件应当为同一个文件
    #最好不要放在项目目录下，不然可能出显示找不到该文件。
}
# 配置服务器
server {
    # 监听的端口号
    listen      80;   
    # 这个是nginx输出的端口，也就是说后面你访问你的网站实例需要登录这个端口，添加端口映射时需要注意
    # 域名
    server_name  106.13.93.45; #此处填写域名或者ip，多个的情况下空格隔开
    charset     utf-8;

    # 最大的文件上传尺寸
    client_max_body_size 75M;

    # 静态文件访问的url
    location /static {
        # 静态文件地址
        alias  programdir/static;           #此处填写static文件路径
    }

    # 最后，发送所有非静态文件请求到django服务器
    location / {
        uwsgi_pass  flask_demo;
        # uwsgi_params文件地址
        include     /etc/nginx/uwsgi_params;
    }
}
```

写完配置文件后，为了测试配置文件是否设置成功，运行命令：`service nginx configtest`，如果不报错，说明成功，如果失败了，可以执行nginx -t，该指令会打印出来出错的地方。
 每次修改完了配置文件，都要记得运行`service nginx restart`。

#### 使用supervisor配置：

让supervisor管理uwsgi，可以在uwsgi发生意外的情况下，会自动的重启。
 1.supervisor的安装：
 在系统级别的python环境下`pip install supervisor`。(这里如果你用的是python3写的项目，也可以直接用pip安装启动supervisor，也就是python2，supervisor安装在3或者2，对你的项目没有任何影响)
 2.在项目的根目录下创建一个文件叫做program_name_supervisor.conf。内容如下：

```shell
# supervisor的程序名字
    [program:program_name]     #program_name 该名称可以随意设置
    # supervisor执行的命令
    command=uwsgi --ini program_name_uwsgi.ini
    # 项目的目录
    directory = programdir
    # 开始的时候等待多少秒
    startsecs=0
    # 停止的时候等待多少秒
    stopwaitsecs=0  
    # 自动开始
    autostart=true
    # 程序挂了后自动重启
    autorestart=true
    # 输出的log文件
    stdout_logfile=programdir/log/supervisord.log          #这里你可能需要先创建log路径
    # 输出的错误文件
    stderr_logfile=programdir/log/supervisord.err            #同上

    [supervisord]
    # log的级别
    loglevel=info

    # 使用supervisorctl的配置
    [supervisorctl]
    # 使用supervisorctl登录的地址和端口号
    serverurl = http://127.0.0.1:9001

    # 登录supervisorctl的用户名和密码
    username = 自定义
    password = 自定义

    [inet_http_server]
    # supervisor的服务器
    port = :9001
    # 用户名和密码
    username = 自定义
    password = 自定义

    [rpcinterface:supervisor]
    supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface
```

然后使用命令`supervisord -c program_name_supervisor.conf`运行就可以了。
 以后如果想要启动uwsgi，就可以通过命令`supervisorctl -c program_name_supervisor.conf`进入到管理控制台，然后可以执行相关的命令进行管理：
 指令如下:

```shell
status # 查看状态
start program_name #启动程序
restart program_name #重新启动程序
stop program_name # 关闭程序
reload # 重新加载配置文件
quit # 退出控制台
```