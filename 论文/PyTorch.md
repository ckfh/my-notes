# 笔记

## 设定每个应用程序的可见设备

CUDA_VISIBLE_DEVICES

## 训练次数

GoogleNet：1GPU训练，1次100，500次；4GPU训练，1个GPU100，125次。每step一次就需要通信同步参数。
