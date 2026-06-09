---
title: "macOSでUSBメモリが激遅だったので原因を計測したら、FAT32が犯人だった"
emoji: "💾"
type: "tech"
topics: ["macos", "usb", "exfat", "ストレージ"]
published: true
---

手元のUSBメモリ（Verbatimの安いやつ、15GB）への書き込みがやたら遅くて、ファイル一個コピーするのに何分も待たされる。USB 3.0対応のはずなのにこれは何かおかしい、と思って原因をちゃんと潰していったら、最終的に書き込みが **41倍** になった。やったことと、その過程で取った数字を残しておく。

![FAT32のときのベースライン計測。書き込み0.9MB/s、読み込み11MB/s](/images/usb-speed-baseline.png)

## まず疑うところを潰す

「遅い」と感じたとき、最初に切り分けたいのは *USBの接続そのものが遅いのか、中身が遅いのか* の2択だ。`ioreg` で接続速度を見る。

```bash
ioreg -p IOUSB -l -w 0 | grep -E '"USB Product Name"|"Device Speed"'
```

`Device Speed = 3` と出た。これは SuperSpeed、つまり USB 3.0（5Gbps）でちゃんとリンクできているという意味（2だと USB 2.0）。ケーブルもポートも差し方も悪くない。となると遅いのは中身、ドライブ側の問題だ。

実測してみる。`dd` でべたっと書いて読む。

```bash
dd if=/dev/zero of=/Volumes/USB/test bs=1m count=200   # 書き込み
dd if=/Volumes/USB/test of=/dev/null bs=1m             # 読み込み
```

結果がこれ。

| | 速度 |
|---|---|
| 書き込み | **0.9 MB/s** |
| 読み込み | 11 MB/s |

書き込み0.9MB/s。USB 3.0でリンクしてるのに、これはフロッピーかという速度だ。読み込みの11MB/sもUSB3.0としては全然出ていない。リンクは速いのに中身が遅い、という最初の見立て通りだった。

## フォーマットを見たら FAT32 だった

ドライブの構成を見る。

```bash
diskutil list external
mount | grep Volumes
```

`Windows_FAT_32` で、`msdos` として fskit 経由でマウントされていた。ここがポイントで、最近の macOS は FAT/exFAT のマウントに [fskit](https://developer.apple.com/documentation/fskit) という新しい仕組みを使っている。そして体感的に、**この FAT32（msdos）の書き込みパスが異常に遅い**。

ちなみに `noatime`（アクセス時刻を書き込まない）は既に効いていたので、ここはいじる余地なし。

フォーマットを exFAT に変えてみる。中身は要らなかったので一気に消した。

```bash
diskutil eraseDisk ExFAT "USB" MBR /dev/disk2
```

で、同じ計測をやり直す。

| | FAT32 | exFAT |
|---|---|---|
| 書き込み | 0.9 MB/s | **18.6 MB/s** |
| 読み込み | 11 MB/s | **192 MB/s** |

書き込みが20倍、読み込みに至っては17倍。**犯人はほぼフォーマットだった**。

注意したいのは、exFAT も FAT32 と同じく fskit 経由でマウントされている点。つまり「fskitが遅い」のではなく、**fskitの中でも FAT32（msdos）の書き込みだけが極端に遅い**、というのが正確な表現になる。ここは自分でも雑に「FAT32×fskit」とまとめかけたんだけど、切り分けるとそうじゃない。

## ついでに Spotlight を止めたらさらに倍

もう一段。外付けドライブには macOS が勝手に Spotlight のインデックスを張りに来る。`mdworker` が裏で読み書きを続けるので、ただでさえ遅いフラッシュの帯域を食う。フォーマット直後の状態を見たら、案の定インデックスが有効に戻っていた。

```bash
mdutil -s /Volumes/USB        # Indexing enabled. と出る
mdutil -i off /Volumes/USB    # 止める
mdutil -E /Volumes/USB        # 既存インデックスを消す
touch /Volumes/USB/.metadata_never_index   # 再マウントしても張らせない
```

ファイル変更ログを書く `.fseventsd` も止めておく。

```bash
mkdir -p /Volumes/USB/.fseventsd && touch /Volumes/USB/.fseventsd/no_log
```

これで計測し直すと、書き込みが **18.6 → 37.3 MB/s** とまた倍になった。インデクサが残りの帯域を持っていっていたわけだ。

## 最終結果

![3段階の比較。書き込みは0.9→37.3MB/sで約41倍、読み込みは11→200MB/sで約18倍](/images/usb-speed-result.png)

| 段階 | 書き込み | 読み込み |
|---|---|---|
| FAT32 + Spotlight ON（元） | 0.9 MB/s | 11 MB/s |
| exFAT に変更 | 18.6 MB/s | 192 MB/s |
| + Spotlight / fseventsd 停止 | **37.3 MB/s** | **200 MB/s** |
| 改善倍率 | 約41倍 | 約18倍 |

効いた順にいうと、効果の大半は exFAT 化。Spotlight 停止はそこに上乗せで書き込みをもう一段持ち上げる、という関係だった。

## 計測の前提（ここ大事）

数字を鵜呑みにされても困るので条件を書いておく。

- ドライブは安物のUSBメモリ1本。良いSSDだとそもそも様子が違うはず。
- `dd` のゼロフィルなので、実ファイル（小さいの大量とか）だと数字は変わる。
- 読み込み計測で `sudo purge` がパスワード必須で使えなかったので、書き込み後に `diskutil unmount` → `mount` でアンマウント・再マウントしてページキャッシュを落としてから読んでいる。これをやらないと直前に書いたファイルがRAMキャッシュに残って「7GB/s」みたいな嘘の数字が出る。
- なので「読み込み200MB/s」はキャッシュ無しの実測。逆にいうと、この数字は素直に信じていい。

## 結論

USB 3.0対応のはずなのに遅い、というときは、まず `ioreg` でリンク速度を見て接続を切り分け、それが正常ならフォーマットを疑うと早い。**FAT32 のまま使っていると、最近の macOS では書き込みがびっくりするほど遅くなる**ことがある。中身を退避できるなら exFAT に切り替えて、ついでに Spotlight のインデックスを止めておくと、安物ドライブでもそれなりに使える速度になる。

ちなみに書き込み37MB/sはこの安ドライブのほぼ実力上限で、リンクは400MB/s出せるのにチップが追いつかない。ここから先はもう手がなくて、買い替え案件になる。
