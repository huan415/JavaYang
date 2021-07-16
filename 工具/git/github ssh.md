# github ssh

## git 命令ssh-keygen生成密钥

1. 删除文件 C:\Users\huan415\\.ssh

2. ssh-keygen -t rsa -C "435826111@qq.com"

3. cat  ~/.ssh/id_rsa.pub

   ![ssh_1](D:\project\huan415\http\JavaYang\工具\git\images\ssh_1.png)

4. 添加id_rsa.pub到GitHub

   ![ssh_9](D:\project\huan415\http\JavaYang\工具\git\images\ssh_9.jpg)

5. ssh -T git@github.com

   ![ssh_2](D:\project\huan415\http\JavaYang\工具\git\images\ssh_2.png)

6. git clone git@github.com:huan415/JavaYang.git
   ![image-20210716175205157](C:\Users\huan415\AppData\Roaming\Typora\typora-user-images\image-20210716175205157.png)

注意：此时只能用命令窗口，不能用TortoiseGit，因为TortoiseGit用的密钥是PuTTYgen生成以.ppk结尾的密钥，而不是ssh-keygen生成的rsa密钥

## TortoiseGit  PuTTYgen生成以.ppk结尾的密钥

1. PuTTYgen生成密钥   注意鼠标要在窗口在一直移动，否则进度条不会走
   ![ssh_3](D:\project\huan415\http\JavaYang\工具\git\images\ssh_3.png)

![ssh_4](D:\project\huan415\http\JavaYang\工具\git\images\ssh_4.png)

2. 复制key, 添加到Github
   
   ![ssh_9](D:\project\huan415\http\JavaYang\工具\git\images\ssh_9.jpg)
   
3. 保存成私钥，扩展名为.ppk

   ![ssh_5](D:\project\huan415\http\JavaYang\工具\git\images\ssh_5.png)

4. Pageant添加私钥，.ppk文件

   ![ssh_6](D:\project\huan415\http\JavaYang\工具\git\images\ssh_6.png)

   ![ssh_7](D:\project\huan415\http\JavaYang\工具\git\images\ssh_7.png)

## 以上两种结合，ssh-keygen生成密钥，PuTTYgen加工生成以.ppk结尾的密钥

 前提： 已经利用git 命令生成公钥和密钥

1. 选择用命令生成的密钥，并用PuTTYKey生成扩展名为.ppk的文件

![ssh_8](D:\project\huan415\http\JavaYang\工具\git\images\ssh_8.png)



2.  重复上面tortoiseGit中的第四步（Pageant添加私钥，.ppk文件）