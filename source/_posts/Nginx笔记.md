---
title: Nginx笔记
date: 2020-12-02 17:10:16
tags:
	- Nginx
categories:
	- Nginx

---

### 优势

1. IO多路复用
   * 多个描述符的I/O操作都能在一个线程内并发交替地顺序完成，这里的“复用”指的是复用同一个线程。
   * select、poll、epoll
2. 轻量级
3. CPU亲和（afinity）
   * CPU核心和nginx work进程绑定方式，把work进行固定在一个cpu上执行，减少切换cpu的cache miss，获得更好的性能。
4. sendfile
   * 零拷贝



### 安装目录

| 路径                                                         | 类型           | 作用                                       |
| ------------------------------------------------------------ | -------------- | ------------------------------------------ |
| /etc/logrotate.d/nginx                                       | 配置文件       | Nginx日志轮转，用于logrotate服务的日志切割 |
| /etc/nginx<br />/etc/nginx/nginx.conf<br />/etc/nginx/conf.d<br />/etc/nginx/conf.d/default.conf | 目录、配置文件 | 主配置文件<br />                           |
|                                                              |                |                                            |
|                                                              |                |                                            |
|                                                              |                |                                            |
|                                                              |                |                                            |
|                                                              |                |                                            |
|                                                              |                |                                            |
|                                                              |                |                                            |

