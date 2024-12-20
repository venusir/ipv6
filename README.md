### routeros开启ipv6

* [RouterOS IPv6 NAT配置，保护内网设备](https://www.youtube.com/watch?v=lrQ0HwRPOiY&t=663s)

* [RouterOS IPv4,IPv6防火墙配置](https://www.youtube.com/watch?v=NopQ50-Tq3A)

* [RouterOS配置家庭公网IPv6以及对IPv6地址分配原理浅析](https://blog.lonelyman.site/archives/jia-ting-gong-wang-ipv6-pei-zhi-yi-ji-dui-ipv6-de-zhi-fen-pei-yuan-li-qian-xi)

### routeros开启ddns

* [RouterOS 设置Cloudflare IPV6 DDNS only](https://velaciela.ms/routeros-using-cloudflare-update-ddns-ipv6-only)

* 脚本:

```
:local token "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx";
:local zoneId "yyyyyyyyyyyyyyyyyyyyyyyyy";
:local dnsId "zzzzzzzzzzzzzzzzzzzzzzzzzz";
:local domain "ipv6.example.com";

:local ip [/ipv6 dhcp-client get pppoe-out1 prefix];
:set $ip [:pick $ip 0 [:find $ip "/"]];
:local lastIp [/system script get ddns comment];


:local address "https://api.cloudflare.com/client/v4/zones/$zoneId/dns_records/$dnsId";
:local header "Authorization: Bearer $token,Content-Type:application/json";
:local body "{\"type\":\"AAAA\",\"name\":\"$domain\",\"content\":\"$ip\",\"ttl\":120,\"proxied\":false}";
:if ($lastIp != $ip) do={
    :put $address;    
    /tool fetch url=$address  http-method=put output=none http-header-field=$header http-data=$body;
    /system script set ddns comment=$ip;
}
```

* 获取Cloudflare域名的dns id

```
curl -X GET "https://api.cloudflare.com/client/v4/zones/域名ID/dns_records?page=1&per_page=20&order=type&direction=asc" \
-H "Authorization: Bearer 前面获取的 API 令牌" \
-H "Content-Type: application/json"
```


### Mikrotik RouterOS 路由器 CloudFlare DDNS 动态解析脚本(IPv4/IPv6)
> https://www.cloudrw.com/article/routeros-ddns-cloudflare

```
#########################################################################
#         ==================================================            #
#         $ Mikrotik RouterOS update script for CloudFlare $            #
#         ==================================================            #
#              Credits for Samuel Tegenfeldt, CC BY-SA 3.0              #
#                        Modified by kiler129                           #
#                        Modified by viritt                             #
#                        Modified by hscpro                             #
#########################################################################

################# 程序配置信息 #################
#调试信息 true/false
:local CFDebug "false"
#IPV4使用的接口
:global WANInterface4 "pppoe-out1"
#IPV6使用的接口
:global WANInterface6 "pppoe-out1"
#IPV6后缀（"::"解析到RouterOS;可填写局域网内LAN固定后缀解析到内网某个设备,）
:local LANipv6end ":xxxx:xxxx:xxxx:xxxx"
#TTL
:local CFttl "120"
#主域名
:local CFzone "hscbook.com"
#IPv4子域名
:local CFdomain "ipv4.hscbook.com"
:local CFdomainid "087dxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
#双栈IPv6域名ID
:local CFdomainid46 "8adcxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
#IPv6子域名 true/false
:local switchv6 "true"
:local CFdomain6 "ipv6.hscbook.com"
:local CFdomainid6 "f100xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
#CloudFlare账号与APIKEY
:local CFemail "xxxxx@xxxxxx.com"
:local CFtkn "101fbxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
:local CFzoneid "c25abxxxxcxxxxxxxxxxxxxxxxxxxxxx"

################# 内部变量 #################
#ipv4
:local currentIP ""
:local resolvedIP ""
:global WANip ""
#ipv6
:local currentIP6 ""
:local resolvedIP6 ""
:global WANip6 ""

################# 解析和设置IP变量 #################
#获取公网IPv4
:set currentIP [/ip address get [/ip address find interface=$WANInterface4 ] address];
:set WANip [:pick $currentIP 0 [:find $currentIP "/"]];
#获取域名IPv4
:set resolvedIP [:resolve $CFdomain];
#获取公网IPv6（DHCP方式）以及域名IPv6
:if ([/ipv6 dhcp-client get [find interface=$WANInterface6] status] = "bound") do={
    :if ([/ipv6 dhcp-client get [find interface=$WANInterface6 status=bound] prefix] != "true") do={
        :set currentIP6 [/ipv6 dhcp-client get [find interface=$WANInterface6 status=bound] prefix];
        #IPv6地址=公网IPv6前缀+设定的后缀
        :set WANip6 ([:pick $currentIP6 0 [:find $currentIP6 "::/"]] . $LANipv6end);
        :set resolvedIP6 [:resolve $CFdomain6];
    };
} else={
    :log info ("CF: 本机没有启用IPv6或配置不正确")
    :set switchv6 "false"
}
################# 生成 CloudFlare API 链接 (v4) #################
#IPv4
:local CFurl4 "<https://api.cloudflare.com/client/v4/zones/>"
:set CFurl4 ($CFurl4 . "$CFzoneid/dns_records/$CFdomainid");
#IPv6
:local CFurl46 "<https://api.cloudflare.com/client/v4/zones/>"
:local CFurl6 "<https://api.cloudflare.com/client/v4/zones/>"
:if ($switchv6 = "true") do={
    :set CFurl46 ($CFurl46 . "$CFzoneid/dns_records/$CFdomainid46");
    :set CFurl6 ($CFurl6 . "$CFzoneid/dns_records/$CFdomainid6");
};

################# 将调试信息写入日志 #################
:if ($CFDebug = "true") do={
    :log info ("CF: 调试模式打开")
    :log info ("CF: 解析域名 $CFdomain")
    :log info ("CF: 域名解析IPv4 $resolvedIP")
    :log info ("CF: 当前公网IPv4 $WANip")
    :log info ("CF: 使用的API地址v4 $CFurl4&content=$WANip")
    :if ($switchv6 = "true") do={
        :log info ("CF: 域名解析IPv6 $resolvedIP6")
        :log info ("CF: 当前公网IPv6 $WANip6")
        :log info ("CF: 使用的API地址v6 $CFurl6&content=$WANip")
    };
    :put "Get CFdomainid: curl -X GET \\"<https://api.cloudflare.com/client/v4/zones/$CFzoneid/dns_records\\>" -H \\"X-Auth-Email: $CFemail\\" -H \\"X-Auth-Key: $CFtkn\\" -H \\"Content-Type: application/json\\" | python -mjson.tool"
};

################# IPv4比较和更新域名记录 #################
:if ($resolvedIP != $WANip) do={
    :log info ("CF: 正在更新 IPv4 解析地址 $CFdomain = $WANip")
    /tool fetch http-method=put mode=https url="$CFurl4" http-header-field="X-Auth-Email:$CFemail,X-Auth-Key:$CFtkn,content-type:application/json" as-value output=user http-data="{\\"type\\":\\"A\\",\\"name\\":\\"$CFdomain\\",\\"content\\":\\"$WANip\\",\\"ttl\\":$CFttl,\\"proxied\\":false}"
    #/ip dns cache flush 执行间隔时大于TTS一倍可免于清理dns（TTS120->5m TTS300->10m）
} else={
    :log info "CF: IPv4公网地址与解析的地址匹配无需更新！"
}

################# IPv6比较和更新域名记录 #################
:if ($switchv6 = "true") do={
    :if ($resolvedIP6 != $WANip6) do={
        #双栈
        :log info ("CF: 正在更新 IPv6 解析地址 $CFdomain = $WANip6")
        /tool fetch http-method=put mode=https url="$CFurl46" http-header-field="X-Auth-Email:$CFemail,X-Auth-Key:$CFtkn,content-type:application/json" as-value output=user http-data="{\\"type\\":\\"AAAA\\",\\"name\\":\\"$CFdomain\\",\\"content\\":\\"$WANip6\\",\\"ttl\\":$CFttl,\\"proxied\\":false}"
        #单IPv6域名
        :log info ("CF: 正在更新 IPv6 解析地址 $CFdomain6 = $WANip6")
        /tool fetch http-method=put mode=https url="$CFurl6" http-header-field="X-Auth-Email:$CFemail,X-Auth-Key:$CFtkn,content-type:application/json" as-value output=user http-data="{\\"type\\":\\"AAAA\\",\\"name\\":\\"$CFdomain6\\",\\"content\\":\\"$WANip6\\",\\"ttl\\":$CFttl,\\"proxied\\":false}"
        #/ip dns cache flush
    } else={
        :log info "CF: IPv6公网地址与解析的地址匹配无需更新！"
    }
}

```
