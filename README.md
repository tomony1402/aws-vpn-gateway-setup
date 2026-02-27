# aws-vpn-gateway-setup

オンプレミスの分散配信機とAWS環境をVPNで接続し、プライベートIPでの通信を可能にする基盤構築の記録。

## 1. 構成概要
- **オンプレ側**: 分散配信機
- **AWS側**: VPNサーバー (AlmaLinux 9)
- **目的**: 拠点間をプライベートIPで接続し、セキュアな配信環境を構築する。

## 2. ネットワーク設計
セグメントを分離し、ルーティングとDNSによってセキュアな通信を確保。

- **ネットワーク構成**: 
    - AWS VPC（VPNサーバー用セグメント）
    - オンプレミス/他環境（配信機用セグメント）
    ※ 両環境のVPC/ネットワークは分離して運用。
- **VPN接続方式**: StrongSwan (IPsec VPN)
- **名前解決**: 
    - オンプレミス側のDNSにて、配信機のプライベートIPを返すよう構成。
    - これにより、VPNトンネル経由でのセキュアなプライベート通信を実現。
 
## 3. 構築手順

### 3.1 AWS：ネットワーク基盤の準備
VPNサーバーが通信を仲介（ルーティング）できるように設定。

1. **VPCピアリングの設定**
    - VPNサーバー用VPCと配信機用VPCを接続。
    - 各VPCのルートテーブルに、相手側VPCおよびオンプレミス側ネットワーク（CIDR）へのルートを追加。
2. **EC2（AlmaLinux 9）の起動と詳細設定**
    - **ソース/宛先チェックの無効化（重要）**: 
        - [アクション] > [ネットワーキング] > [ソース/宛先チェックを変更] を選択。
        - 「停止」に設定。
        - ※ これを忘れると、自分宛て以外のパケットをEC2が破棄するため、VPN通信が成立しません。
3. **セキュリティグループの解放**
    - StrongSwanが使用するポート（UDP 500, 4500）をオンプレIPに対して許可。

### 3.2 AWS：VPNサーバー（StrongSwan）の設定
StrongSwan (swanctl) を使用して、オンプレミス環境とのIPsecトンネルを構築。

1. **設定ファイルの配置**
    - 以下のパスに設定ファイルを配置。
    - **パス**: `/etc/strongswan/swanctl/swanctl.conf`
    - **設定内容**: [swanctl.conf](./vpn-server/swanctl.conf)

2. **設定のポイント**
    - `local_ts = 10.10.0.0/16`: VPNサーバー側のVPCセグメントを指定。
    - `remote_ts = 2.56.0.0/24`: オンプレミス（配信機）側のセグメントを指定。
    - `remote_addrs = %any`: 接続元のオンプレIPを動的に受け入れ可能な設定。

3. **サービスの起動と設定反映**
    ```bash
    # swanctlの設定を読み込み
    swanctl --load-all
    
    # サービスの起動と有効化
    systemctl enable --now strongswan
    ```
    
#### 運用のスケーラビリティ
- **ネットワーク範囲による許可 (`remote_ts = 2.56.0.0/24`)**:
  個別のIPアドレスではなくサブネット単位で通信を許可しているため、オンプレミス側に新しい配信機を増設する際も、AWS側の設定変更やメンテナンス時間を設けることなく、即座に疎通を開始できます。

### 3.3 オンプレ：分散配信機（AlmaLinux 9）の設定
オンプレミス環境の分散をVPNクライアントとして設定し、AWS環境へ接続します。

1. **事前準備（スナップショットの取得）**
    - 作業ミスによるサービス停止を防ぐため、XCP-ng上で対象機（2.56.0.126）のスナップショットを取得します。

2. **ネットワーク設定の変更（IPアドレス固定）**
    - 配信機セグメント（2.56.0.0/24）の空きIPを割り当てます。
    - **設定ファイルパス**: `/etc/NetworkManager/system-connections/[接続名].nmconnection`
    - **設定例**:
      ```ini
      [ipv4]
      address1=2.56.0.126/24,2.56.0.1  # IPアドレス/サブネット, ゲートウェイ
      dns=8.8.8.8;                     # DNSサーバー
      method=manual
      ```
    - **反映手順**:
      ```bash
      # ファイル権限を適切に設定（NetworkManagerの仕様）
      chmod 600 /etc/NetworkManager/system-connections/enX0.nmconnection
      
      # 設定の再読み込みと反映
      nmcli connection reload
      nmcli connection up enX0
      
      # 反映確認
      ip addr show
      ```

3. **設定ファイルの配置**
    - AWS側のVPNサーバーへ接続するための設定を記述します。
    - **パス**: `/etc/strongswan/swanctl/swanctl.conf`
    - **設定内容**: [swanctl.conf](./onprem/etc/strongswan/swanctl/swanctl.conf)

4. **VPN接続の開始と確認**
    - 設定を読み込み、手動でトンネルを確立させます。
    ```bash
    # 設定の読み込み
    swanctl --load-all

    # トンネルの開始
    swanctl --initiate --child net
    ```

## 4. 疎通確認
- `ping` および `traceroute` を使用し、オンプレ配信機からAWS側のプライベートIP（10.10.x.xなど）へ到達できることを確認。
- オンプレ側DNSで名前解決を行い、意図したプライベートIPが返ることを確認。

## 4. 疎通確認
- `ping` および `traceroute` を使用し、オンプレ配信機からAWS側のプライベートIPへ到達できることを確認。
- オンプレ側DNSで名前解決を行い、意図したプライベートIPが返ることを確認。
