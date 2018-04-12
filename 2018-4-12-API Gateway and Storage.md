---
layout: post
title: 2018-4-12-API Gateway and Storage
date: 2018-4-12 13:27:55 +0754
categories: 未分类
tags: API_Gateway  Storage
description: 简要说明。
---

API Gateway使用应用服务的最上层，直接为用户提供服务；    
而Storage则处于应用和系统之间，为服务应用提供数据存储功能，在Netflix中他们所处的层次关系如图所示。    
![enter description here](https://www.github.com/yujinyu/markdown/raw/master/images/2018-4-12-API Gateway and Storage/v2-d364f3f5ac1d46d0ea37bab7f99d680c_r.jpg)

 1. 问题两者是否可以联系起来？   
 2. 如何将两者进行结合？      




ps. 依据客户的需求（网络带宽、存储容量等）使用API Gateway来限制或者配置后端服务应用的存储、网络等参数。