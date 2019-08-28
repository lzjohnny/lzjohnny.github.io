---
title: Nginx 与网站问题排查
date: 2019-02-06 22:19:08
tags:
---
## 一、问题描述：

线上 A 站点所有 URL 使用 https 访问反复刷新时，有时报错有时正常，http 访问始终正常。
## 二、分析步骤：

##### web机器排查

1. 根据现象：不针对特定URL，全部URL均可能会报错
判断不是由业务代码引发的，需重点排查服务器关键进程、框架配置、入口文件等基础服务。
2. 基础服务排查
检查 Nginx、PHP-FPM、MySQL 进程正常。
检查 PHP-FPM 错误日志始终正常。
检查 Nginx access 日志始终正常。
3. 根据现象：错误不稳定出现
日志正常且错误不稳定出现，判断可能有的机器运行正常，有的机器报错。
4. 打开 A 站点对应的全部机器的 Nginx 日志观察
注意到页面访问正常时，Nginx 对应日志正常，页面报错时，请求无对应日志。判断可能是 web 机的上层负载均衡配置有误，导致请求被转发到其他无关机器上。

##### 网关机器排查

1. 检查上层网关机器 Nginx 日志
网关之下对应 A、B、C、D 等多个 web 站点，观察 A 站点 Nginx 访问日志发现 http 请求有日志记录，但 https 请求无日志记录，怀疑 https 请求可能由于 DNS 配置原因未经过网关机器。
2. 确认请求流程
重新检查 web 站点 Nginx 日志，观察来源 IP，发现请求确实是来自于网关机器，此时陷入僵局。
（web机器有访问日志，且请求来源于网关机器，但网关机器又观察不到访问日志）
3. 再次确认请求流程
既然 web 机器日志记录了请求来自于网关机器，则检查网关所有站点 Nginx 访问日志（之前只观察A站点的访问日志），结果在另外毫不相干的站点的访问日志中发现该请求。
就是说：浏览器访问 A 域名，在网关层 A.access.log 日志无记录，却在 B.access.log 有记录，并且 A、B 站点业务上毫无关系。这里应该就是问题的突破口。
4. 检查网关 A、B 域名对应的 Nginx 配置
发现 A 域名只配置监听 80 端口，而 B 域名配置了 80 和 443 端口，这就是问题所在：
**Nginx 配置中 server_name 和 listen 端口不能同时匹配时，会优先匹配端口而忽略 server_name，故使用 https 访问A站点时，由于没有完全匹配的配置，实际使用了 B 站点的 Nginx 配置。**

具体来说，A 站点是 admin.example.com，B 站点是 qnstatic.example.com，C 站点是 wxapi.example.com，查看 B 的负载均衡规则：
```
server {
    listen       443 ssl;
    server_name  qnstatic.example.com;
    charset utf-8;
    access_log  /var/log/nginx/qnstatic.example.com.access.log main;
        location / {
                if ($http_user_agent  ~* (spider))  { proxy_pass  http://spider.example.com; }
                proxy_pass              http://wxapi.example.com;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header REMOTE-HOST $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
}
```

访问 admin.example.com:443 时，优先使用端口匹配到上述配置（qnstatic.example.com 站点的配置），请求由 proxy_pass 到 wxapi.example.com 站点，会优先使用 upstream 中 wxapi.example.com 域名的配置来解析成 IP，如没有相应配置会使用 DNS 解析 wxapi.example.com 域名

```
upstream wxapi.example.com {
        server 172.16.16.97:80 weight=5 max_fails=2 fail_timeout=2;
        server 172.16.16.98:80 weight=5 max_fails=2 fail_timeout=2;
        server 172.16.16.95:80 weight=5 max_fails=2 fail_timeout=2;
        server 172.16.16.96:80 weight=5 max_fails=2 fail_timeout=2;
        server 172.16.16.79:80 weight=5 max_fails=2 fail_timeout=2;
}
```

所以请求最终会被分发到 wxapi 站点对应的web机器上，恰巧 wxapi 站点对应的5台服务器中，3台也同时被 admin 站点共用。如果请求碰巧转发到共用的机器上，则一切正常，否则则会报错。这就是使用 https 访问反复刷新时，有时报错有时正常的原因。

## 三、排查结论
概括来说，浏览器访问 A 域名，网关层 Nginx 匹配到 B 域名的配置，proxy_pass 到 C 域名，分发到了 C 域名下的 web 服务器。

另外补充说明，该请求到达 C 的 web 服务器后，在该服务器的 Nginx 中仍然会用原始的 A 域名去匹配 server_name，而非 proxy_pass 配置的 C 域名！C 域名仅仅用来确定转发的目的机器，不起到匹配作用。