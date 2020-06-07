##Openresty WAF模块编译和配置
###ubuntu安装openresty
>https://openresty.org/cn/linux-packages.html

install some prerequisites needed by adding GPG public keys (could be removed later)
```bash
sudo apt-get -y install --no-install-recommends wget gnupg ca-certificates
```

import our GPG key:
```bash
wget -O - https://openresty.org/package/pubkey.gpg | sudo apt-key add -
```

add the our official APT repository:
```bash
echo "deb http://openresty.org/package/ubuntu $(lsb_release -sc) main" \
    | sudo tee /etc/apt/sources.list.d/openresty.list
```
    
出现错误:
bash: lsb_release: command not found
sudo: setrlimit(RLIMIT_CORE): Operation not permitted

安装 lsb-release:
```bash
apt-get install lsb-release
```

配置sudo(https://unix.stackexchange.com/questions/578949/sudo-setrlimitrlimit-core-operation-not-permitted), 
```bash
echo "Set disable_coredump false" >> /etc/sudo.conf
```

重新执行ok, openresty安装成功.创建软连接:
```bash
sudo ln -s /usr/local/openresty/nginx/sbin/nginx /usr/bin/nginx
```
    
to update the APT index:
```bash
sudo apt-get update
```

Then you can install a package, say, openresty, like this:
sudo apt-get -y install openresty
This package also recommends the openresty-opm and openresty-restydoc packages so the latter two will also automatically get installed by default. If that is not what you want, you can disable the automatic installation of recommended packages like this:
sudo apt-get -y install --no-install-recommends openresty

查看openresty版本:
```bash
nginx -v
nginx version: openresty/1.15.8.3
```

下载openresty源码:
```bash
wget https://openresty.org/download/openresty-1.15.8.3.tar.gz
tar zxf ./openresty-1.15.8.3.tar.gz
```
下载ModSecurity:
```bash
git clone https://github.com/SpiderLabs/ModSecurity.git
```
下载Modsecurity connector:
```bash
git clone https://github.com/SpiderLabs/ModSecurity-nginx.git　
```

现在目录样式为:  
>├── ModSecurity    
├── ModSecurity-nginx  
├── openresty+waf.md  
├── openresty-1.15.8.3  
└── openresty-1.15.8.3.tar.gz  

#Openresty(Nginx) WAF
                
###编译ModSecurity:                
https://github.com/SpiderLabs/ModSecurity-nginx
>根据 https://github.com/SpiderLabs/ModSecurity-nginx#compilation 中的描述:  
Before compile this software make sure that you have libmodsecurity(https://github.com/SpiderLabs/ModSecurity.git) installed.
You can download it from the ModSecurity git repository.
For information pertaining to the compilation and installation of libmodsecurity please consult the documentation provided along with it.

在安装modsecurity-nginx之前需要先安装modsecurity, 安装步骤:
>参考资料:https://github.com/SpiderLabs/ModSecurity/wiki/Compilation-recipes-for-v3.x#ubuntu-1504
```bash
sudo apt-get -y install bison curl doxygen libyajl-dev libgeoip-dev libtool dh-autoreconf libcurl4-gnutls-dev libxml2 libpcre++-dev libxml2-dev openssl git libssl-dev

cd /opt/
git clone https://github.com/SpiderLabs/ModSecurity
cd ModSecurity/
git checkout -b v3/master origin/v3/master
sh build.sh
git submodule update --init --recursive  #[for bindings/python, others/libinjection, test/test-cases/secrules-language-tests]
./configure
make
make install

安装输出结果:
Libraries have been installed in:  
   /usr/local/modsecurity/lib  
If you ever happen to want to link against installed libraries  
in a given directory, LIBDIR, you must either use libtool, and  
specify the full pathname of the library, or use the '-LLIBDIR'  
flag during linking and do at least one of the following:  
   - add LIBDIR to the 'LD_LIBRARY_PATH' environment variable  
   during execution  
   - add LIBDIR to the 'LD_RUN_PATH' environment variable  
   during linking  
   - use the '-Wl,-rpath -Wl,LIBDIR' linker flag  
   have your system administrator add LIBDIR to '/etc/ld.so.conf'  
See any operating system documentation about shared libraries for  
more information, such as the ld(1) and ld.so(8) manual pages.  
```

ModeSecurity 被安装到了:/usr/local/modsecurity/目录下, header和lib都在这个目录下

##编译openresty with modesecurity_module
根据文档https://github.com/SpiderLabs/ModSecurity-nginx#compilation 描述:  
> Compilation
With libmodsecurity installed, you can proceed with the installation of the ModSecurity-nginx connector, which follows the nginx third-party module installation procedure.
From the nginx source directory:  
./configure --add-module=/path/to/ModSecurity-nginx  
Or, to build a dynamic module:  
./configure --add-dynamic-module=/path/to/ModSecurity-nginx --with-compat
Note that when building a dynamic module, your nginx source version needs to match the version of nginx you're compiling this for.  
Further information about nginx third-party add-ons support are available here: http://wiki.nginx.org/3rdPartyModules  
./configure --add-module=/data/openresty-waf/ModSecurity-nginx

这里有两种方式安装, 一种是直接静态链接, 把模块编译到openresty中, 
还有一种是使用动态库链接的方式, 这里, 我们选择第二种, 因为:

> Dynamic modules – NGINX 1.11.5 was a milestone moment in the development of NGINX. It introduced
 the new --with-compat option which allows any module to be compiled and dynamically loaded into a 
running NGINX instance of the same version (and an NGINX Plus release based on that version). 
There are over 120 modules for NGINX contributed by the open source community, and now you can load 
them into our NGINX builds, or those of an OS vendor, without having to compile NGINX from source. 
For more information on compiling dynamic modules, see Compiling Dynamic Modules for NGINX Plus.

先安装一些乱七八糟的库,然后再编译modesecurity_module
```bash
sudo apt-get install apache2-dev autoconf automake build-essential bzip2 \
    checkinstall devscripts flex g++ gcc git graphicsmagick-imagemagick-compat \ 
    graphicsmagick-libmagick-dev-compat libaio-dev libaio1 libass-dev \
    libatomic-ops-dev libavcodec-dev libavdevice-dev libavfilter-dev \
    libavformat-dev libavutil-dev libbz2-dev libcurl4-openssl-dev libfaac-dev \ 
    libfreetype6-dev libgd-dev libgeoip-dev libgeoip1 libgif-dev libgpac-dev  \
    libgsm1-dev libjack-jackd2-dev libjpeg-dev libjpeg-progs libjpeg8-dev  \
    liblmdb-dev libmp3lame-dev libncurses5-dev libopencore-amrnb-dev \ 
    libopencore-amrwb-dev libpam0g-dev libpcre3 libpcre3-dev libperl-dev \ 
    libreadline-dev librtmp-dev libsdl1.2-dev libssl-dev libswscale-dev \
    libtheora-dev libtiff5-dev libtool libva-dev libvdpau-dev libvorbis-dev \
    libxml2-dev libxslt-dev libxslt1-dev libxslt1.1 libxvidcore-dev libxvidcore4 \
    libyajl-dev make openssl perl pkg-config tar texi2html unzip zip \
    zlib1g-dev
cd openresty-1.15.8.3/
./configure --with-compat --add-dynamic-module=/data/openresty-waf/ModSecurity-nginx
输出结果:
-------------------------------
Configuration summary
  + using system PCRE library
  + using system OpenSSL library
  + using system zlib library

  nginx path prefix: "/usr/local/openresty/nginx"
  nginx binary file: "/usr/local/openresty/nginx/sbin/nginx"
  nginx modules path: "/usr/local/openresty/nginx/modules"
  nginx configuration prefix: "/usr/local/openresty/nginx/conf"
  nginx configuration file: "/usr/local/openresty/nginx/conf/nginx.conf"
  nginx pid file: "/usr/local/openresty/nginx/logs/nginx.pid"
  nginx error log file: "/usr/local/openresty/nginx/logs/error.log"
  nginx http access log file: "/usr/local/openresty/nginx/logs/access.log"
  nginx http client request body temporary files: "client_body_temp"
  nginx http proxy temporary files: "proxy_temp"
  nginx http fastcgi temporary files: "fastcgi_temp"
  nginx http uwsgi temporary files: "uwsgi_temp"
  nginx http scgi temporary files: "scgi_temp"
-------------------------------
ln -s /usr/bin/make /usr/bin/gmake
gmake
gmake install
```


一些其它的问题:
sudo命令:
https://www.itread01.com/p/1383143.html
apt-get install sudo -y
安装网络工具命令:
apt-get install -y net-tools
Modsec WAF on Openresty & Ubuntu
https://medium.com/@rrohitrockss/modsec-waf-on-openresty-ubuntu-e0125dd2301b

参考资料汇总:
    openresty+waf安装过程:
    https://medium.com/@rrohitrockss/modsec-waf-on-openresty-ubuntu-e0125dd2301b
    openresty+WAF配置流程参考:
    https://www.cnblogs.com/Hi-blog/p/OpenResty-ModSecurity.html
    牛逼的第三方nginx模块:
    https://www.nginx.com/resources/wiki/modules/index.html
