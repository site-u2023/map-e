# NICHIBAN countermeasures
This README is edited in Japanese.

ニチバン対策に map.sh を書き換えます。

https://github.com/fakemanhk/openwrt-jp-ipoe#advanced-custom-configuration

ADVANCED CUSTOM CONFIGURATIONを元に以下を変更

```diff
- 		json_add_int mtu "${mtu:-1280}"
+ 		json_add_int mtu "${mtu:-1460}"

-	[ -z "$ip4prefixlen" ] && ip4prefixlen=32
```
