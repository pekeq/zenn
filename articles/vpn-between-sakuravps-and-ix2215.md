---
title: "さくらVPS上のUbuntu 24とNEC IX2215間でIPv4 over IPv6 VPNする"
emoji: "🥌"
type: "tech"
topics: ["network","vpn","ix2215","strongswan"]
published: true
---
# はじめに

本記事では、さくらVPSとフレッツ光 (IPv6 IPoE) 間のIPv6ネットワークを経由して、IPv4プライベートアドレスを持つ2つのネットワーク間を接続するIKEv2/IPsec VPNトンネルの構築例を紹介します。

# 前提条件

| |イニシエータ|レスポンダ|
|-|-|-|
|機種|Ubuntu 24.04 on さくらVPS|NEC UNIVERGE IX2215|
|バージョン|Ubuntu 24.04.3 LTS|IX2215 10.11.9|
|IPv6アドレス|2401:2500:102::52|2409:10:2900::1|
|IPv4 プライベートアドレス|192.168.18.10/24|192.168.19.1/24|

# Ubuntu 24 (さくらVPS上) の設定

## 前提条件

- さくらVPSでIPv6の設定がされていること
  - [IPv6有効化](https://manual.sakura.ad.jp/vps/network/index.html#ipv6) (manual.sakura.ad.jp)
- IPv4ルーティングが有効になっていること (`net.ipv4.ip_forward=1`)

## strongSwanインストール

```shell
sudo apt install charon-systemd strongswan-swanctl
```

## ループバックインターフェース上にサブネット (192.168.18.0/24) 作成

このサブネットがさくらVPS側のローカル側ネットワークとして扱われます。

```yaml:/etc/netplan/99-local.yaml
network:
  version: 2
  ethernets:
    lo:
      addresses:
        - 192.168.18.10/24
```

設定を適用します。

```shell
sudo netplan apply
```

## ファイアウォール設定

IX2215のIPv6アドレスからのIKEv2 (UDP 500) およびIPsec (ESP) 通信を許可します。

```shell
sudo ufw allow proto udp from 2409:10:2900::1 to any port 500
sudo ufw allow proto esp from 2409:10:2900::1 to any
```

## strongSwan設定

Pre-Shared Key (事前共有鍵) が含まれているので、ファイルのパーミッションを適切に設定してください。

```properties:/etc/swanctl/conf.d/home.conf
connections {
  home {
    local_addrs = 2401:2500:102::52
    remote_addrs = 2409:10:2900::1
    local {
      auth = psk
      id = 2401:2500:102::52
    }
    remote {
      auth = psk
      id = 2409:10:2900::1
    }
    children {
      v4-tunnel {
        remote_ts = 192.168.19.0/24
        local_ts = 192.168.18.0/24
        start_action = start
        updown = /usr/lib/ipsec/_updown iptables
        esp_proposals = aes256-sha512-modp3072
      }
    }
    version = 2
    mobike = no
    proposals = aes256-sha512-modp3072
  }
}

secrets {
  ike-home-psk {
    id = 2409:10:2900::1
    secret = secret1
  }
}
```

# IX2215の設定

```
! デバッグ用にIKEv2のログレベルを上げる
logging subsystem ike2 notice

! さくらVPS側のプライベートネットワーク (192.168.18.0/24) をVPNトンネルインタフェースに向ける
ip route 192.168.18.0/24 Tunnel10.0

! 対向ホストからのIKEv2, ESP通信を許可
ipv6 access-list permit-ipsec permit 50 src 2401:2500:102::52/128 dest 2409:10:2900::1/128
ipv6 access-list permit-ipsec permit udp src 2401:2500:102::52/128 sport eq 500 dest 2409:10:2900::1/128 dport eq 500

! PSKの値
ikev2 authentication psk id ipv6 2401:2500:102::52 key char secret1

! フレッツONUに接続しているI/F (抜粋)
interface GigaEthernet0.0
  no ip address
  ipv6 enable
  ipv6 traffic-class tos 0
  ipv6 tcp adjust-mss auto
  ipv6 filter permit-ipsec 2000 in
  no shutdown

! 宅内LANに接続しているI/F (抜粋)
interface GigaEthernet2.0
  ip address 192.168.19.1/24
  ipv6 enable
  ipv6 interface-identifier 00:00:00:00:00:00:00:01
  no shutdown

! IPsec VPN I/F
interface Tunnel10.0
  tunnel mode ipsec-ikev2
  ip unnumbered GigaEthernet2.0
  ip tcp adjust-mss auto
  ikev2 child-pfs 3072-bit
  ikev2 child-proposal enc aes-cbc-256
  ikev2 child-proposal integrity sha2-512
  ikev2 dpd interval 10
  ikev2 negotiation-direction responder
  ikev2 sa-proposal enc aes-cbc-256
  ikev2 sa-proposal integrity sha2-512
  ikev2 sa-proposal dh 3072-bit
  ikev2 source-address GigaEthernet2.0
  ikev2 peer 2401:2500:102::52 authentication psk
  no shutdown
```

# 接続確認

## Ubuntu 24側

```shell
sudo systemctl restart strongswan.service
sudo swanctl --list-conns
sudo swanctl --list-sas
```

## IX2215側

```
show ikev2 sa
show ikev2 child-sa
```


# 参考

- [Configuration Examples](https://wiki.strongswan.org/projects/strongswan/wiki/ConfigurationExamples) (wiki.strongswan.org)
- [UNIVERGE IXシリーズ コマンドリファレンスマニュアル](https://jpn.nec.com/univerge/ix/Manual/index.html?#crm) (jpn.nec.com)
- [UNIVERGE IXシリーズ 機能説明書](https://jpn.nec.com/univerge/ix/Manual/index.html?#fd) (jpn.nec.com)
- [UNIVERGE IXシリーズ 設定事例集](https://jpn.nec.com/univerge/ix/Manual/index.html?#ex) (jpn.nec.com)
- [フレッツ 光ネクスト向け IPv6 VPN接続 設定ガイド（IPv6 IPoE）](https://jpn.nec.com/univerge/ix/Support/ipv6/native/ipv6-vpn_d.html)  (jpn.nec.com)
