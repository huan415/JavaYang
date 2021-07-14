# nginx实现缩略图

## 两种实现方式（两个模块）

1. --with-http_image_filter_module

   自己在nginx.conf配置截取长宽

2. ngx_image_thumb

​       nginx.conf配置开关



### with-http_image_filter_module

1. ```shell
   yum -y install gd-devel
   ```

2. ```shell
   ningx -V
   ./configure (上一步nginx -V查出的模块) --with-http_image_filter_module
   ```

   注意：安装模块前要用nginx -V将已安装好的模块查出来，安装时都要带上，否则会被覆盖

3. ```shell
   mv /usr/sbin/nginx /usr/sbin/nginxbak
   ```

   备份

4. ```shell
   cp /root/nginx-1.18.0/objs/nginx /usr/sbin/
   ```

   复制编译完的文件

5. ```shell
   service nginx stop
   service nginx start
   ```

   重启，注意不是：nginx -s reload

6. ```nginx
   location ~ ^/home/images/(.+)\.(jpg|gif|png|ioc|jpeg)!(\d+)_(\d+)$ {
           autoindex on;
           set $w $3;
           set $h $4;
           rewrite /images/(.+)\.(jpg|gif|png|ioc|jpeg)!(\d+)_(\d+)$  /images/$1.$2 break;
           image_filter resize $w $h;
           image_filter_buffer 10M;
           root /home/;
          } 
   ```

   或者

   ```nginx
   location ^~ /home/images/ {
             alias /home/images/;
             if ($arg_attname ~ "^(.*).apk") {
              add_header Content-Disposition "attachment;filename=$arg_attname";
             }
             if ($request_uri ~* ^/home/images/(.+)\.(jpg|gif|png|ioc|jpeg)!(\d+)_(\d+)$) {
               rewrite /home/images/(.+)\.(jpg|gif|png|ioc|jpeg)!(\d+)_(\d+)$  /home/images2/$1.$2!$3_$4 last;
               root /home/;
             }
         }
   
   location ~ ^/home/images2/(.+)\.(jpg|gif|png|ioc|jpeg)!(\d+)_(\d+)$ {
           autoindex on;
           set $w $3;
           set $h $4;
           rewrite /images2/(.+)\.(jpg|gif|png|ioc|jpeg)!(\d+)_(\d+)$  /images/$1.$2 break;
           image_filter resize $w $h;
           image_filter_buffer 10M;
           root /home/;
        }
   ```

   

7. 访问图片http://127.0.0.1/home/images/test.jpg!c300x200.jpg
   其中!c300x200.jpg可在nginx.conf随意配置



### ngx_image_thumb

参考[github ngx_image_thumb](https://github.com/oupula/ngx_image_thumb)

1. ```shell
   sudo yum install gd-devel pcre-devel libcurl-devel 
   ```

2. ```shell
   wget http://nginx.org/download/nginx-1.4.0.tar.gz
   tar -zxvf nginx-1.4.0.tar.gz
   cd nginx-1.4.0
   ```

3. ```shell
   location / {
      image on;
      image_output on;
   }
   ```

   nginx.conf的location配置开关

4. 访问图片http://127.0.0.1/home/images/test.png!c300x200.png



## 压测比较

结果：with-http_image_filter_module cpu升高一点

​            ngx_image_thumb   有指定大小时  cpu快速升高

​                                                没有指定大小时 cpu升高一点

### with-http_image_filter_module

![1](E:\materis\自己整理的\markdown\nginx实现缩略图\1.png)



###    ngx_image_thumb

![2](E:\materis\自己整理的\markdown\nginx实现缩略图\2.png)



![3](E:\materis\自己整理的\markdown\nginx实现缩略图\3.png)
