# OpenWrt マルチセッション (ニチバンベンチ)対策 MAP書換版

This README is edited in Japanese.

## map.sh
> mapパッケージに付属してる
> MAP-E方式をサポートするネットワーク設定スクリプト
> 19.07まではFW3対応、21.02以降FW4対応となった

**sha1sum map-19.07.sh**
- 431ad78fc976b70c53cdc5adc4e09b3eb91fd97f  map-19.07.sh

**sha1sum map-21.02.sh map-22.03.sh map-23.05.sh map-24.10.sh**
- 7f0682eeaf2dd7e048ff1ad1dbcc5b913ceb8de4  map-21.02.sh
- 7f0682eeaf2dd7e048ff1ad1dbcc5b913ceb8de4  map-22.03.sh
- 7f0682eeaf2dd7e048ff1ad1dbcc5b913ceb8de4  map-23.05.sh
- 7f0682eeaf2dd7e048ff1ad1dbcc5b913ceb8de4  map-24.10.sh

```sh
wget -qO- https://raw.githubusercontent.com/openwrt/openwrt/openwrt-**.**/package/network/ipv6/map/files/map.sh | sha1sum
```

## ニチバンベンチ

**ニチバンベンチとは**
> [ニチバン株式会社の公式サイト](https://www.nichiban.co.jp/)は、大量の小さなファイルを同時セッションで配信する構造のため、OpenWrtのMAP-E環境でNATポート枯渇を引き起こしやすい
> このサイトを10窓程度でリロードすることでポート枯渇問題を簡易的に再現できることから、「ニチバンベンチ」と呼ばれるようになった

**ニチバン（SNATポート枯渇）対策とは**
> OpenWrtのmap.shが複数セッション確立時にSNATテーブルの競合によってポート枯渇エラー（SNAT failed）を引き起こす問題に対処することをいう

※ニチバン株式会社及びそのサービスに問題があるわけではなく、OpenWrtのMAP-E環境側の制約による現象です

## [map.sh.new](https://github.com/fakemanhk/openwrt-jp-ipoe/blob/main/map.sh.new)

[Configuring OpenWrt to work with Japan NTT IPv6 (MAP-E) service](https://github.com/fakemanhk/openwrt-jp-ipoe#advanced-custom-configuration)

### マルチセッション対策に`map.sh`を書き換え

**map.shの使用ポート数の違い：**
| | 使用ポート数 | 詰まりやすさ |
|---|---|---|
| デフォルト | 先頭16ポート | 多少詰まりにくい |
| map.sh.new | 全割り当てポート均等分散（240または1008ポート） | 実用上問題なし |

**TCP/UDP/ICMP分散**

> `numgen inc mod`（全割り当てポートを均等ラウンドロビン）

**ポート計算**

> nftablesで処理

**DONT_SNAT_TO**

> サーバー等で使用中のポートをSNAT対象から除外できる

```sh
#DONT_SNAT_TO="2938 7088 10233"
DONT_SNAT_TO="0"
```

## [map.sh.all](https://github.com/site-u2023/map-e/blob/main/map.sh.all)

### map.sh.newをベースに機能追加・修正

**MTU修正**
```diff
- json_add_int mtu "${mtu:-1280}"
+ json_add_int mtu "${mtu:-1460}"
```

**カウンター追加**
```diff
- nft add rule inet mape srcnat ip protocol $proto oifname "map-$cfg" snat ip to $(eval "echo \$RULE_${k}_IPV4ADDR") : numgen inc mod $portcount map { $allports }
+ nft add rule inet mape srcnat ip protocol $proto oifname "map-$cfg" counter snat ip to $(eval "echo \$RULE_${k}_IPV4ADDR") : numgen inc mod $portcount map { $allports } comment "mape-snat-${proto}"
```

**DSCPリセット（CS0）**
> ニチバン対策。IPv4/IPv6両方のDSCPをCS0にリセット
```sh
ip dscp set cs0
ip6 dscp set cs0
```
> `inet mape_dscp` テーブルとして動的に適用（teardown時に削除）

**TCPMSSクランプ（PMTU自動調整）**
> ipip6トンネルのMTU制限によるフラグメント発生を防止
> TCP接続確立時にMSSをPMTUに合わせて自動調整
```sh
oifname "map-$cfg" tcp flags syn / syn,rst tcp option maxseg size set rt mtu
```

**teardown時にクリーンアップ**
```sh
nft delete table inet mape
nft delete table inet mape_dscp
nft delete table inet mape_tcpmss
```

**conntrackタイムアウトチューニング**
> MAP-Eのポート数制約に合わせてconntrackの保持時間を短縮（ポート枯渇対策）
```sh
net.netfilter.nf_conntrack_tcp_timeout_established=3600
net.netfilter.nf_conntrack_tcp_timeout_time_wait=120
net.netfilter.nf_conntrack_udp_timeout=180
net.netfilter.nf_conntrack_udp_timeout_stream=180
net.netfilter.nf_conntrack_icmp_timeout=60
net.netfilter.nf_conntrack_generic_timeout=60
```

**FMR（Forwarding Mapping Rule: RFC 7597） UCI rule**
> `network.*.fmr_rule`からFMRルール読み込み

**Block-QUIC-MAP-E**
MAP-E環境ではQUIC(UDP/443)が大量のポートを消費するため、
QUICをブロックしてTCP/443にフォールバックさせることでポート枯渇を抑制
> map.sh自体には含まれない（firewallルールで設定）
```sh
# Block-QUIC-MAP-E設定
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
```

**設定**
> map.sh単独インストール可
```sh
#!/bin/sh
(
MAPSH="/lib/netifd/proto/map.sh"
HASH21="7f0682eeaf2dd7e048ff1ad1dbcc5b913ceb8de4"
PKGS="map coreutils-sha1sum"
command -v opkg && for pkg in $PKGS; do opkg list-installed | grep -qw "$pkg" || opkg install "$pkg" 2>/dev/null; done
command -v apk && for pkg in $PKGS; do apk info -e "$pkg" 2>/dev/null || apk add "$pkg" 2>/dev/null; done
HASH="$(sha1sum "$MAPSH" | awk '{print $1}')"
if [ "$HASH" != "$HASH21" ]; then
    echo "ハッシュ不一致: $HASH"
    echo -n "続行しますか? [y/N]: "
    read answer </dev/tty
    [ "$answer" = "y" ] || [ "$answer" = "Y" ] || exit 0
fi
wget -O "${MAPSH}.new" "https://raw.githubusercontent.com/site-u2023/map-e/refs/heads/main/map.sh.all" && {
    cp -f "$MAPSH" "${MAPSH}.bak"
    mv -f "${MAPSH}.new" "$MAPSH"
    chmod +x "$MAPSH"
    echo "更新成功"
} || { echo "更新失敗"; exit 1; }
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
echo "Enterキーを押すとサービスを再起動します"
read dummy </dev/tty
for s in network firewall dnsmasq odhcpd ttyd; do /etc/init.d/$s restart 2>/dev/null; done
echo "完了しました"
)
```

### 正常性確認
```sh
(
CHECK_OK=0
CHECK_NG=0

ok() { echo "正常: $1"; CHECK_OK=$((CHECK_OK+1)); }
ng() { echo "異常: $1"; CHECK_NG=$((CHECK_NG+1)); }

check_sysctl() {
    val=$(sysctl -n $1 2>/dev/null)
    [ "$val" = "$2" ] && ok "$1=$val" || ng "$1=$val (期待値: $2)"
}

nft list table inet mape_tcpmss >/dev/null 2>&1 && ok "mape_tcpmss" || ng "mape_tcpmss: テーブルなし"
nft list table inet mape_dscp >/dev/null 2>&1 && ok "mape_dscp" || ng "mape_dscp: テーブルなし"
nft list table inet mape >/dev/null 2>&1 && ok "mape SNAT" || ng "mape SNAT: テーブルなし"

check_sysctl net.netfilter.nf_conntrack_tcp_timeout_established 3600
check_sysctl net.netfilter.nf_conntrack_tcp_timeout_time_wait 120
check_sysctl net.netfilter.nf_conntrack_udp_timeout 180
check_sysctl net.netfilter.nf_conntrack_udp_timeout_stream 180
check_sysctl net.netfilter.nf_conntrack_icmp_timeout 60
check_sysctl net.netfilter.nf_conntrack_generic_timeout 60

echo "========================"
echo "正常: $CHECK_OK / 異常: $CHECK_NG"
)
```

### ブラウザ動作確認

- [nichiban.co.jp](https://www.nichiban.co.jp/)

- [soyabus.co.jp](http://www.soyabus.co.jp/)

### Qiita

- [OpenWrt IPoE設定 MAP-e(OCNバーチャルコネクト・V6プラス・NURO光 ニチバン対策) DS-LITE(Transix・Xpass・v6コネクト)](https://qiita.com/site_u/items/ec3099bd0d675d47fa98)

---

First draft: 5 Aug 2023

Update: 26 February 2026
