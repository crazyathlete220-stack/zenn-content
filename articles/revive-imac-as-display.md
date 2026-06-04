---
title: "2011年のiMacを、2枚目のディスプレイとして蘇らせた（macOS仮想ディスプレイ自作）"
emoji: "🖥️"
type: "tech"
topics: ["macos", "imac", "objectivec", "個人開発", "oss"]
published: true
---

## 作ったもの

押し入れに眠っていた2011年製のiMacを、メインMacの**2枚目のディスプレイ**として使えるようにしました。
そのために作った小さなツールを公開しています。

- GitHub: https://github.com/crazyathlete220-stack/oldmac-display

```sh
make
./oldmac-display --keep-alive
```

## なぜ作ったか

古いiMacって、捨てるのも忍びないし、かといって使い道もない。
「せめてサブモニタにできれば」と思って調べたら、これがことごとく茨の道でした。

- **2011 iMacはHDMI入力モニターにはできない**（映像入力端子がない）
- Apple公式の **Target Display Mode** は対応機種・OSの条件が厳しく、今のメインMac環境ではほぼ使えない
- **VNC / 画面共有**は動くものの、ラグが大きくて「ディスプレイ」としては使い物にならない

「古いハードを活かしたい」という、ただそれだけの動機。
なのに正攻法が全部詰んでいる。それなら作るしかない、となりました。

## 方針：macOS側に「存在しないディスプレイ」を作る

発想を変えました。物理的に繋ぐのを諦めて、**メインMacの中に仮想のディスプレイを作る**ことにしたんです。

macOSには `CGVirtualDisplay` という**非公開API**があり、これを使うと
「物理的には存在しないが、OSからは本物の外部ディスプレイに見える」画面を作れます。

```objc
@interface CGVirtualDisplay : NSObject
- (id)initWithDescriptor:(id)descriptor;
- (BOOL)applySettings:(id)settings;
@end
```

この仮想ディスプレイを作ってしまえば、あとは**その中身を古いiMacに見せるだけ**。
構成はこうなりました。

```
メインMac
  oldmac-display        ← CGVirtualDisplayで仮想ディスプレイを作成
  Sunshine              ← その仮想ディスプレイを低遅延ストリーミング
        │
        ↓ ネットワーク
2011 iMac (macOS 10.13)
  Moonlight             ← 受信してフルスクリーン表示
```

メインMacの「システム設定 > ディスプレイ」に **Old Mac Display** という画面が増え、
普通の外部モニタと同じようにウィンドウを並べられます。それを古いiMacに映している、というわけです。

## ハマりどころ

### 画面が"迷子"になる事故

仮想ディスプレイの配置をミスると、ウィンドウが見えない画面に飛んでいって戻せなくなります。
なので**デフォルトは45秒で自動終了する安全装置**を入れました。挙動を確認してから `--keep-alive` で常駐させます。

```sh
# 迷子になったら即停止
pkill -f oldmac-display
```

### 低遅延化：VNCからSunshine/Moonlightへ

最初はVNCで映していましたが、ラグが致命的でした。
ゲーム配信で使われる **Sunshine（送信）+ Moonlight（受信）** に切り替えたら、体感が一気に実用レベルに。

### 古いmacOSでは新しいMoonlightが動かない

2011 iMacのmacOS 10.13では、最新のMoonlight（5系/6系）はOSバージョン不足で起動しません。
**古いMoonlightビルドを使う**必要がありました。同じ轍を踏まないようREADMEに明記しています。

## 注意：非公開APIに依存している

`CGVirtualDisplay` は Apple の**非公開API**です。今は動きますが、将来のmacOSで変更・削除される可能性があります。
そこは割り切って「動くうちに楽しむ実験プロジェクト」として公開しています。

## おわりに

「古いiMacを捨てるのが面倒」から始まって、気づいたらmacOSの仮想ディスプレイを自作していました。
仮想ディスプレイ作成そのものは、古いMac用途に限らず、ヘッドレス環境やカスタム解像度のサブ画面など、わりと応用が効きます。

MITライセンスです。同じように古いMacを持て余している人の役に立てば。

⭐ https://github.com/crazyathlete220-stack/oldmac-display
