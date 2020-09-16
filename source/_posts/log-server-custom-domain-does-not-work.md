---
title: aliyun log service custom domain does not work
date: 2020-09-16 14:21:51
tags:
- Aliyun
- 技巧
---

### 1. problem
The code does not work when you want to use aliyun sdk to send log to aliyun log server with your custom domain.
After review the nodejs sdk source code, I find that the api domain is built:
```js
var hostname = req.httpRequest.endpoint.hostname;

var projectName = req.params['projectName'];
var host = projectName+'.'+ hostname;


if(!/^[0-9.]+$/.test(hostname)){
  //不是ip,  是域名, 则需要拼接project名
  var protocol = req.httpRequest.endpoint.protocol;
  var port = req.httpRequest.endpoint.port;

  //real endpoint
  var endpointObj = parseURL(protocol+'//'+host+':'+port);

  ALY.util.update(req.httpRequest, {endpoint: endpointObj });
  // ALY.util.update(req.service, {endpoint: endpointObj });
}
```
**The endpoint also has the project name prefix**. This will result a error address resolution.

### 2. solution

add two address resolution rule:

|machine record| record type | record value|
|---|---|---|
|@	|CNAME |cn-beijing.log.aliyuncs.com|
|*projectName*|	CNAME  |*projectName*.cn-beijing.log.aliyuncs.com|

NOTE: `cn-beijing.log.aliyuncs.com`: the domain we need to replace with custom domain.
In the second rule, the project name acts the subdomain, the resolved address is the aliyun origiinal log sevice domain. If you use the `nslookup` to test the: *projectName*.custom.com, we can find that it is ok to resolve to right address.

### 3. test and verify
Before testing, you sould change the endpoint from `cn-beijing.log.aliyuncs.com` to your custom domain for example: `jimmy.cn` in your code. I have tested below three sdk:
- [nodejs sdk](https://github.com/aliyun-UED/aliyun-sdk-js)
- [python sdk](https://github.com/aliyun/aliyun-log-python-sdk)(loging base on the python logging)
- [web trancking](https://help.aliyun.com/document_detail/31752.html)

After change the endpoint from aliyun domain to custom domain, all my code works to send log to aliyun log service, so the above solution is ok.