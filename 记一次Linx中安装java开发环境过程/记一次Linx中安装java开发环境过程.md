

---

## Linx中安装java开发环境（上）

> 提示：本文会比较长，会分为上下两篇文章。下面目录。

[TOC]

### 准备工作：

1. 本次环境配置中，安装的主要环境有：JDK , Mysql ,Nginx ,  FastDFS , Redis .

2. 环境版本设置：

   - Linx ： CentOS 6.8  32位机 ；


   - JDK ： jdk-8u151-linux-i586 ；
   - Mysql ：mysql-5.6.35；
   - Nginx : nginx 1.8 ;
   - FastDFS : 
   - Redis : 

3. 环境所需要的安装包需要下载好，并核对操作系统和软件版本是否一致；毕竟很多软件的是无法向上兼容的。

---

### JDK安装：

- 将jdk-8u151-linux-i586.tar.gz使用SSH工具上传至Linx中。

  > 我这边除了mysql之外将所有软件放在 /usr/local/server/目录下，可以更改至其他目录，但因下面将会有很多路径会参照该路径，所以不建议更改。


- 解压 jdk-8u151-linux-i586.tar.gz

  ```shell
  [root@Xingchen server] tar -zxvf jdk-8u151-linux-i586.tar.gz
  ```


- 编辑配置文件，配置环境变量

  ```shell
  [root@Xingchen server] vim /etc/profile
  ```


- 在 profile 末尾加上以下环境变量

  ```shell
  #java
  JAVA_HOME=/usr/local/server/jdk1.8.0_151
  CLASSPATH=$JAVA_HOME/lib/
  PATH=$PATH:$JAVA_HOME/bin
  export PATH JAVA_HOME CLASSPATH
  ```

- 使配置文件立即生效

  ```shell
  [root@Xingchen server] source /etc/profile
  ```

- 测试是否安装成功

  ```shell
  [root@Xingchen server] java -version
  ```

- 如果出现以下输出，说明安装成功

  ```shell
  java version "1.8.0_151"
  Java(TM) SE Runtime Environment (build 1.8.0_151-b12)
  Java HotSpot(TM) Client VM (build 25.151-b12, mixed mode)
  ```

  ​

此时，java安装成功。Oh yes！

---

### Mysql安装 :

- 首先看看本地是否已经安装过mysql , 否则以后可能会出一堆问题。

  ```shell
  [root@Xingchen] rpm -qa | grep mysql
  ```


- 有的话通过下面的命令来卸载掉

  ```shell
  [root@Xingchen] rpm -e mysql
  ```

  注：如果以上方式无法成功删除原mysql ， 可以使用强力删除模式删除 ： rpm -e --nodeps mysql

  安装编译源码所需的工具和库

  ```shell
  [root@Xingchen] yum -y install gcc gcc-c++ ncurses-devel bison-devel perl-devel  perl-Module-Install.noarch
  ```

  安装cmake

  ```shell
  [root@Xingchen] cd /usr/local/src  
  [root@Xingchen] wget https://cmake.org/files/v3.5/cmake-3.5.2.tar.gz  
  ```

  如果wget工具未安装，先安装wget,不使用wget也可以，就是把下载好的源码包上传到服务器

  ```shell
  [root@Xingchen] yum -y install wget
  [root@Xingchen] tar -xzf cmake-3.5.2.tar.gz
  [root@Xingchen] cd cmake-3.5.2  
  [root@Xingchen] ./bootstrap
  [root@Xingchen] make
  [root@Xingchen] make install 
  ```

  检测weget是否安装成功

  ```shell
  [root@Xingchen] cmake -version
  cmake version 3.5.2  		#说明安装成功
  CMake suite maintained and supported by Kitware (kitware.com/cmake).
  ```


- 安装Mysql

  1.创建mysql的安装目录及数据库存放目录

  ```shell
  [root@Xingchen] mkdir -p /usr/local/mysql  //安装mysql  
  [root@Xingchen] mkdir -p /usr/local/mysql/data //存放数据库 
  ```

  2.开始安装

  ```shell
  [root@Xingchen] cd /usr/local/src  
  [root@Xingchen] wget http://mirrors.sohu.com/mysql/MySQL-5.6/mysql-5.6.35.tar.gz  
  [root@Xingchen] tar -xzf mysql-5.6.35.tar.gz  
  [root@Xingchen] cd mysql-5.6.35  
  [root@Xingchen] cmake . -DCMAKE_INSTALL_PREFIX=/usr/local/mysql -DMYSQL_DATADIR=/usr/local/mysql/data -DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci -DEXTRA_CHARSETS=all -DENABLED_LOCAL_INFILE=1

  [root@Xingchen] make  //这个时间比较长  
  [root@Xingchen] make install
  ```

  cmake命令参数说明：

​	-DENABLED_LOCAL_INFILE=1 //允许从本地导入数据

​	-DCMAKE_INSTALL_PREFIX=/usr/local/mysql   //安装目录

​	-DINSTALL_DATADIR=/usr/local/mysql/data   //数据库存放目录

​	-DDEFAULT_CHARSET=utf8//使用utf8字符
​	-DDEFAULT_COLLATION=utf8_general_ci    //校验字符

​	-DEXTRA_CHARSETS=all //安装所有扩展字符集

- 注：如果要重新编译，需要清除就得对象和编译信息

  ```shell
  [root@Xingchen] make clean  
  [root@Xingchen] rm -f CMakeCache.txt  
  [root@Xingchen] rm -rf /etc/my.cnf 
  ```

  - mysql配置

    设置目录权限

    ```shell
    [root@Xingchen] cd /usr/local/mysql  
      
    //把当前目录中所有文件的所有者所有者设为msyql，所属组为mysql；不要丢掉最后一个‘ . ’  
    [root@Xingchen] chown -R mysql:mysql . 
    ```

    复制mysql服务启动配置文件到/usr/local/mysql下并重命名为my.cnf，/usr/local/mysql/my.cnf如存在则输入y和回车覆盖配置文件，如果没有的话，mysql所有的参数将会是默认值

    ```shell
    [root@Xingchen] cp support-files/my-default.cnf/usr/local/mysql/my.cnf          
    创建系统数据库的表  
    [root@Xingchen] scripts/mysql_install_db --user=mysql  
    设置环境变量  
    [root@Xingchen] vi /etc/profile  
    ```

    在最下面添加下面代码 ,然后保存文件

    ```shell
    #mysql
    export PATH=/usr/local/mysql/bin:$PATH 
    ```

    使文件立即生效，并查看是否生效

    ```shell
    [root@Xingchen] source /etc/profile  
    [root@Xingchen] echo $PATH  
    ```

    将mysql的启动服务添加到系统服务中

    ```shell
    [root@Xingchen] cp support-files/mysql.server /etc/init.d/mysql  
    ```

    这样配置完成之后就可以使用下面的命令启动和停止mysql服务

    ```shell
    [root@Xingchen] service mysql start //启动  
    [root@Xingchen] service mysql stop //停止  
    [root@Xingchen] service mysql restart //重启 
    ```

    将mysq服务加入开机启动中

    ```shell
    [root@Xingchen] chkconfig --level 35 mysql on  
    ```

    检查mysql服务是否启动

    ```shell
    [root@Xingchen]  netstat -tulnp | grep 3306   
    ```

    修改MySQL的root用户的密码
    设置mysql的root用户密码，此处设成123456，此root用户与linux不是一个

    ```shell
    [root@Xingchen] mysqladmin -u root password '123456' 
    ```

    登陆mysql，用刚才设置的密码

    ```shell
    [root@Xingchen] mysql -u root -p
    Enter password: 					#输入密码
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 11
    Server version: 5.6.35 Source distribution

    Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    mysql> 
    ```

    Mysql安装成功，本机可以使用了。但是很多时候我们需要配置数据库为远程调用，这个配置方法有很多种，这里使用其中一种。如下：

    首先使用root账户登录mysql,然后执行以下语句。

  - ```shell
    mysql>grant all privileges on *.* to admin@"%" identified by 'something' with grant option;
    # 添加一个用户admin并授权可从任何其它主机发起的访问（通配符％）。使用这一条语句即可。something就是远程登录帐号密码
    ```

- 此时。Mysql安装完成。

---

### Nginx安装：

使用nginx作为代理服务器。



​	







































































