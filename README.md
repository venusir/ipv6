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

* [RouterOS CloudFlare DDNS配置](https://xiaomz.com/index.php/2023/03/26/routeros-cloudflare-ddns%E9%85%8D%E7%BD%AE%EF%BC%88%E6%88%90%E5%8A%9F%EF%BC%89/)

* 脚本

```
################# CloudFlare 变量 #################
# 是否开启debug调试模式
:local CFDebug "false"
# 是否开启CFcloud功能
:local CFcloud "false"

# 修改为有公网IP的接口名称
:global WANInterface "修改成你的外网接口"

# 修改为你要ddns的域名，若是二级域名，这里填写完整的二级域名
:local CFdomain "修改成你的域名，这里我用的二级域名"

# CloudFlare 全局密钥token或者有权限操作解析域名的token
:local CFtkn "这里不要用全局token,新建一个，新建方法见下文"


# 域名zoneId
:local CFzoneid "CloudFlare面板中可以找到"
# 要ddns的域名记录id
:local CFid "这个ID可以通过脚本获取，脚本见下文"

# 记录类型 一般无需修改
:local CFrecordType ""
:set CFrecordType "A"

# 记录ttl值，一般无需修改
:local CFrecordTTL ""
:set CFrecordTTL "120"

#########################################################################
########################  下面的内容请勿修改 ############################
#########################################################################

:log info "开始更新解析记录 $CFDomain ..."

################# 内部变量 variables #################
:local previousIP ""
:global WANip ""

################# 构建 CF API Url (v4) #################
:local CFurl "https://api.cloudflare.com/client/v4/zones/"
:set CFurl ($CFurl . "$CFzoneid/dns_records/$CFid");
 
################# 获取或设置以前的ip变量 #################
:if ($CFcloud = "true") do={
    :set WANip [/ip cloud get public-address]
};

:if ($CFcloud = "false") do={
    :local currentIP [/ip address get [/ip address find interface=$WANInterface ] address];
    :set WANip [:pick $currentIP 0 [:find $currentIP "/"]];
};

:if ([/file find name=ddns.tmp.txt] = "") do={
    :log error "没有找到记录前一个公网IP地址的文件, 自动创建..."
    :set previousIP $WANip;
    :execute script=":put $WANip" file="ddns.tmp";
    :log info ("CF: 开始更新解析记录, 设置 $CFDomain = $WANip")
    /tool fetch http-method=put mode=https output=none url="$CFurl" http-header-field="Authorization:Bearer $CFtkn,content-type:application/json" http-data="{\"type\":\"$CFrecordType\",\"name\":\"$CFdomain\",\"ttl\":$CFrecordTTL,\"content\":\"$WANip\"}"
    :error message="没有找到前一个公网IP地址的文件."
} else={
    :if ( [/file get [/file find name=ddns.tmp.txt] size] > 0 ) do={ 
    :global content [/file get [/file find name="ddns.tmp.txt"] contents] ;
    :global contentLen [ :len $content ] ;  
    :global lineEnd 0;
    :global line "";
    :global lastEnd 0;   
            :set lineEnd [:find $content "\n" $lastEnd ] ;
            :set line [:pick $content $lastEnd $lineEnd] ;
            :set lastEnd ( $lineEnd + 1 ) ;   
            :if ( [:pick $line 0 1] != "#" ) do={   
                #:local previousIP [:pick $line 0 $lineEnd ]
                :set previousIP [:pick $line 0 $lineEnd ];
                :set previousIP [:pick $previousIP 0 [:find $previousIP "\r"]];
            }
    }
}

######## 将调试信息写入日志 #################
:if ($CFDebug = "true") do={
 :log info ("CF: 域名 = $CFdomain")
 :log info ("CF: 前一个解析IP地址 = $previousIP")
 :log info ("CF: 当前IP地址 = $currentIP")
 :log info ("CF: 公网IP = $WANip")
 :log info ("CF: 请求CFurl = $CFurl&content=$WANip")
 :log info ("CF: 执行命令 = \"/tool fetch http-method=put mode=https url=\"$CFurl\" http-header-field="Authorization:Bearer $CFtkn,content-type:application/json" output=none http-data=\"{\"type\":\"$CFrecordType\",\"name\":\"$CFdomain\",\"ttl\":$CFrecordTTL,\"content\":\"$WANip\"}\"")
};
  
######## 比较并更新记录 #####
:if ($previousIP != $WANip) do={
 :log info ("CF: 开始更新解析记录, 设置 $CFDomain = $WANip")
 /tool fetch http-method=put mode=https url="$CFurl" http-header-field="Authorization:Bearer $CFtkn,content-type:application/json" output=none http-data="{\"type\":\"$CFrecordType\",\"name\":\"$CFdomain\",\"ttl\":$CFrecordTTL,\"content\":\"$WANip\"}"
 /ip dns cache flush
    :if ( [/file get [/file find name=ddns.tmp.txt] size] > 0 ) do={
        /file remove ddns.tmp.txt
        :execute script=":put $WANip" file="ddns.tmp"
    }
} else={
 :log info "CF: 未发生改变，无需更新!"
}
```

* 获取Cloudflare域名的dns id

```
curl -X GET "https://api.cloudflare.com/client/v4/zones/这里填入zoneID/dns_records" -H "X-Auth-Email: 这里填入CF的用户名" -H "X-Auth-Key: 这里填入CF的全局token" -H "Content-Type: application/json" | python -mjson.tool
```


