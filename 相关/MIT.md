# 笔记

## The Shell

`drwxr-xr-x`。首先，本行第一个字符`d`表示是一个目录。然后接下来的九个字符，每三个字符构成一组。它们分别代表了文件所有者，用户组以及其他所有人具有的权限。其中 `-`表示该用户不具备相应的权限。从上面的信息来看，只有文件所有者可以修改`(w)`文件夹（例如，添加或删除文件夹中的文件）。为了进入某个文件夹，用户需要具备该文件夹以及其父文件夹的“搜索”权限（以“可执行”：`x`）权限表示。为了列出它的包含的内容，用户必须对该文件夹具备读权限`(r)`。对于文件来说，权限的意义也是类似的。

```shell
# 将echo程序输出流从终端重定向到hello.txt
echo hello > hello.txt
# 将hello.txt作为cat程序的输入流
cat < hello.txt
# 将hello.txt作为cat程序的输入流，将hello2.txt作为cat程序的输出流
cat < hello.txt > hello2.txt
# 将字符串world追加到hello.txt中，会启用新的一行进行追加
echo world >> hello.txt
# |操作符允许将一个程序的输出和另外一个程序的输入连接起来，此处将输出`ls -l /`的最后一行内容
ls -l / | tail -n1
# 将返回的HTTP标头中的`Content-Length: 81`内容提取出来，然后用` `进行分隔，输出第二个区域的内容
curl --head --silent baidu.com | grep --ignore-case content-length | cut --delimiter=' ' -f2
```

```shell
# 为了在/sys目录下查找，我们使用`sudo`启用根用户进行操作，查找时使用到了`find`命令自身的一些参数
sudo find -L /sys/class/backlight -maxdepth 2 -name '*brightness*'
# 但是当我尝试使用echo程序向某个内核文件写入时，遇到了权限禁止的问题
sudo echo 3 > brightness
# 这是因为|、>、<等操作符是通过shell执行的，而不是被各个程序单独执行，echo等程序并不知道这些操作符的存在
# 对于上面的情况，shell在设置`sudo echo`前尝试打开brightness文件并写入，但是被系统拒绝
# 因为打开brightness文件时的shell不是根用户
# 可以理解为我们用根用户操作了echo程序，但echo程序并没有将根用户的权限传递给brightness文件
# 明白了这一点后，将上述操作转化为下述操作，此时打开brightness文件的是tee这个程序
# 并且这个程序以root权限在运行，可以成功的进行打开操作，然后将echo程序的输出流重定向为该文件
echo 3 | sudo tee brightness
```

```shell
# 课后练习
touch semester
echo '#!/bin/sh' > semester
echo 'curl --head --silent https://www.baidu.com' >> semester
chmod u+x ./semester
./semester
./semester | grep --ignore-case last-modified > last-modified.txt
```
