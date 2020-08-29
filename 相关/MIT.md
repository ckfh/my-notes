# 笔记

## The Shell

```shell
# 将echo程序输出流从终端重定向到hello.txt
echo hello > hello.txt
# 将hello.txt作为cat程序的输入流
cat < hello.txt
# 将hello.txt作为cat程序的输入流，将hello2.txt作为cat程序的输出流
cat < hello.txt > hello2.txt
# 将字符串world追加到hello.txt中，会启用新的一行进行追加
echo world >> hello.txt
ls -l / | tail -n1
```
