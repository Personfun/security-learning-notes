# 04 - CS 流量特征分析

## 一、 抓到的是什么
GET /dpixel HTTP/1.1 是 Beacon 的心跳包。

## 二、 逐字段解读
- URI /dpixel：伪装成像素图请求
- Host IP:端口：默认不伪装
- User-Agent IE 7.0：默认伪装
- Cookie 超长：加密的 C2 数据藏在这里

## 三、 蓝队检测点
- URI 固定
- User-Agent 过时
- Cookie 长度异常
- 回连节律规律
- Host 是 IP

## 四、 Malleable C2 Profile 的作用
定制这些特征，让流量像正常业务。
