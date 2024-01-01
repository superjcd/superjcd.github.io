---
title: 网络编程入门
date: 2024-01-01
categories:
    - 网络
tags:
    - 阅读笔记
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# 网络编程入门
同时也算是[beej‘s guide to network programming](https://beej.us/guide/bgnet/html/split/index.html)， 
当然这里不一定会按章原书的章节原封不动的来， 会穿插一些个人见解（囿于水平有限， 无法保证都是对的）

## 前置知识

### 什么是socket
在unix系统中， 所有的一切都是文件，这些文件会关联一个文件描述符（其实就是一个整数， 可以理解为这些文件的id）。  
当我们在unix系统中， 使用scoket()会返回一个文件描述符（文件id）, 然后我们通过send和recv对这个文件上进行了类似write和read的操作， 
以此来达到交换数据的目的

### socket类型
网络socket是有两种类型的：  
- stream
- dgram(DATAGRAM)
前者使用的是TCP协议(Transmission Control Protocol), TCP协议在传输过程中是要一直保证链接的， 而且数据包是有序传递的(使用telnet就是顺序传输信息的)且会是完整传递；  
相反地datagram使用的是UDP（User Datagram Protocol）， 数据不能保证是有序的， 甚至不能保证数据是完整的（当然相应的， udp的传输可以更快）。

>  有些文件传输软件， 比如tftp其实用的也是udp， 但是它在udp协议的基础上又添加了一层协议，这层协议会要求接收者受到信息后返回一个信号ACK,   
>  信息发送者只有收到这个信号才能表示发送信息成功， 不然信息会被重发

### Data Encapsulation
我们知道OSI（open system interconnect ）会有7个层级： 
- Application
- Presentation
- Session
- Transport
- Network
- Data Link
- Physical

每一个层级都会携带一些相应的信息，最开始用户发送的出去的信息可能就是一个简单的字符串， 这一串信息会被层层封包：  

![image](imgs/1.png)

越外层的封包信息， 越接近物理层， 为什么？因为数据的接收者， 肯定先是是物理层， 比如Ethernet， wifi这些靠近物理的layer， 最后层层解包， 拿到被封装在最里面的数据data