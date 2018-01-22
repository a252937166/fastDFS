[TOC]
# 基础概念
首先简单了解一下基础概念，FastDFS是一个开源的轻量级分布式文件系统，由跟踪服务器（tracker server）、存储服务器（storage server）和客户端（client）三个部分组成，主要解决了海量数据存储问题，特别适合以中小文件（建议范围：4KB < file_size <500MB）为载体的在线服务。
## Tracker（跟踪器）
Tracker主要做调度工作，相当于mvc中的controller的角色，在访问上起负载均衡的作用。跟踪器和存储节点都可以由一台或多台服务器构成，跟踪器和存储节点中的服务器均可以随时增加或下线而不会影响线上服务，其中跟踪器中的所有服务器都是对等的，可以根据服务器的压力情况随时增加或减少。Tracker负责管理所有的Storage和group，每个storage在启动后会连接Tracker，告知自己所属的group等信息，并保持周期性的心跳，tracker根据storage的心跳信息，建立group==>[storage server list]的映射表，Tracker需要管理的元信息很少，会全部存储在内存中；另外tracker上的元信息都是由storage汇报的信息生成的，本身不需要持久化任何数据，这样使得tracker非常容易扩展，直接增加tracker机器即可扩展为tracker cluster来服务，cluster里每个tracker之间是完全对等的，所有的tracker都接受stroage的心跳信息，生成元数据信息来提供读写服务。
## Storage（存储节点）
Storage采用了分卷[Volume]（或分组[group]）的组织方式，存储系统由一个或多个组组成，组与组之间的文件是相互独立的，所有组的文件容量累加就是整个存储系统中的文件容量。一个卷[Volume]（组[group]）可以由一台或多台存储服务器组成，一个组中的存储服务器中的文件都是相同的，组中的多台存储服务器起到了冗余备份和负载均衡的作用，数据互为备份，存储空间以group内容量最小的storage为准，所以建议group内的多个storage尽量配置相同，以免造成存储空间的浪费。
# 相关资源
本次搭建要用的资源都在这里了：https://github.com/a252937166/fastDFS.git
# 安装
## libfastcommon
因为libfastcommon使用c语言写的，所以我们先要安装gcc编译器：

```
yum -y install gcc-c++
```

看到如图(1)的信息说明gcc已经安装成功： 

![这里写图片描述](http://img.blog.csdn.net/20170507213451041?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTXJfT09P/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
<center>图(1)</center>

如果没有装解压工具unzip可以通过以下yum命令进行安装后再解压：
```
yum -y install unzip zip
```
首先第一步是安装libfastcommon，我这里将libfastcommon上传到的/usr/local目录下，直接解压：
```
unzip libfastcommon-master.zip
```

解压完成后就可以进行编译安装了，进入`libfastcommon-master`目录分别执行

```
./make.sh
```

没有error信息的话就说明编译成功了，最后再执行

```
./make.sh install
```

进行安装。
ibfastcommon.so 默认安装到了`/usr/lib64/libfastcommon.so`，但是FastDFS主程序设置的lib目录是`/usr/local/lib`，所以此处需要重新设置软链接（类似于Windows的快捷方式）：

```
ln -s /usr/lib64/libfastcommon.so /usr/local/lib/libfastcommon.so
ln -s /usr/lib64/libfastcommon.so /usr/lib/libfastcommon.so
ln -s /usr/lib64/libfdfsclient.so /usr/local/lib/libfdfsclient.so
ln -s /usr/lib64/libfdfsclient.so /usr/lib/libfdfsclient.so
```

设置完毕后就可以开始安装fastdfs了。
## FastDFS
第一步依然是解压：

tar -zxvf fastdfs-5.05.tar.gz 
解压完成后进入目录fastdfs-5.05，依次执行`./make.sh`和`./make.sh install`：

```
./make.sh
./make.sh install
```

![这里写图片描述](http://img.blog.csdn.net/20170507215010203?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTXJfT09P/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
<center>图(2)</center>

没有报错就说明安装成功了，在log中我们可以发现安装路径： 

![这里写图片描述](http://img.blog.csdn.net/20170507214332554?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTXJfT09P/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
<center>图(3)</center>

没错，正是安装到了/etc/fdfs中，安装成功后就会生成如上的3个.sample文件（示例配置文件），我们再分别拷贝出3个后面用的正式的配置文件：

```
cp client.conf.sample client.conf
cp storage.conf.sample storage.conf
cp tracker.conf.sample tracker.conf
```
再查看一下/etc/fdfs的文件目录：

![这里写图片描述](http://img.blog.csdn.net/20170507214855576?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTXJfT09P/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
<center>图(4)</center>

至此FastDFS已经安装完毕，接下来的工作就是依次配置Tracker和Storage了。
## Tracker

在配置Tracker之前，首先需要创建Tracker服务器的文件路径，即用于存储Tracker的数据文件和日志文件等，我这里选择在/opt目录下创建一个fastdfs_tracker目录用于存放Tracker服务器的相关文件：

```
mkdir /opt/fastdfs_tracker
```

接下来就要重新编辑上一步准备好的`/etc/fdfs`目录下的tracker.conf配置文件，打开文件后依次做以下修改：

```
disabled=false #启用配置文件（默认启用）
port=22122 #设置tracker的端口号，通常采用22122这个默认端口
base_path=/opt/fastdfs_tracker #设置tracker的数据文件和日志目录
http.server_port=6666 #设置http端口号，默认为8080
```

配置完成后就可以启动Tracker服务器了，但首先依然要为启动脚本创建软引用，因为fdfs_trackerd等命令在/usr/local/bin中并没有，而是在/usr/bin路径下：

```
ln -s /usr/bin/fdfs_trackerd /usr/local/bin
ln -s /usr/bin/stop.sh /usr/local/bin
ln -s /usr/bin/restart.sh /usr/local/bin
```

最后通过命令启动Tracker服务器：

```
service fdfs_trackerd start
```

如果启动命令执行成功，那么同时在刚才创建的tracker文件目录/opt/fastdfs_tracker中就可以看到启动后新生成的data和logs目录，tracker服务的端口也应当被正常监听，最后再通过netstat命令查看一下端口监听情况：

```
netstat -unltp|grep fdfs
```

可以看到tracker服务运行的22122端口正常被监听。
确认tracker正常启动后可以将tracker设置为开机启动，打开/etc/rc.d/rc.local并在其中加入以下配置：

```
service fdfs_trackerd start
```

![这里写图片描述](http://img.blog.csdn.net/20170507214731544?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTXJfT09P/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
<center>图(5)</center>

Tracker至此就配置好了，接下来就可以配置FastDFS的另一核心——Storage。 
## Storage
步骤基本与配置Tracker一致，首先是创建Storage服务器的文件目录，需要注意的是同Tracker相比我多建了一个目录，因为Storage还需要一个文件存储路径，用于存放接收的文件：

```
mkdir /opt/fastdfs_storage
mkdir /opt/fastdfs_storage_data
```

接下来修改`/etc/fdfs`目录下的storage.conf配置文件，打开文件后依次做以下修改：

```
disabled=false #启用配置文件（默认启用）
group_name=group1 #组名，根据实际情况修改
port=23000 #设置storage的端口号，默认是23000，同一个组的storage端口号必须一致
base_path=/opt/fastdfs_storage #设置storage数据文件和日志目录
store_path_count=1 #存储路径个数，需要和store_path个数匹配
store_path0=/opt/fastdfs_storage_data #实际文件存储路径
tracker_server=10.211.55.5:22122 #tracker 服务器的 IP地址和端口号，如果是单机搭建，IP不要写127.0.0.1，否则启动不成功（此处的ip是我的CentOS虚拟机ip）
http.server_port=8888 #设置 http 端口号
```

配置完成后同样要为Storage服务器的启动脚本设置软引用：

```
ln -s /usr/bin/fdfs_storaged /usr/local/bin
```

接下来就可以启动Storage服务了：

```
service fdfs_storaged start
```
如果启动成功，/opt/fastdfs_storage中就可以看到启动后新生成的data和logs目录

![这里写图片描述](http://img.blog.csdn.net/20170507215949302?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTXJfT09P/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity)
<center>图(6)</center>

如图(6)，没有任何问题，data下有256个1级目录，每级目录下又有256个2级子目录，总共65536个文件，新写的文件会以hash的方式被路由到其中某个子目录下，然后将文件数据直接作为一个本地文件存储到该目录中。那么最后我们再看一下storage服务的端口监听情况：

![这里写图片描述](http://img.blog.csdn.net/20170507220034858?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTXJfT09P/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity)
<center>图(7)</center>

如图(7)，可以看到此时已经正常监听tracker的22122端口和storage的23000端口，至此storage服务器就已经配置完成，确定了storage服务器启动成功后，还有一项工作就是看看storage服务器是否已经登记到 tracker服务器（也可以理解为tracker与storage是否整合成功），运行以下命令：

```
/usr/bin/fdfs_monitor /etc/fdfs/storage.conf
```

![这里写图片描述](http://img.blog.csdn.net/20170507220206297?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTXJfT09P/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity)
<center>图(8)</center>

如图(8)所示，看到10.211.55.5 ACTIVE 字样即可说明storage服务器已经成功登记到了tracker服务器，同理别忘了添加开机启动，打开/etc/rc.d/rc.local并将如下配置追加到文件中：

```
service fdfs_storage start
```

至此我们就已经完成了fastdfs的全部配置，此时也就可以用客户端工具进行文件上传下载的测试了。

## 初步测试
测试时需要设置客户端的配置文件，编辑/etc/fdfs目录下的client.conf 文件，打开文件后依次做以下修改：

```
base_path=/opt/fastdfs_tracker #tracker服务器文件路径
tracker_server=192.168.111.11:22122 #tracker服务器IP地址和端口号
http.tracker_server_port=6666 # tracker 服务器的 http 端口号，必须和tracker的设置对应起来
```

配置完成后就可以模拟文件上传了，先给/opt目录下随便放个文件（简历.pdf）。
然后通过执行客户端上传命令尝试上传：

```
/usr/bin/fdfs_upload_file  /etc/fdfs/client.conf  /opt/简历.pdf
```

运行后可以发现给我们返回了一个路径： 

![这里写图片描述](http://img.blog.csdn.net/20170507220534068?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTXJfT09P/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
<center>图(9)</center>

这就表示我们的文件已经上传成功了，当文件存储到某个子目录后，即认为该文件存储成功，接下来会为该文件生成一个文件名，文件名由group、存储目录、两级子目录、fileid、文件后缀名（由客户端指定，主要用于区分文件类型）拼接而成，我们到对应目录下也能找到对应的文件：

![这里写图片描述](http://img.blog.csdn.net/20170507220639994?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTXJfT09P/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
<center>图(10)</center>

但是此时并不能通过http访问，因为FastDFS目前已不支持http协议，所以此处在nginx上使用FastDFS的模块fastdfs-nginx-module，这样做最大的好处就是提供了HTTP服务并且解决了group中storage服务器的同步延迟问题，接下来就具体记录一下fastdfs-nginx-module的安装配置过程。
##fastdfs-nginx-module
在安装nginx之前需要先安装一些模块依赖的lib库，直接贴出安装代码：

```
yum -y install pcre pcre-devel  
yum -y install zlib zlib-devel  
yum -y install openssl openssl-devel
```

依次装好这些依赖之后就可以开始安装nginx了。
### storage nginx
首先分别进行解压：

```
tar -zxvf nginx-1.13.0.tar.gz
unzip fastdfs-nginx-module-master.zip
```

解压成功后就可以编译安装nginx了，进入nginx目录并输入以下命令进行配置：

```
./configure --prefix=/usr/local/nginx --add-module=/usr/local/fastdfs-nginx-module-master/src
```
配置成功后会看到如下信息：

![这里写图片描述](http://img.blog.csdn.net/20170507221101059?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTXJfT09P/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
<center>图(11)</center>

紧接着就可以进行编译安装了，依次执行以下命令：

```
make
make install
```

安装完成后，可以看到`/usr/local/`下新增了一个`nginx`目录，这就是刚才新安装的nginx的目录。
接下来要修改一下nginx的配置文件，进入conf目录并修改nginx.conf，把listen端口为80的server改为：

```
    server {
	listen       9999;

     	location ~/group1/M00 {
            root /opt/fastdfs_storage_data/data;
            ngx_fastdfs_module;
     }
    }
```
然后进入FastDFS的安装目录`/usr/local/fastdfs-5.05`目录下的conf目录，将http.conf和mime.types拷贝到/etc/fdfs目录下：

```
cp -r /usr/local/fastdfs-5.05/conf/http.conf /etc/fdfs/
cp -r /usr/local/fastdfs-5.05/conf/mime.types /etc/fdfs/
```

接下来还需要把fastdfs-nginx-module安装目录中`src`目录下的mod_fastdfs.conf也拷贝到`/etc/fdfs`目录下：

```
cp -r /usr/local/fastdfs-nginx-module-master/src/mod_fastdfs.conf /etc/fdfs/
```

没什么问题，接下来就需要编辑刚拷贝的这个mod_fastdfs.conf文件了，打开mod_fastdfs.conf并按顺序依次编译以下内容：
```
base_path=/opt/fastdfs_storage #保存日志目录
tracker_server=10.211.55.5:22122 #tracker服务器的IP地址以及端口号
storage_server_port=23000 #storage服务器的端口号
url_have_group_name = true #文件 url 中是否有 group 名
store_path0=/opt/fastdfs_storage_data # 存储路径
group_count = 3 #设置组的个数，事实上这次只使用了group1
```
设置了group_count = 3，接下来就需要在文件尾部追加这3个group setting：
```
[group1]
group_name=group1
storage_server_port=23000
store_path_count=1
store_path0=/opt/fastdfs_storage_data

[group2]
group_name=group2
storage_server_port=23000
store_path_count=1
store_path0=/opt/fastdfs_storage_data

[group3]
group_name=group3
storage_server_port=23000
store_path_count=1
store_path0=/opt/fastdfs_storage_data
```
接下来还需要建立 M00 至存储目录的符号连接：

```
ln  -s  /opt/fastdfs_storage_data/data  /opt/fastdfs_storage_data/data/M00
```

最后启动nginx：

```
/usr/local/nginx/sbin/nginx
```
显示如下信息说明nginx已启动成功：

![这里写图片描述](http://img.blog.csdn.net/20170507222322939?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTXJfT09P/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

<center>图(12)</center>

通过浏览器也可以看到nginx的主页： 

![这里写图片描述](http://img.blog.csdn.net/20170507222412640?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTXJfT09P/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

<center>图(13)</center>

如果访问不了，先ping一下自己的ip，如果能ping通，说明是防火墙的问题，在防火墙上打开对应端口就可以了：

```
/sbin/iptables -I INPUT -p tcp --dport 9999 -j ACCEPT
```
其他端口以此类推。
如果ping不通ip说明nginx配置不正确，或者ip不对。
storage服务器的nginx就已经安装完毕，接下来看一下tracker服务器的nginx安装。

### tracker nginx
同理，再装一个nginx，目录命名为nginx2，安装路径依旧放在/usr/local下，由于和之前一样，此处就不再做详细解释：
```
mkdir nginx2
cd nginx-1.13.0/
./configure --prefix=/usr/local/nginx2 --add-module=/usr/local/fastdfs-nginx-module-master/src
make
make install
```
接下来依然是修改nginx2的配置文件，进入conf目录并打开nginx.conf文件加入以下配置，tracker的nginx无需修改listen端口，即默认的80端口，并将upstream指向storage的nginx地址，在http节点下新增：
```
upstream fdfs_group1 {
     server 127.0.0.1:9999;
}

```
在server节点下新增：

```
location /group1/M00 {
     proxy_pass http://fdfs_group1;
}
```

接下来启动nginx2：

```
/usr/local/nginx2/sbin/nginx
```
最后一步就是需要修改`/etc/fdfs`目录下的client.conf文件，打开该文件并加入以下配置：

```
base_path=/opt/fastdfs_storage  #日志存放路径
tracker_server=10.211.55.5:22122  #tracker 服务器 IP 地址和端口号
http.tracker_server_port=6666  # tracker 服务器的 http 端口号，必须和tracker的设置对应起来
```

至此关于fastdfs就已经全部配置完毕了，再一次进行测试看看是否能正常上传文件并通过http访问文件。
### HTTP测试
再给/opt目录下上传一张图片（timg.jpg）
通过客户端命令测试上传： 

![这里写图片描述](http://img.blog.csdn.net/20170507225818348?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTXJfT09P/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
<center>图(14)</center>

如图（14），依旧上传成功，接下来的关键就是通过HTTP测试文件访问，打开浏览器输入ip地址+文件名看看是否能正常访问该图片： 

![这里写图片描述](http://img.blog.csdn.net/20170507225927208?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTXJfT09P/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
<center>图(15)</center>

一切正常~ 至此关于FastDFS在CentOS 7下的部署测试就已经全部完成了。
# 参考文章
http://blog.csdn.net/wlwlwlwl015/article/details/52619851
我基本上都是照上面这篇文章操作的，讲得很详细。


