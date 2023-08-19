# NICHIBAN countermeasures
This README is edited in Japanese.

ニチバン対策に map.sh を書き換えます。

元ファイル：
https://github.com/fakemanhk/openwrt-jp-ipoe#advanced-custom-configuration

ADVANCED CUSTOM CONFIGURATIONを元に以下を変更

```diff
- 		json_add_int mtu "${mtu:-1280}"
+ 		json_add_int mtu "${mtu:-1460}"

-	[ -z "$ip4prefixlen" ] && ip4prefixlen=32

- nft add rule inet mape srcnat ip protocol $proto oifname "map-$cfg" snat ip to $(eval "echo \$RULE_${k}_IPV4ADDR") : numgen inc mod $portcount map { $allports }
+ nft add rule inet mape srcnat ip protocol $proto oifname "map-$cfg" counter snat ip to $(eval "echo \$RULE_${k}_IPV4ADDR") : numgen inc mod $portcount map { $allports }
```
