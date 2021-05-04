# 笔记

## The Shell

`drwxr-xr-x`。首先，本行第一个字符`d`表示是一个目录。然后接下来的九个字符，每三个字符构成一组。它们分别代表了文件所有者，用户组以及其他所有人具有的权限。其中 `-`表示该用户不具备相应的权限。从上面的信息来看，只有文件所有者可以修改`(w)`文件夹（例如，添加或删除文件夹中的文件）。为了进入某个文件夹，用户需要具备该文件夹以及其父文件夹的“搜索”权限（以“可执行”：`x`）权限表示。为了列出它的包含的内容，用户必须对该文件夹具备读权限`(r)`。对于文件来说，权限的意义也是类似的。

```bash
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

```bash
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

```bash
# 课后练习
touch semester
echo '#!/bin/sh' > semester
echo 'curl --head --silent https://www.baidu.com' >> semester
chmod u+x ./semester
./semester
./semester | grep --ignore-case last-modified > last-modified.txt
```

## Shell工具和脚本

创建命令流程（pipelines）、将结果保存到文件、从标准输入中读取输入，这些都是shell脚本中的原生操作，这让它比通用的脚本语言更易用。

```bash
foo=bar
echo $foo # bar
echo "$foo" # bar
echo '$foo' # $foo
```

Bash中的字符串通过'和"分隔符来定义，但是它们的含义并不相同。以'定义的字符串为原义字符串，其中的变量不会被转义，而"定义的字符串会将变量值进行替换。

```bash
mcd () {
    mkdir -p "$1"
    cd "$1"
}
```

```bash
vim mcd.sh
# 激活脚本
source mcd.sh
# 调用脚本中定义的mcd函数，`test`将作为第一个参数`$1`传入
mcd test
mkdir /mnt/new
# 写了一大段命令但是权限不足时，可以使用`!!`重复上一条命令
sudo !!

mkdir test
rmdir test
# `$_`获取上一条命令的最后一个参数，如果是在交互式shell中，可以按下ESC键加逗号键获取这个值
mkdir $_

echo "hello"
# 获取前一个命令的返回码或退出状态，返回0表示正常执行，其它非0返回值都表示有错误发生
echo $? # 0

grep foobar mcd.sh
echo $? # 1

true
echo $? # 0
false
echo $? # 1

false || echo "Oops, fail"
true || echo "Will not be printed"
true && echo "Things went well"
false && echo "Will not be printed"
# 多个命令可以使用分号进行分隔
false ; echo "This will always run"
```
