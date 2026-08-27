---
title: "Proxmox VMのフリーズをi6300esbウォッチドッグで自動復旧する ― ブラックリストと wd_keepalive の罠"
emoji: "🐕"
type: "tech"
topics: ["proxmox", "linux", "qemu", "systemd", "watchdog"]
published: true
---

## TL;DR

- 「VMは起動しているのにゲストOS内がハングしている」状態は、Proxmoxの外からは正常に見えるので自動復旧されない。
- QEMUの仮想ハードウェアウォッチドッグ `i6300esb` を使うと、ゲストが `/dev/watchdog` をペットしなくなった時点でQEMUがVMをハードリセットしてくれる。外部監視サーバは不要。
- ただしDebian/Ubuntuで素直に動かない罠が2つ：
  1. **`i6300esb` カーネルモジュールがディストリのブラックリストに載っている** → `systemd-modules-load` 経由の自動ロードが拒否される。
  2. **`watchdog` パッケージ同梱の `wd_keepalive` が先に `/dev/watchdog` を掴む** → 本体の `watchdog.service` が `Device or resource busy` で起動失敗する。
- 両方の回避方法と、実際にカーネルパニックを誘発した復旧テストの結果を書く。

## 背景

自宅のProxmoxクラスタで、たまにVMのゲストOSがカーネルハングして応答しなくなることがあった。`qm status` は `running` のまま、Proxmoxのグラフ上もCPU/メモリは動いているように見える。結局こちらが気づいて手動で `qm reset` するまで落ちたまま。

`kernel.panic=10` を入れておけばパニック時は再起動するが、それはあくまで「カーネルがパニックを認識できた場合」。ソフトロックアップやI/Oデッドロックで完全に固まると効かない。

そこで**ゲストの生存を外側（QEMU）から監視する**ハードウェアウォッチドッグを入れることにした。

## 仕組み

- **Proxmox側**：VMに仮想ウォッチドッグデバイスを追加する。
  ```bash
  qm set <vmid> --watchdog model=i6300esb,action=reset
  ```
  反映にはVMの停止→起動が必要（ホットプラグ不可）。`action=reset` のほか `poweroff` / `pause` も選べる。
- **ゲスト側（Linux）**：`watchdog` デーモンが `/dev/watchdog` を定期的に書き込んで「生きてるよ」と伝える。この書き込みが途絶えると、i6300esbのタイマーが切れてQEMUがVMをリセットする。
- **ゲスト側（OpenWrt）**：`procd` が `/dev/watchdog` の存在を自動検知してペットする。**追加設定は不要**。`ubus call system watchdog` で状態を確認できる。

## 罠1：i6300esb がブラックリストされている

Debian/Ubuntuでは `i6300esb` が最初からブラックリスト登録されている。

```console
$ grep -r i6300esb /lib/modprobe.d/ /etc/modprobe.d/
/lib/modprobe.d/blacklist_linux_6.1.0-xx.conf:blacklist i6300esb
```

やっかいなのは、**手動 `modprobe i6300esb` は成功する**が、`/etc/modules-load.d/*.conf` に書いて `systemd-modules-load.service` にロードさせようとすると拒否される点。

```console
$ journalctl -u systemd-modules-load
... systemd-modules-load[123]: Module 'i6300esb' is deny-listed (by kmod)
```

`modprobe` はブラックリストを無視するオプション経由で呼ばれるが、`systemd-modules-load` はブラックリストを尊重する。両者で挙動が違う。

### 回避：専用のoneshotサービスで明示的に modprobe する

```ini
# /etc/systemd/system/i6300esb-load.service
[Unit]
Description=Load i6300esb watchdog module
DefaultDependencies=no
Before=watchdog.service

[Service]
Type=oneshot
ExecStart=/sbin/modprobe i6300esb
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

`modules-load.d` を経由せず直接 `modprobe` を叩くので、ブラックリストに引っかからない。

`watchdog.service` がこれより後に起動するよう、drop-inで順序を保証する。

```ini
# /etc/systemd/system/watchdog.service.d/override.conf
[Unit]
Requires=i6300esb-load.service
After=i6300esb-load.service
```

```bash
systemctl daemon-reload
systemctl enable --now i6300esb-load.service
```

## 罠2：wd_keepalive が /dev/watchdog を横取りする

Debianの `watchdog` パッケージには `wd_keepalive` という補助デーモンが同梱されている。これは「`watchdog.service` を再起動・停止している最中もウォッチドッグを生かし続ける」ためのもので、デフォルト（`run_wd_keepalive=1`）で有効。

ところがこいつが先に `/dev/watchdog` を開いてしまうと、本体の `watchdog.service` がこうなる：

```console
$ systemctl status watchdog
... watchdog[456]: cannot open /dev/watchdog (errno = 16 = 'Device or resource busy')
```

`/dev/watchdog` は基本的に1プロセスしか開けない。

### 回避

```bash
# /etc/default/watchdog
run_wd_keepalive=0

systemctl disable --now wd_keepalive
```

自動復旧が目的なら本体の `watchdog.service` に一本化してよい。

## 設定：/etc/watchdog.conf

デフォルトでは監視デバイスの行がコメントアウトされているので有効化する。

```ini
# /etc/watchdog.conf
watchdog-device = /dev/watchdog
watchdog-timeout = 15
interval        = 1
```

`watchdog-timeout` はi6300esbのハードウェアタイマー値、`interval` はペット間隔。

```bash
systemctl enable --now watchdog
```

## 復旧テスト：本当にカーネルパニックから戻るか

設定しただけで安心せず、実際に固めて確認した。

```bash
# 自然復帰しないようにしてから
sysctl -w kernel.panic=0
sysctl -w kernel.sysrq=1

# カーネルクラッシュを誘発
echo c > /proc/sysrq-trigger
```

結果：**約90秒後にVMが自動的にリセットされ、正常に起動、SSHも復旧**した。`kernel.panic=0` にしてあるので、戻ってきたのはウォッチドッグの仕事で間違いない。

## ハマりどころまとめ

| 症状 | 原因 | 対処 |
|---|---|---|
| `systemd-modules-load` が `deny-listed` を吐く | ディストリのブラックリスト | 専用oneshotサービスで直接 `modprobe` |
| `watchdog.service` が `errno=16 busy` | `wd_keepalive` が先に掴んでいる | `run_wd_keepalive=0` + `disable wd_keepalive` |
| リセットが速すぎ/遅すぎ | `watchdog-timeout` / `interval` | 用途に合わせて調整（自宅なら15〜30秒でも十分） |

## 補足：LXCコンテナは対象外

`i6300esb` はQEMUの仮想デバイスなので、ホストとカーネルを共有するLXCコンテナには使えない。コンテナのハング対策が要るなら、プロセス/ヘルスチェックベースの別方式（外形監視 + `pct reboot` など）を用意する必要がある。
