**这是继 Linx中安装java开发环境过程(上) 的一篇文章**

---

目录

[TOC]

##FastDFS的安装配置

###什么是FastDFS

FastDFS是用C语言编写的一款开源的分布式文件系统。FastDFS为互联网量身定制，充分考虑了冗余备份、负载均衡、线性扩容等机制，并注重高可用、高性能等指标，使用FastDFS很容易搭建一套高性能的文件服务器集群提供文件上传、下载等服务。

###FastDFS架构

FastDFS架构包括 Tracker server和Storageserver。客户端请求Tracker server进行文件上传、下载，通过Tracker server调度最终由Storage server完成文件上传和下载。 Trackerserver作用是负载均衡和调度，通过Tracker server在文件上传时可以根据一些策略找到Storage server提供文件上传服务。可以将tracker称为追踪服务器或调度服务器。Storage server作用是文件存储，客户端上传的文件最终存储在Storage服务器上，Storage server没有实现自己的文件系统而是利用操作系统 的文件系统来管理文件。可以将storage称为存储服务器。可以参照下图理解这句话：

![dfs](C:\Users\ASUS\Documents\A_BLOG\记一次Linx中安装java开发环境过程(下)\dfs.png)

3. 高级用途(了解一下)

其实在很多现在的大型项目中，特别是文件存储方面业务较多的项目。FastDFS是不错的选择，结合Tracker server和storager server ,都可以搭建集群以实现项目的高可用，高并发，增强项目的健壮性。那么FastDFS搭建集群的原理是什么呢？

- Tracker 集群

astDFS集群中的Tracker server可以有多台，Tracker server之间是相互平等关系同时提供服务，Tracker server不存在单点故障。客户端请求Tracker server采					用轮询方式，如果请求的tracker无法提供服务则换另一个tracker。

- Storage集群

​       Storage集群采用了分组存储方式。storage集群由一个或多个组构成，集群存储总容量为集群中所有组的存储容量之和。一个组由一台或多台存储服务器组成，组内的Storage server之间是平等关系，不同组的Storage server之间不会相互通信，同组内的Storage server之间会相互连接进行文件同步，从而保证同组内每个storage上的文件完全一致的。一个组的存储容量为该组内存储服务器容量最小的那个，由此可见组内存储服务器的软硬件配置最好是一致的。

​       采用分组存储方式的好处是灵活、可控性较强。比如上传文件时，可以由客户端直接指定上传到的组也可以由tracker进行调度选择。一个分组的存储服务器访问压力较大时，可以在该组增加存储服务器来扩充服务能力（纵向扩容）。当系统容量不足时，可以增加组来扩充存储容量（横向扩容）。

- Storage状态收集

  Storage server会连接集群中所有的Tracker server，定时向他们报告自己的状态，包括磁盘剩余空间、文件同步状况、文件上传下载次数等统计信息。

### FastDFS文件上传下载

1. 文件上传流程

   ![upload](C:\Users\ASUS\Documents\A_BLOG\记一次Linx中安装java开发环境过程(下)\upload.png)

解释：客户端上传文件后[存储服务器]()将[文件]()ID返回给客户端，此文件ID用于以后访问该文件的索引信息。文件索引信息包括：组名，虚拟磁盘路径，数据两级目录，文件名。

返回的索引信息如下：

```java
group1/M00/02/44/abcdefghijklmdfd.html
```

拆开：

- 组名【group01】:文件上传后所在的[storage]()组名称，在文件上传成功后有storage服务器返回，需要客户端自行保存。 
- 虚拟磁盘路径【M00】:[storage]()配置的虚拟路径，与磁盘选项[store_path]()*对应。如果配置了store_path0则是M00，如果配置了store_path1则是M01，以此类推。
- 数据两级目录【/02/44/】：storage服务器在每个虚拟磁盘路径下创建的两级目录，用于存储数据文件。
- 文件名【abcdefghijklmdfd.html】 : 与文件上传时不同。是由存储服务器根据特定信息生成，文件名包含：源存储服务器IP地址、文件创建时间戳、文件大小、随机数和文件拓展名等信息。



2. 文件下载流程

   ![download](C:\Users\ASUS\Documents\A_BLOG\记一次Linx中安装java开发环境过程(下)\download.png)

解释：

​	2.1 通过组名tracker能够很快的定位到客户端需要访问的存储服务器组是group1，并选择合适的存储服务器提供客户端访问。  

​	2.2存储服务器根据“文件存储虚拟磁盘路径”和“数据文件两级目录”可以很快定位到文件所在目录，并根据文件名找到客户端需要访问的文件。



上面了解了FastDFS系统之后，下面就是安装配置。由于给外界访问时需要服务器，因此我们使用Nginx代理服务器，将请求转发给nginx的一个fastdfs-nginx-module来处理。由此实现文件上传下载。

---

###FastDFS + Nginx安装配置

首先看一下系统架构图，如下图：

![TIM截图20171205212202](C:\Users\ASUS\Documents\A_BLOG\记一次Linx中安装java开发环境过程(下)\TIM截图20171205212202.png)

安装开始

---

####准备相关安装包

下载地址：http://sourceforge.net/projects/FastDFS/或 https://github.com/happyfish100/FastDFS（推荐）

> 我会将所有安装包和相关软件全部上传至 /usr/local/server/  路径下，以下也都是基于这个前提的。如果你的路径跟我的不一致，需要注意将路径替换为你自己的。

```shell
fastdfs-nginx-module_v1.16.tar.gz
FastDFS_v5.05.tar.gz
libfastcommonV1.0.7.tar.gz
```

####版本约定

该文章使用的是 FastDFS_v5.05

####FastDFS依赖库安装

1. **gcc-c++安装**

```shell
[root@Xingchen] yum install gcc-c++
```

注：如果出现以下则说明已经安装过，可以直接进行下一步骤。

```shell
Loaded plugins: fastestmirror
Setting up Install Process
Loading mirror speeds from cached hostfile
Package gcc-c++-4.4.7-18.el6.i686 already installed and latest version
Nothing to do
```

2. **Libevent安装**

```shell
[root@Xingchen] yum -y install libevent
```

3. **libfastcommon安装**

- libfastcommon是FastDFS官方提供的，libfastcommon包含了FastDFS运行所需要的一些基础库。首先就是将准备好的 **libfastcommonV1.0.7.tar.gz **上传解压。

```shell
[root@Xingchen] cd /usr/local/server
[root@Xingchen server] tar -zxvf libfastcommonV1.0.7.tar.gz
```

- 进入libfastcommon编译安装

```shell
[root@Xingchen server] cd libfastcommon-1.0.7
[root@Xingchen libfastcommon-1.0.7] ./make.sh && ./make.sh install

#......这里省略其他输出信息
mkdir -p /usr/lib64
install -m 755 libfastcommon.so /usr/lib64
mkdir -p /usr/include/fastcommon
install -m 644 common_define.h hash.h chain.h logger.h base64.h shared_func.h pthread_func.h ini_file_reader.h _os_bits.h sockopt.h sched_thread.h http_func.h md5.h local_ip_func.h avl_tree.h ioevent.h ioevent_loop.h fast_task_queue.h fast_timer.h process_ctrl.h fast_mblock.h connection_pool.h /usr/include/fastcommon
```

- 将/usr/lib64目录下的 **libfastcommon.os** 拷贝到/usr/lib目录下

```shell
[root@Xingchen libfastcommon-1.0.7] cp /usr/lib64/libfastcommon.so /usr/lib/libfastcommon.so
```

####Tracker安装

1. 上传FastDFS_v5.05.tar.gz文件到服务器,解压。

```shell
[root@Xingchen server] tar -zxvf FastDFS_v5.05.tar.gz
```

2. 编译：到解压目录FastDFS下，可以看到一个make.sh，输入编译指令./make.sh执行编译。

   安装：./make.sh install

```shell
[root@Xingchen server] cd FastDFS
[root@Xingchen FastDFS] ./make.sh && ./make.sh install
```

​	执行完成后， /etc/fdfs 路径下会产生以下三个文件：

​	client.conf.sample  

​	storage.conf.sample  

​	tracker.conf.sample

3. 开始配置 「**这里配置有点多**」

​	3.3.1 前往  /etc/fdfs  ,将这三个文件后缀sample删掉。删掉后就是下面这样 ： 

​	client.conf  

​	storage.conf  

​	tracker.conf

​	3.3.2 编辑tracker.conf文件 ，更改相关字段，具体如下：

​	base_path   :   日志和数据存储目录 , 配置成  /home/FastDFS 。然后在创建改文件夹。

​	store_lookup :  这里不需要改，但要明白store_lookup=N 和store_group=groupN 这里的N必须相同。

```shell
# -----tracker.conf----
# the base path to store data and log files
base_path=/home/FastDFS	
```

```shell
[root@Xingchen /] mkdir /home/FastDFS
```

```shell
# the method of selecting group to upload files
# 0: round robin
# 1: specify group
# 2: load balance, select the max free space group to upload file
store_lookup=1

# which group to upload file
# when store_lookup set to 1, must set store_group to the group name
store_group=group1
```

​	3.3.3 启动 tracker

```shell
[root@Xingchen /] /usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf restart
```

​	通过ps –ef|grep tracker 查看进程信息

```shell
[root@Xingchen server] ps -ef|grep tracker
root     24810     1  0 22:15 ?        00:00:00 /usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf restart
root     24820 24174  0 22:15 pts/1    00:00:00 grep tracker
```

​	3.3.4 设置开机启动

```shell
[root@Xingchen server] vi /etc/rc.local
```

​	将启动指令写入文件即可。

```shell
#Tracker
/usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf restart
```

####storage安装

说明： Storage在同一台安装tracker的机器上不需要进行安装，只需要修改配置即可，但如果不是同一台机器这里需要执行上面tracker安装的过程（libfastcommon、libevent、storage的编译+安装）

1.配置

1.1 到/etc/fdfs目录，编辑storage.conf，修改storage相关配置，具体如下：

```html
group_name=group1		----> 这个根据你之前配Tracker使用的是哪一个组就配置成哪一个足。因为上面步骤中我配置成了group1,所以这里就是group1

base_path=/home/yuqing/FastDFS改为：base_path=/home/FastDFS

store_path0=/home/yuqing/FastDFS改为：store_path0=/home/FastDFS/fdfs_storage

#如果有多个挂载磁盘则定义多个store_path，如下

#store_path1=.....

#store_path2=......

tracker_server=192.168.174.130:22122   #配置tracker服务器:IP

#如果有多个则配置多个tracker

tracker_server=192.168.101.4:22122

注意：store_path0=/home/FastDFS/fdfs_storage 这里的路径必须存在，否则启动storage的时候会一直报错。[下面会有步骤创建该文件夹]

```

1.2 创建 /home/FastDFS/fdfs_storage 文件夹

```shell
[root@Xingchen server] mkdir /home/FastDFS/fdfs_storage
```



2. 启动

```shell
[root@Xingchen server] /usr/bin/fdfs_storaged /etc/fdfs/storage.conf restart
```

启动打印的Log文件在这里： /home/FastDFS/logs/   这个文件夹下有 Tracker和storage启动日志。如果发现一直卡顿在启动命令下，可以到这个log文件下寻找答案。下面是启动成功时storage.log文件的部分内容。

```shell
#创建文件仓库 ....
mkdir data path: FD ...
mkdir data path: FE ...
mkdir data path: FF ...
data path: /home/FastDFS/fdfs_storage/data, mkdir sub dir done.
[2017-12-05 22:31:00] INFO - file: storage_param_getter.c, line: 191, use_storage_id=0, id_type_in_filename=ip, storage_ip_changed_auto_adjust=1, store_path=0, reserved_storage_space=10.00%, use_trunk_file=0, slot_min_size=256, slot_max_size=16 MB, trunk_file_size=64 MB, trunk_create_file_advance=0, trunk_create_file_time_base=02:00, trunk_create_file_interval=86400, trunk_create_file_space_threshold=20 GB, trunk_init_check_occupying=0, trunk_init_reload_from_binlog=0, trunk_compress_binlog_min_interval=0, store_slave_file_use_link=0
[2017-12-05 22:31:00] INFO - file: storage_func.c, line: 254, tracker_client_ip: 192.168.1.222, my_server_id_str: 192.168.1.222, g_server_id_in_filename: -570316608
[2017-12-05 22:31:00] INFO - local_host_ip_count: 2,  127.0.0.1  192.168.1.222
[2017-12-05 22:31:00] INFO - file: tracker_client_thread.c, line: 310, successfully connect to tracker server 119.23.62.228:22122, as a tracker client, my ip is 192.168.1.222
[2017-12-05 22:31:00] INFO - file: storage_sync.c, line: 2698, successfully connect to storage server 119.23.62.228:23000
#启动成功
```

3. 设置开机启动,将启动命令加入到 /etc/rc.local

```shell
#storage
/usr/bin/fdfs_storaged /etc/fdfs/storage.conf restart
```

#### Nginx安装「根据FastDFS」

> 说明：这个nginx安装时针对FastDFS的配置 ，如果你的nginx不需要FastDFS ,不用按照这个步骤安装。只需要直接安装，再更改conf.properties即可。
>
> 我也有相关文章，如需要。可移步至 //todo绑定nginx安装文章URL

1. **安装fastdfs-nginx-module**

   1.1 解压

```shell
[root@Xingchen server] tar –xf fastdfs-nginx-module_v1.16.tar.gz
```

​	1.2 再进入fastdfs-nginx-module/src目录

```shell
[root@Xingchen src] vi config
ngx_addon_name=ngx_http_fastdfs_module
HTTP_MODULES="$HTTP_MODULES ngx_http_fastdfs_module"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_fastdfs_module.c"
CORE_INCS="$CORE_INCS /usr/local/include/fastdfs /usr/local/include/fastcommon/"
CORE_LIBS="$CORE_LIBS -L/usr/local/lib -lfastcommon -lfdfsclient"
CFLAGS="$CFLAGS -D_FILE_OFFSET_BITS=64 -DFDFS_OUTPUT_CHUNK_SIZE='256*1024' -DFDFS_MOD_CONF_FILENAME='\"/etc/fdfs/mod_fastdfs.conf\"'"
```

源文件如上，我们需要将所有路径中的 local 删掉。看图：

![TIM图片20171205224911](C:\Users\ASUS\Documents\A_BLOG\记一次Linx中安装java开发环境过程(下)\TIM图片20171205224911.png)

嗯，直接删掉。为什么？因为这个路径不存在，不删等会会报错。编辑好后 ESC - :wq 保存退出。

​	1.3 配置 mod_fastdfs.conf ,具体如下：

```shell
Base_path=/home/FastDFS		#这里是fastdfs的文件目录存放地址。
tracker_server=119.23.62.228:22122  #这个是tracker的访问ip和端口
group_name=group1 #这个是每一个storage对应的组
url_have_group_name=true #访问文件，是否需要加上组名
store_path0  #组群中store_path0的文件存储目录
```

​	1.4 将 mod_fastdfs.conf 拷贝到/etc/fdfs目录下

```shell
[root@Xingchen src]# cp /usr/local/server/fastdfs-nginx-module/src/mod_fastdfs.conf /etc/fdfs/mod_fastdfs.conf
```

​	1.5 拷贝对应库文件到/usr/lib目录

​	Fastdfs需要用到/usr/lib64目录下的2个文件，需要将这2个文件拷贝到/usr/lib目录下

```shell
libfastcommon.so
libfdfsclient.so
```

```shell
#拷贝文件
[root@Xingchen src] cd /usr/lib64
[root@Xingchen lib64] cp * ../lib
```



2. **安装Nginx**

将上传解压到/usr/local/server目录下

```shell
[root@Xingchen src]# tar -zxvf nginx-1.8.0.tar.gz
```

进入nginx-1.8.0目录。

```shell
[root@Xingchen server]# cd /usr/local/server/nginx-1.8.0
```

安装Nginx之前，需要安装nginx的安装环境「以下命令需要一条一条执行」：

```shell
yum install gcc-c++

yum install -y pcre pcre-devel

yum install -y zlib zlib-devel

yum install -y openssl openssl-devel
```

执行配置

```shell
[root@Xingchen server] ./configure --prefix=/usr/local/server/nginx \
--add-module=/usr/local/server/fastdfs-nginx-module/src
```

编译安装

```shell
[root@Xingchen server] make && make install
```

前往 /usr/local/server/FastDFS/conf 拷贝两个配置文件 到 **/etc/fdfs**

```shell
http.conf
mime.types
```

启动

```shell
[root@Xingchen server] /usr/local/server/nginx/sbin/nginx
```

加入到开机启动配置 

```shell
[root@Xingchen server] vi /etc/rc.local
```

```shell
#nginx
/usr/local/server/nginx/sbin/nginx
```

此时Nginx就可以访问了。我们试试

![TIM截图20171205232145](C:\Users\ASUS\Documents\A_BLOG\记一次Linx中安装java开发环境过程(下)\TIM截图20171205232145.png)

---

Oh yes!

但是这里还没配置完成。还有FastDFS和Nginx之间的关系配置上去。见下

3. 配置Nginx

前往/usr/local/server/nginx/conf ,编辑nginx.conf

```shell
#上面的省略...
server {
        listen       80;				#监听端口
        server_name  localhost;			#localhost

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location /group1/M00 {
        	root /home/fastDFS/fdfs_storage/data;
        	ngx_fastdfs_module;
        }
        
        #....下面的省略
     }
```

保存即可，此时。需要重启 Tracker , Storage , nginx ，然后使用java客户端工具，将文件上传至FastDFS ,即可访问测试。

####启动Tracker , Storage , nginx

```shell
#重启Tracker , Storage , nginx
/usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf restart
/usr/bin/fdfs_storaged /etc/fdfs/storage.conf restart
/usr/local/server/nginx/sbin/nginx
```

---

为了方便测试，这里附加一个java编写FastDFS上传，下载客户端的工具类。

前提，需要下载一个fastdfs-client-java包。当然如果你使用的是maven管理项目的话，引入一下坐标即可。

```xml
<!--FastDFS 客户端-->
<dependency>
    <groupId>org.csource</groupId>
    <artifactId>fastdfs-client-java</artifactId>
    <version>1.27-SNAPSHOT</version>
</dependency>
```

以下是工具类（注：暂时只写了 上传和删除两个功能,下载功能过段时间再完善）

```java
public class FastDFSClient {
    private TrackerClient trackerClient = null;
    private TrackerServer trackerServer = null;
    private StorageServer storageServer = null;
    private StorageClient1 storageClient = null;

    public FastDFSClient(String conf) throws Exception {
        if (conf.contains("classpath:")) {
            conf = conf.replace("classpath:", this.getClass().getResource("/").getPath());
        }
        ClientGlobal.init(conf);
        trackerClient = new TrackerClient();
        trackerServer = trackerClient.getConnection();
        storageServer = null;
        storageClient = new StorageClient1(trackerServer, storageServer);
    }

    /**
     * 上传文件方法
     * <p>Title: uploadFile</p>
     * <p>Description: </p>
     *
     * @param fileName 文件全路径
     * @param extName  文件扩展名，不包含（.）
     * @param metas    文件扩展信息
     * @return
     * @throws Exception
     */
    public String uploadFile(String fileName, String extName, NameValuePair[] metas) throws Exception {
        String result = storageClient.upload_file1(fileName, extName, metas);
        return result;
    }


    public String uploadFile(String fileName) throws Exception {
        return uploadFile(fileName, null, null);
    }

    public String uploadFile(String fileName, String extName) throws Exception {
        return uploadFile(fileName, extName, null);
    }

    /**
     * 删除文件
     * @param groupName 组名
     * @param remoteName 文件名
     * @return 结果
     * @throws IOException
     * @throws MyException
     */
    public int deleteFile(String groupName,String remoteName) throws IOException, MyException {
        return storageClient.delete_file(groupName,remoteName);
    }

    /**
     * 删除文件 组名+路径
     * @param storagePath
     * @return
     * @throws IOException
     * @throws MyException
     */
    public int deleteFile(String storagePath) throws IOException, MyException {
        return storageClient.delete_file1(storagePath);
    }
}
```

工具类使用：

再代码中创建一个FastDFSClient对象，在构造函数中传入配置文件路径。在调用upload方法即可。其配置文件内容就是让Client知道我们的Tracker server在哪里。举个例子，我们在 E:\client下创建client.conf

client.conf内容为：

```shell
tracker_server=119.23.62.228:22122		#这是你的tracker server 所在IP+端口
```

调用工具类

```java
//省略程序入口和其他代码
FastDFSClient client = new FastDFSClient("E:/client/client.conf");
String rtn = client.upload("你的文件路径","你的文件后缀");
//rtn就是返回信息；如果上传成功，就返回文件在FastDFS中的位置：组名+磁盘+文件目录+文件名+文件类型
```



---

总结：至此，FastDFS和Nginx安装配置完毕，刚开始准备吧Redis的安装也写在这篇文章中。不过现在回去看了一下篇幅，感觉这里就够了。而且FastDFS配置相对繁琐，复杂。当初我也配了4,5遍才能正常使用，因此配置时需要耐心，细心。而且这种环境配置不是经常使用，容易忘记，所以以这种方式记录下来，加深理解和记忆。也是为了以后遇到相关需求时能够快速无误的完成环境问题，提升工作效率（少加点班>_<）。



