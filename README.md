# OpenWrt 22.03 23.05 FW4 ニチバン対策 (のみ) 全自動構成 MAP書換版
This README is edited in Japanese.

ニチバン対策に map.sh を書き換えます。

元ファイル：
https://github.com/fakemanhk/openwrt-jp-ipoe#advanced-custom-configuration

ADVANCED CUSTOM CONFIGURATIONを元に以下を変更

```diff
-	[ -z "$ip4prefixlen" ] && ip4prefixlen=32

- 		json_add_int mtu "${mtu:-1280}"
+ 		json_add_int mtu "${mtu:-1460}"

- nft add rule inet mape srcnat ip protocol $proto oifname "map-$cfg" snat ip to $(eval "echo \$RULE_${k}_IPV4ADDR") : numgen inc mod $portcount map { $allports }
+ nft add rule inet mape srcnat ip protocol $proto oifname "map-$cfg" counter packets 0 bytes 0 snat ip to $(eval "echo \$RULE_${k}_IPV4ADDR") : numgen inc mod $portcount map { $allports }
```

First draft: 20 Aug 2023

Update: 20 Aug 2023
