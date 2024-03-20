# OpenWrt 22.03 23.05 FW4 ニチバン対策 (のみ) 全自動構成 MAP書換版
This README is edited in Japanese.

ニチバン対策に map.sh を書き換えます。

元情報：
https://github.com/fakemanhk/openwrt-jp-ipoe#advanced-custom-configuration

ADVANCED CUSTOM CONFIGURATIONを元に以下を変更

```diff
- [ -z "$ip4prefixlen" ] && ip4prefixlen=32

- json_add_int mtu "${mtu:-1280}"
+ json_add_int mtu "${mtu:-1460}"

- nft add rule inet mape srcnat ip protocol $proto oifname "map-$cfg" snat ip to $(eval "echo \$RULE_${k}_IPV4ADDR") : numgen inc mod $portcount map { $allports }
+ nft add rule inet mape srcnat ip protocol $proto oifname "map-$cfg" counter packets 0 bytes 0 snat ip to $(eval "echo \$RULE_${k}_IPV4ADDR") : numgen inc mod $portcount map { $allports }
```

元情報：
https://ficusonline.com/ja/Blog/OpenWRT--V6purasuMAP-E~b1236

テーブルを削除する条件文スクリプト if ~ nft delete table inet mape ~fi 追加

```diff
+ if nft list tables | grep -q "table inet mape"; then
+ nft delete table inet mape
+ fi
```

First draft: 5 Aug 2023

Update: 20 March 2024
