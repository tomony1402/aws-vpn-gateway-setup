# aws-vpn-gateway-setup (WireGuard)

オンプレミスの分散配信機とAWS VPC環境をWireGuard VPNで接続し、プライベートIPでの通信を可能にする基盤構築の記録。

## 1. 構成概要
- **オンプレミス側**: 分散配信機 (10.30.0.101)
- **AWS側**: VPNルーター (AlmaLinux 9 / 10.20.1.109)
- **目的**: 既存のメール配信機能をモデルとしたコンテナサービスへの移行を見据え、セキュアなハイブリッドネットワークを構築する。

## 2. ネットワーク設計
VPN専用の仮想セグメント（10.30.0.0/24）を介して、AWS VPC内のリソースへルーティングを行う。

- **ネットワーク構成**: 
    - **AWS VPC**: 10.20.0.0/16（VPNサーバー用）、10.10.0.0/16（配信機用）
    - **VPNトンネル**: 10.30.0.0/24（WireGuard仮想網）
- **VPN方式**: WireGuard (UDP 51820 / 公開鍵認証)
- **ルーティング戦略**: 
    - オンプレ側にて `10.10.0.0/16` の宛先を `wg0` インターフェースへ向けることで、VPC内への透過的なアクセスを実現。

## 3. 構築手順

### 3.0 WireGuardとiptablesインストール (AlmaLinux 9)
```bash
# WireGuardと管理ツールのインストール
sudo dnf install -y epel-release
sudo dnf install -y wireguard-tools iptables-services

cd /etc/wireguard/
umask 077
# 認証鍵の作成
wg genkey | tee privatekey | wg pubkey > publickey

# 鍵の内容を確認
cat privatekey
cat publickey
```

### 3.1 AWS：ネットワーク基盤の設定
1. **ソース/宛先チェックの無効化（重要）**:
    - EC2（VPNルーター）のアクション > ネットワーキング > [ソース/宛先チェックを変更] を「停止」に設定。
    - ※ 自身が宛先ではないパケットを転送するために必須。
2. **セキュリティグループの調整**:
    - **VPNルーター**: UDP 51820 をオンプレIPから許可。
    - **配信機**: ICMP および TCP 25 を VPNルーター(10.20.1.109)から許可。

### 3.2 AWS：VPNサーバーの設定 (AlmaLinux 9)
1. **IPフォワーディング有効化**:
    ```bash
    echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
    sysctl -p
    ```
2. **NAT（マスカレード）設定**:
    ```bash
    # 10.30帯からの通信が eth0(VPC側) へ出ていく時に送信元IPを書き換え
    dnf install iptables -y
    iptables -t nat -A POSTROUTING -s 10.30.0.0/24 -o eth0 -j MASQUERADE
    
    # 設定の保存と永続化
    dnf install iptables-services -y
    iptables-save > /etc/sysconfig/iptables
    systemctl enable --now iptables
    ```

### 3.3 オンプレ：分散配信機の設定
1. **WireGuard設定 (`/etc/wireguard/wg0.conf`)**:
    ```ini
    [Interface]
    PrivateKey = <オンプレ機の秘密鍵>
    Address = 10.30.0.101/24　　   #38を101としているので39を102としてください

    [Peer]
    PublicKey = eDjMW2D9r8GfAw9A9lGeAs1EL6uNDhZYdEquZWsXJnM=
    Endpoint = <AWSルーターのグローバルIP>:51820
    # 配信機（10.10帯）を含むように範囲を指定
    AllowedIPs = 10.30.0.0/24, 10.10.0.0/16
    PersistentKeepalive = 25
    ```
    
2. **接続開始**:
    ```bash
    wg-quick down wg0 (スナップショットからの場合)
    wg-quick up wg0
　　
  
### 3.4 AWS：VPNルーター側の設定
1. **WireGuard設定 (`/etc/wireguard/wg0.conf`)**:
   ```ini
   [Interface]
   PrivateKey = SOoC28kVj3d25dWOEzPAMkKx8UY0DiZc7+CqfbDazlE=
   # VPNネットワーク内でのこのルーターのIP
   Address = 10.30.0.1/24
   # 待ち受けポート（セキュリティグループでUDP 51820を許可）
   ListenPort = 51820
   MTU = 1420

   [Peer]
   PublicKey = <オンプレ側の公開鍵>
   AllowedIPs = 10.30.0.101/32

   # 2台目（追加）
   [Peer]
   PublicKey = <2台目の公開鍵>
   AllowedIPs = 10.30.0.102/32
   ```
   
   ### 設定の反映（再起動）
   ```bash
   wg syncconf wg0 <(wg-quick strip wg0)
   wg show
   
## 4. 疎通確認
-  systemctl enable wg-quick@wg0　自動起動設定
-  systemctl is-enabled wg-quick@wg0　で反映されたか確認
- **ICMP**: オンプレ機からVPNルーター側(10.30.0.1)にping
- **ICMP**: オンプレ機から配信機(10.10.49.210等)
- **SMTP**: オンプレ機 → 配信機(Port 25) 接続成功。
    - 配信機側の `maillog` にて `connect from ip-10-20-1-109.ap-northeast-1.compute.internal` の記録を確認。
- dnf update　最後にupdateで終了
     

