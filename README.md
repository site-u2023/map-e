# OpenWrt マルチセッション (ニチバンベンチ)対策 MAP書換版
This README is edited in Japanese.

マルチセッション対策に map.sh を書き換えます

元情報：
https://github.com/fakemanhk/openwrt-jp-ipoe#advanced-custom-configuration

ADVANCED CUSTOM CONFIGURATIONを元に以下を変更

```diff
- [ -z "$ip4prefixlen" ] && ip4prefixlen=32

- json_add_int mtu "${mtu:-1280}"
+ json_add_int mtu "${mtu:-1460}"

- nft add rule inet mape srcnat ip protocol $proto oifname "map-$cfg" snat ip to $(eval "echo \$RULE_${k}_IPV4ADDR") : numgen inc mod $portcount map { $allports }
+ nft add rule inet mape srcnat ip protocol $proto oifname "map-$cfg" counter snat ip to $(eval "echo \$RULE_${k}_IPV4ADDR") : numgen inc mod $portcount map { $allports } comment "mape-snat-${proto}"
```

First draft: 5 Aug 2023

Update: 15 January 2026
