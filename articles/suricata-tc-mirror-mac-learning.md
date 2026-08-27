---
title: "tcミラーでSuricataにパケットが流れない ― LinuxブリッジのMAC学習という落とし穴"
emoji: "🪞"
type: "tech"
topics: ["suricata", "proxmox", "linux", "network", "ids"]
published: false
---

## TL;DR

- Proxmoxホスト上で `tc` の `mirred` を使い、VMのtapインターフェースを丸ごとミラーしてSuricata（IDS）に流す構成を組んだ。
- ミラー先ではARP・mDNS・STPなどの**ブロードキャスト/マルチキャストだけ**が見え、肝心の**ユニキャストTCPが一切届かない**。
- 原因はSuricataでもtcでもなく、**ミラー用Linuxブリッジの標準MACアドレス学習**。全方向のトラフィックを1本のvethに集約した結果、ブリッジが「送信元も宛先も同じポートの先にいる」と学習し、ユニキャストを全部捨てていた。
- 対策は該当ブリッジの**全ポートで `learning off`**（＋誤学習エントリのフラッシュ）。ミラー専用ブリッジにL2ループの心配はないので、学習を切ってflood任せにするのが正しい。

## 背景：なぜスイッチSPANをやめてtcミラーにしたか

最初はL3スイッチのミラーポート（SPAN）でルーターの上流ポートをミラーしていた。しかし実ユニキャストTCPがほとんど捕捉できず、ブロードキャスト/ARP/mDNSばかり。使っていたスイッチはハードウェア高速転送パスがミラーフックをバイパスする仕様で、これはチップの制約だと判断した。

そこで「ルーターVMが乗っているハイパーバイザ側でミラーすれば、物理スイッチの都合は関係ない」と考え、Proxmoxホスト上のLinux `tc` に切り替えた。

## 構成

ルーターVM（OpenWrt）には4本のtapがぶら下がっている（WAN / LAN / transit / DMZ）。この4本すべてのingress/egressを、`tc` の `matchall` + `mirred egress mirror` で1本のvethにコピーする。

```bash
# ミラー受け渡し用の veth ペア
ip link add veth-mir0 type veth peer name veth-mir1
ip link set veth-mir0 up
ip link set veth-mir1 up

# ミラー専用ブリッジに片側を参加させる（IDS用VMのNICも同じブリッジのメンバー）
ip link set veth-mir0 master mirror

# 各tapにclsact qdiscを付けて、ingress/egress両方をveth-mir1へミラー
for IF in tap100i0 tap100i1 tap100i2 tap100i3; do
  tc qdisc add dev "$IF" clsact
  tc filter add dev "$IF" ingress matchall action mirred egress mirror dev veth-mir1
  tc filter add dev "$IF" egress  matchall action mirred egress mirror dev veth-mir1
done
```

IDS用VMから見ると、`mirror` ブリッジ経由でミラーされたフレームが1本のNIC（`ens19` 等）に入ってくる、という想定。

## 症状

`tc -s filter show` を見るとミラーフィルタは大量のバイト（数十〜数百MB）を確実に転送している。ところがIDS用VM内で `tcpdump -i ens19` すると：

```
ARP, Request who-has 192.0.2.1 tell 192.0.2.23
IP 192.0.2.23.5353 > 224.0.0.251.5353: 0 PTR (QM)? _googlecast._tcp.local.
STP 802.1d, Config, Flags [none], bridge-id ...
```

……ブロードキャストとマルチキャストしか出てこない。TLSハンドシェイクもHTTPも、普通のTCP会話が一切見えない。Suricataの `eve.json` もflow/tlsイベントをほぼ吐かない。

## 切り分け

tcが仕事をしているのは `tc -s` のカウンタで確認済み。ではどこで消えているのか。

**ブリッジの入口と出口でバイトカウンタを同期測定する**のが決定打だった。

```bash
# 10秒間の増加量を測る
read_rx() { cat /sys/class/net/$1/statistics/rx_bytes; }
read_tx() { cat /sys/class/net/$1/statistics/tx_bytes; }

a=$(read_rx veth-mir0); b=$(read_tx tap109i1)   # tap109i1 = IDS用VMのNIC
sleep 10
c=$(read_rx veth-mir0); d=$(read_tx tap109i1)
echo "veth-mir0 RX +$(( (c-a)/1024 )) KiB"
echo "IDS-VM   TX +$(( (d-b)/1024 )) KiB"
```

結果は極端だった：

```
veth-mir0 RX +48213 KiB
IDS-VM   TX +312 KiB
```

ブリッジには48MB入ってきているのに、IDS用VMには0.3MBしか出ていっていない。差の大半＝ユニキャストが、ブリッジの中で消えている。

## 原因：ブリッジのMAC学習とスプリットホライズン

Linuxブリッジは普通のスイッチと同じように**受信フレームの送信元MACを「どのポートから来たか」で学習**し、以後そのMAC宛のフレームは学習済みポートにだけ送る。そして**フレームを受けたポートと同じポートには送り返さない**（split horizon）。

今回の構成では、4本のtapの**ingressもegressも全部** `veth-mir1` に流し込んでいる。つまりミラーブリッジには：

- LAN端末 → 外部サーバ の通信（送信元 = 端末MAC）
- 外部サーバ → LAN端末 の通信（送信元 = ルーターMAC、宛先 = 端末MAC）

**どちらも `veth-mir0` という同じポートから**入ってくる。結果、ブリッジは「端末MACもルーターMACも外部サーバMACも、全部 `veth-mir0` の先にいる」と学習する。

すると、IDS用VMのポート（`tap109i1`）に転送すべきユニキャストフレームは、「宛先MACは `veth-mir0` にいる」＝「入ってきたポートと同じ」と判断され、split horizonで**黙って捨てられる**。ブロードキャスト/マルチキャストだけは常に全ポートにフラッドされるので、それだけがtcpdumpに見えていた。

さらに私の環境では、旧スイッチミラーの物理ポートがまだブリッジに残っており、そいつもLAN全端末のMACを学習していた。これも「宛先MACはあっちのポート」となってIDS用VMに届かない要因を増やしていた。

```
       ┌─────────────── mirror bridge ───────────────┐
tap ×4 │                                             │
──────▶│ veth-mir0 ──┐                               │
       │             ├─ FDB: 端末MAC → veth-mir0     │
       │             │      ルーターMAC → veth-mir0   │  ← 全部同じポート学習
       │             │      外部MAC → veth-mir0       │
       │             ▼                                │
       │  「宛先は veth-mir0 の先」＝入口と同じ → drop  │
       │                                             │
       │  tap109i1 (IDS-VM) ── ここにユニキャストが来ない │
       └─────────────────────────────────────────────┘
```

## 対策

ミラー専用ブリッジは**L2転送の実体を持たない**（フレームを"配る"だけで、ここを経由して端末同士が通信するわけではない）。だから学習は不要どころか有害。**全ポートで学習を切り、未知ユニキャストのfloodに任せる**のが正しい設計。

```bash
bridge link set dev veth-mir0 learning off
bridge link set dev tap109i1 learning off
bridge link set dev ens4f1   learning off   # 旧スイッチミラーの物理ポートが残っていれば

# 既存の誤学習エントリを消す（kernelによっては fdb flush が無いので個別 del）
bridge fdb show br mirror | awk '$1 ~ /:/ {print $1, $3}' | while read mac dev; do
  bridge fdb del "$mac" dev "$dev" 2>/dev/null
done
```

この変更は**可逆で、本番トラフィックには一切影響しない**（ミラーブリッジしか触らない）。

### 結果

```
veth-mir0 RX +47980 KiB
IDS-VM   TX +47120 KiB      # ほぼ一致（誤差2%程度）
```

IDS用VMのtcpdumpでTLSのSNIやJA3が見えるようになり、Suricataの `eve.json` も10秒で100行超のtls/quic/flowイベントを健全に吐くようになった。

## おまけ：VM再起動でtc設定が消える問題

`tc filter` はtapデバイスに紐づく揮発的な設定なので、**ルーターVMを再起動するとtapが作り直されてミラーが丸ごと消える**。しばらく「Suricataが何も見ていない」ことに気づかなかった。

Proxmoxの `hookscript` でVM起動の `post-start` フェーズにveth作成＋tcフィルタ追加＋`learning off` を冪等に流すようにして解決した。

```perl
# /var/lib/vz/snippets/ids-mirror-hook.pl（概略）
# phase == 'post-start' && vmid == '100' のとき、
# veth-mir ペアが無ければ作成 → mirror ブリッジに参加 →
# 4本の tap に tc qdisc/filter を追加（既存なら skip）→ learning off
```

`local` ストレージに `snippets` content type を足す必要がある：

```bash
pvesm set local --content images,vztmpl,backup,rootdir,iso,snippets
qm set 100 --hookscript local:snippets/ids-mirror-hook.pl
```

## まとめ

- ミラーが動いているのにIDSにユニキャストが来ないときは、**ミラーブリッジのFDBを疑う**。`bridge fdb show` で全MACが1ポートに寄っていたらこれ。
- ミラー専用ブリッジは**全ポート `learning off`** が定石。
- `tc` ミラーはVM再起動で消える。hookscriptで永続化する。
