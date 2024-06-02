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
curl -X GET "https://api.cloudflare.com/client/v4/zones/这里填入zoneID/dns_records" -H "X-Auth-Email: 这里填入CF的用户名" -H "X-Auth-Key: 这里填入CF的全局token" -H "Content-Type: application/json" | python -mjson.tool
```


