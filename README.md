# OpenWrt マルチセッション (ニチバンベンチ)対策 ホットプラグ適用方式

This README is edited in Japanese.

### ニチバン対策（ホットプラグ適用方式）

<details><summary><b>ニチバンベンチについて</b></summary>

**ニチバンベンチとは**
> [ニチバン株式会社の公式サイト](https://www.nichiban.co.jp/)は、大量の小さなファイルを同時セッションで配信する構造のため、OpenWrtのMAP-E環境でNATポート枯渇を引き起こしやすい
> このサイトを10窓程度でリロードすることでポート枯渇問題を簡易的に再現できることから、「ニチバンベンチ」と呼ばれるようになった

**ニチバン（SNATポート枯渇）対策とは**
> OpenWrtのmap.shが複数セッション確立時にSNATテーブルの競合によってポート枯渇エラー（SNAT failed）を引き起こす問題に対処することをいう

※ニチバン株式会社及びそのサービスに問題があるわけではなく、OpenWrtのMAP-E環境側の制約による現象です

---

</details>

<details><summary><b>map.shについて</b></summary>

> mapパッケージに付属してる
> MAP-E方式をサポートするネットワーク設定スクリプト
> 19.07まではFW3対応、21.02以降FW4対応となった

**sha1sum map-19.07.sh**
- 431ad78fc976b70c53cdc5adc4e09b3eb91fd97f  map-19.07.sh

**sha1sum map-21.02.sh ～ map-25.12.sh**
- 7f0682eeaf2dd7e048ff1ad1dbcc5b913ceb8de4  map-21.02.sh ～ map-25.12.sh

```sh
wget -qO- https://raw.githubusercontent.com/openwrt/openwrt/openwrt-**.**/package/network/ipv6/map/files/map.sh | sha1sum
```

**問題点**

[MAP-Eはfw4/nftablesと互換性がありません #11972](https://github.com/openwrt/openwrt/issues/11972)

[MAP-T が割り当てられたポート範囲全体を活用しない #14449](https://github.com/openwrt/openwrt/issues/14449)

---

</details>

### ホットプラグ適用方式

> `attendedsysupgrade` (`owut`)でmap.shが上書きされる為、ホットプラグ適用方式に変更

| ファイル | 役割 | タイミング |
|---|---|---|
| `/etc/init.d/mape-patch` | map.sh portsetループ無効化 | START=19（netifd起動前） |
| `/etc/hotplug.d/iface/99-mape-snat` | numgen SNAT + DSCPリセット + conntrack | mape ifup後 |

<details><summary><b>対策内容</b></summary>

**numgen SNAT（ポート均等分散）**
> `numgen inc mod`で全割り当てポートを均等ラウンドロビン
> デフォルトmap.shは先頭16ポートに偏るが、本対策で240または1008ポート全てを均等に使用

**DSCPリセット（NTT QoS帯域制限回避）**
> IPv4/IPv6両方のDSCPをCS0にリセット

**conntrackタイムアウトチューニング**
> MAP-Eのポート数制約に合わせてconntrackの保持時間を短縮（ポート枯渇対策）

**Block-QUIC-MAP-E**
> MAP-E環境ではQUIC(UDP/443)が大量のポートを消費するため、
> QUICをブロックしてTCP/443にフォールバックさせることでポート枯渇を抑制

---

</details>

<details><summary><b>設定</b></summary>

```sh
#!/bin/sh
(
PKGS="map"
command -v opkg && for pkg in $PKGS; do opkg list-installed | grep -qw "$pkg" || opkg install "$pkg" 2>/dev/null; done
command -v apk && for pkg in $PKGS; do apk info -e "$pkg" 2>/dev/null || apk add "$pkg" 2>/dev/null; done

uci set network.mape.legacymap='1'
uci commit network

uci batch <<EOF
set firewall.block_quic_mape=rule
set firewall.block_quic_mape.name='Block-QUIC-MAP-E'
set firewall.block_quic_mape.proto='udp'
set firewall.block_quic_mape.dest_port='443'
set firewall.block_quic_mape.src='lan'
set firewall.block_quic_mape.dest='wan'
set firewall.block_quic_mape.target='DROP'
set firewall.block_quic_mape.family='ipv4'
set firewall.block_quic_mape.enabled='1'
commit firewall
EOF

mkdir -p /etc/hotplug.d/iface
cat > /etc/hotplug.d/iface/99-mape-snat << 'HOTPLUG'
#!/bin/sh
[ "$ACTION" = "ifup" ] && [ "$INTERFACE" = "mape" ] && {
    R="/tmp/map-mape.rules"; [ -f "$R" ] || exit 0
    eval $(cat "$R"); k=$RULE_BMR; [ -z "$k" ] && exit 0
    n=0; P=""
    for p in $(eval "echo \$RULE_${k}_PORTSETS"); do
        s=${p%%-*}; e=${p##*-}; x=$s
        while [ $x -le $e ]; do P="$P $n : $x ,"; n=$((n+1)); x=$((x+1)); done
    done
    [ "$n" -eq 0 ] && exit 0; P=${P%?}
    nft delete table inet mape 2>/dev/null
    nft add table inet mape
    nft add chain inet mape srcnat { type nat hook postrouting priority 0\; policy accept\; }
    V=$(eval "echo \$RULE_${k}_IPV4ADDR")
    for t in icmp tcp udp; do
        nft add rule inet mape srcnat ip protocol $t oifname "map-mape" counter snat ip to $V : numgen inc mod $n map {$P }
    done
    nft delete table inet mape_dscp 2>/dev/null
    nft add table inet mape_dscp
    nft add chain inet mape_dscp postrouting { type filter hook postrouting priority mangle\; policy accept\; }
    nft add rule inet mape_dscp postrouting ip dscp set cs0 comment \"mape-dscp-reset-v4\"
    nft add rule inet mape_dscp postrouting ip6 dscp set cs0 comment \"mape-dscp-reset-v6\"
    sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=3600 \
        net.netfilter.nf_conntrack_tcp_timeout_time_wait=120 \
        net.netfilter.nf_conntrack_udp_timeout=120 \
        net.netfilter.nf_conntrack_udp_timeout_stream=120 \
        net.netfilter.nf_conntrack_icmp_timeout=60 \
        net.netfilter.nf_conntrack_generic_timeout=60 >/dev/null 2>&1
}
[ "$ACTION" = "ifdown" ] && [ "$INTERFACE" = "mape" ] && {
    nft delete table inet mape 2>/dev/null
    nft delete table inet mape_dscp 2>/dev/null
}
HOTPLUG
chmod +x /etc/hotplug.d/iface/99-mape-snat

cat > /etc/init.d/mape-patch << 'INITD'
#!/bin/sh /etc/rc.common
START=19

start() {
    sed -i '/for portset in.*PORTSETS/c\    for portset in _; do continue' /lib/netifd/proto/map.sh 2>/dev/null
}
INITD
chmod +x /etc/init.d/mape-patch
/etc/init.d/mape-patch enable
/etc/init.d/mape-patch start

echo "設定が完了しました"
echo "Enterキーを押すとサービスを再起動します"
read dummy </dev/tty
for s in network firewall dnsmasq odhcpd ttyd; do /etc/init.d/$s restart 2>/dev/null; done
echo "完了しました"
)
```

---

</details>

<details><summary><b>削除</b></summary>

```sh
#!/bin/sh
(
# hotplug削除
rm -f /etc/hotplug.d/iface/99-mape-snat

# init.d削除
/etc/init.d/mape-patch disable 2>/dev/null
rm -f /etc/init.d/mape-patch

# map.sh復元（romfsから）
cp /rom/lib/netifd/proto/map.sh /lib/netifd/proto/map.sh 2>/dev/null

# nftテーブル削除
nft delete table inet mape 2>/dev/null
nft delete table inet mape_dscp 2>/dev/null

# legacymap削除
uci -q delete network.mape.legacymap
uci commit network

# Block-QUIC削除
uci -q delete firewall.block_quic_mape
uci commit firewall

echo "削除完了"
echo "Enterキーを押すとサービスを再起動します"
read dummy </dev/tty
for s in network firewall dnsmasq odhcpd ttyd; do /etc/init.d/$s restart 2>/dev/null; done
echo "完了しました"
)
```

---

</details>

<details><summary><b>正常性確認</b></summary>

```sh
(
OK=0; NG=0
ok() { echo "正常: $1"; OK=$((OK+1)); }
ng() { echo "異常: $1"; NG=$((NG+1)); }

grep -q 'for portset in _; do continue' /lib/netifd/proto/map.sh 2>/dev/null && ok "map.sh パッチ適用済み" || ng "map.sh 未パッチ"
[ -x /etc/init.d/mape-patch ] && ok "init.d/mape-patch" || ng "init.d/mape-patch 未配置"
[ -x /etc/hotplug.d/iface/99-mape-snat ] && ok "hotplug/99-mape-snat" || ng "hotplug/99-mape-snat 未配置"

NUMGEN=$(nft list table inet mape 2>/dev/null | grep -c numgen)
[ "$NUMGEN" -ge 3 ] && ok "numgen SNAT ${NUMGEN}本" || ng "numgen SNAT ${NUMGEN}本 (期待: 3)"
nft list table inet mape_dscp >/dev/null 2>&1 && ok "mape_dscp" || ng "mape_dscp 未検出"

for p in \
    "net.netfilter.nf_conntrack_tcp_timeout_established 3600" \
    "net.netfilter.nf_conntrack_udp_timeout 120" \
    "net.netfilter.nf_conntrack_udp_timeout_stream 120"; do
    k=${p% *}; v=${p##* }; val=$(sysctl -n $k 2>/dev/null)
    [ "$val" = "$v" ] && ok "$k=$val" || ng "$k=$val (期待: $v)"
done

FW4=$(fw4 restart 2>&1 | grep -c "ignoring section")
[ "$FW4" -eq 0 ] && ok "fw4 クリーン" || ng "fw4 portsetルール ${FW4}件残存"

ping -c 2 -W 3 1.1.1.1 >/dev/null 2>&1 && ok "IPv4疎通" || ng "IPv4疎通失敗"

echo "========================"
echo "正常: $OK / 異常: $NG"
)
```

---

</details>

<details><summary><b>ブラウザ動作確認</b></summary>

ベンチマーク該当サイトの構造
> 偶然にも同じホスティングサービス：KDDI Web Communications Inc.（KWC-NET: 150.60.0.0/16）
> CDNなし：オリジンサーバー（Cloudflare等を挟まない）から直接配信
> HTTP/1.1スタイル：多数の並列TCPコネクションを開く

- [nichiban.co.jp](https://www.nichiban.co.jp/)
- [aichi-mof.com](https://aichi-mof.com/)
- [soyabus.co.jp](http://www.soyabus.co.jp/)

---

</details>

<details><summary><b>過去版（map.sh差し替え方式）</b></summary>

- [map.sh.new](https://github.com/fakemanhk/openwrt-jp-ipoe/blob/main/map.sh.new)（fakemanhk氏）
- [map.sh.all](https://github.com/site-u2023/map-e/blob/main/map.sh.all)（map.sh.newベース、機能追加版）

---

</details>

### Qiita

- [OpenWrt IPoE設定 MAP-e(OCNバーチャルコネクト・V6プラス・NURO光 ニチバン対策) DS-LITE(Transix・Xpass・v6コネクト)](https://qiita.com/site_u/items/ec3099bd0d675d47fa98)

---

First draft: 5 Aug 2023

Update: 5 March 2026
