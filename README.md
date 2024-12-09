# 分支介绍

nekobox通常的配置是发起真实dns请求获取真实ip，但对于需要解锁服务来说就比较蛋疼了。

[GitHub Releases 下载](https://github.com/zizzdog/NekoBoxForAndroid/releases)

## 原本的流程
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
首先开启设置中的`启用FakenDNS`，然后在路由中添加规则，domain以fakedns:为首行，之后填写需要使用fakedns的域名或规则集，把outbound设置为代理，这样就会生成对应的dns规则发送给fakedns服务。
![设置示例](%E8%AE%BE%E7%BD%AE%E7%A4%BA%E4%BE%8B.png)

## 生成配置
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
        "inbound": [
          "tun-in"
        ],
        "rule_set": [
          "geosite:openai"
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

## 效果：
![效果](%E6%95%88%E6%9E%9C.png)

# NekoBox for Android

[![API](https://img.shields.io/badge/API-21%2B-brightgreen.svg?style=flat)](https://android-arsenal.com/api?level=21)
[![Releases](https://img.shields.io/github/v/release/MatsuriDayo/NekoBoxForAndroid)](https://github.com/MatsuriDayo/NekoBoxForAndroid/releases)
[![License: GPL-3.0](https://img.shields.io/badge/license-GPL--3.0-orange.svg)](https://www.gnu.org/licenses/gpl-3.0)

sing-box / universal proxy toolchain for Android.

一款使用 sing-box 的 Android 通用代理软件.

## 下载 / Downloads

[![GitHub All Releases](https://img.shields.io/github/downloads/Matsuridayo/NekoBoxForAndroid/total?label=downloads-total&logo=github&style=flat-square)](https://github.com/Matsuridayo/NekoBoxForAndroid/releases)

[GitHub Releases 下载](https://github.com/Matsuridayo/NekoBoxForAndroid/releases)

**Google Play 版本自 2024 年 5 月起已被第三方控制，为非开源版本，请不要下载。**

**The Google Play version has been controlled by a third party since May 2024 and is a non-open source version. Please do not download it.**

## 更新日志 & Telegram 发布频道 / Changelog & Telegram Channel

https://t.me/Matsuridayo

## 项目主页 & 文档 / Homepage & Documents

https://matsuridayo.github.io

## 支持的代理协议 / Supported Proxy Protocols

* SOCKS (4/4a/5)
* HTTP(S)
* SSH
* Shadowsocks
* VMess
* VLESS
* WireGuard
* Trojan
* Trojan-Go (trojan-go-plugin)
* NaïveProxy (naive-plugin)
* Hysteria (hysteria-plugin)
* Mieru (mieru-plugin)
* TUIC

请到[这里](https://matsuridayo.github.io/m-plugin/)下载插件以获得完整的代理支持.

Please visit [here](https://matsuridayo.github.io/m-plugin/) to download plugins for full proxy supports.

## 支持的订阅格式 / Supported Subscription Format

* 原始格式: 一些广泛使用的格式 (如 Shadowsocks, Clash 和 v2rayN)
* Raw: some widely used formats (like Shadowsocks, Clash and v2rayN)

## 捐助 / Donate

<details>

如果这个项目对您有帮助, 可以通过捐赠的方式帮助我们维持这个项目.

捐赠满等额 50 USD 可以在「[捐赠榜](https://mtrdnt.pages.dev/donation_list)」显示头像, 如果您未被添加到这里, 欢迎联系我们补充.

Donations of 50 USD or more can display your avatar on the [Donation List](https://mtrdnt.pages.dev/donation_list). If you are not added here, please contact us to add it.

USDT TRC20

`TRhnA7SXE5Sap5gSG3ijxRmdYFiD4KRhPs`

XMR

`49bwESYQjoRL3xmvTcjZKHEKaiGywjLYVQJMUv79bXonGiyDCs8AzE3KiGW2ytTybBCpWJUvov8SjZZEGg66a4e59GXa6k5`

</details>

## Credits

Core:
- [SagerNet/sing-box](https://github.com/SagerNet/sing-box)
- [Matsuridayo/sing-box-extra](https://github.com/MatsuriDayo/sing-box-extra)

Android GUI:
- [shadowsocks/shadowsocks-android](https://github.com/shadowsocks/shadowsocks-android)
- [SagerNet/SagerNet](https://github.com/SagerNet/SagerNet)
- [Matsuridayo/Matsuri](https://github.com/MatsuriDayo/Matsuri)

Web Dashboard:
- [Yacd-meta](https://github.com/MetaCubeX/Yacd-meta)
