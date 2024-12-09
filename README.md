# 分支介绍

[GitHub Releases 下载](https://github.com/zizzdog/NekoBoxForAndroid/releases)

俗话说玩dns容易被dns玩死，路由规则domain支持如下配置：
+ `dns:direct` 指定规则内域名使用直连DNS解析
+ `dns:remote` 指定规则内域名使用远程DNS解析
+ `dns:block` 指定规则内域名禁止解析
+ `dns:fakedns` 指定规则内域名使用FakeDNS解析

设置如上配置后dns规则就不会遵循outbound设置的规则：

```
dns:direct
domain:google.com

outbound选择代理
```
那么最终访问google时候，会使用直连DNS解析，而不是远程dns

若开头添加!号如：`!dns:remote`，表示该规则仅作为dns规则，不生成相关的路由规则

已知问题：规则的geosite不能重复，即使一条是普通规则，另一条是仅作用于dns的规则

# 解决dns解锁的思路
## 原本的流程

nekobox通常的配置是发起真实dns请求获取真实ip，但对于需要解锁服务来说就比较蛋疼了。

比如说香港节点访问`chatgpt.com`，gpt不开放香港地区所以一般机场都会提供解锁服务
1. 浏览器发起dns请求解析`chatgpt.com`
2. 软件检测到国外域名请求发给`1.1.1.1`的dns
3. `1.1.1.1`是国外ip请求发给机场。
4. 机场返回`1.1.1.1`给的`chatgpt.com`在香港访问的ip，gpt不开放香港地区，所以返回的ip是无用的。
5. 浏览器获得`chatgpt.com`的无效ip并开始访问
6. 软件检测到是国外ip请求发给机场
7. 机场的香港节点直接帮你访问这个ip。

最后因为访问的ip是无效的所以访问失败。这时候如果开启nekobox设置中的`启用流量探测->探测结果用于目标地址`功能，第6步就会嗅探到该ip对应的域名是`chatgpt.com`，而不会发送ip而是发送域名给机场。机场收到域名进行一次dns解析并完成解锁。
但是这有个问题，即使不用做dns解锁的域名，前面本机解析了一次dns，又发给机场又进行一次解析，进行两次dns解析这样延迟我认为太高了。

## 利用fakedns实现特定域名不解析
可以利用singbox的dns分流方法配合fakedns服务器，而无需开启嗅探域名覆盖目标地址功能。具体的做法就是添加一个fakedns的上游服务器，把需要进行dns解锁的域名走这个服务器解析，然后流程就会变成这样：
1. 浏览器发起dns请求解析`chatgpt.com`
2. 软件检测到走fakedns，直接返回了一个fakeip假设为`198.18.0.1`
3. 浏览器获得`chatgpt.com`的ip：`198.18.0.1`并开始访问
4. 软件检测到是`198.18.0.1`，映射回真实域名`chatgpt.com`并发送请求给机场
5. 机场顺利执行dns解锁并访问


此分支修改了fakedns的选项机制。原本设置中的`启用FakenDNS`开启后会接管所有没有匹配到路由规则的域名，返回虚拟ip地址。个人修改后，开启此选项只会生成一个fakedns的上游dns服务器而不劫持所有dns请求。
## 配置示例
首先开启设置中的`启用FakenDNS`，然后在路由中添加规则，domain以dns:fakedns为首行，之后填写需要使用fakedns的域名或规则集，这样就会生成对应的dns规则发送给fakedns服务。
![设置示例](%E8%AE%BE%E7%BD%AE%E7%A4%BA%E4%BE%8B.png)

## 生成配置如下
```json
{
  "dns": {
    "fakeip": {
      "enabled": true,
      "inet4_range": "198.18.0.0/15",
      "inet6_range": "fc00::/18"
    },
    "independent_cache": true,
    "rules": [
      {
        "disable_cache": true,
        "domain": [
          "www.google.com"
        ],
        "domain_suffix": [
          "netflix.com"
        ],
        "rule_set": [
          "geosite:openai"
        ],
        "inbound": [
          "tun-in"
        ],
        "server": "dns-fake"
      }
      ...
    ],
    "servers": [
      {
        "address": "fakeip",
        "strategy": "ipv4_only",
        "tag": "dns-fake"
      }
      ...
    ]
  }
  ...
}
```
## 对某应用进行解锁
对应用的所有请求都应用fakedns服务，可以使用如下配置然后选择应用
```
dns:fakedns
regexp:.*
```
![对应用解锁](%E5%BA%94%E7%94%A8%E8%A7%A3%E9%94%81%E7%A4%BA%E4%BE%8B.png)

## 原本的`启用FakeDNS`功能
使用如下配置且不选择任何应用
```
!dns:fakedns
regexp:.*
```

## 其他用法请自行探索