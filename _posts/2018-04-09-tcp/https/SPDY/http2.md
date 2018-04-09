---
published: false
---
# 前言

之前准备了下面试，发现网络这块几乎都会问http过程,tcp握手过程以及https中的ssl/tls过程，最近随着http2的普及，http2也成为了必问的内容之一，原来不清楚https这块，所以就去学习了一下，发现其实https就这一个过程可以囊括了http,tcp三握四挥，ssl/tls加密验证这些知识点

# tcp三握四挥

## three-way handshake

## four-way handshake

# SSL/TLS
tls是新一代的ssl，可以这么理解
CA
Asymmetric key
symmetric key

# http2
要说http2之前，首先要说说SPDY(speedy缩写)，因为http2就是从SPDY改进来的，SPDY是Google的产物，为了减少网络传输延迟而开发的协议，相比http1.1本质上是在tcp和http之间加了一层SPDY层，这一层把原本http1.1文本传输变为了二进制帧，这一改进是其他新特性的基础。

SPDY相对于http1.1的新特性:
1.无需FIFO的多路复用(Multiplexing);
2.二进制分帧;
3.强制压缩(包括headers);
4.优先级排序(Priority);
5.双向通讯;
6.服务器推送;

而从SPDY到http2的话，最显著的就是头部压缩算法的改变，从dynamic stream-based compression algorithm到fixed huffman code-based compression algorithm，为了防止Compression oracle attack而做的改进，可以说是安全性的改进

值得注意的是，虽然http2在协议上没有要求https，但是在SPDY时代，就已经默认使用https，所以现实中Chrome,Firefox等主流的浏览器一般都是默认http2必须要在ssl/tls上。
