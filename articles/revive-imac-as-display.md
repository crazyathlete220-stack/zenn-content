---
title: "2011年のiMacを、2枚目のディスプレイとして蘇らせた（macOS仮想ディスプレイ自作）"
emoji: "🖥️"
type: "tech"
topics: ["macos", "imac", "objectivec", "個人開発", "oss"]
published: true
---

2011年製のiMacが押し入れに眠ってる。デザインが好きすぎて、どうしても捨てられない。

でも置いておくだけなのも勿体ない。せめて2枚目のディスプレイにできないかな、と思って調べ始めたら、これが思った以上に詰んでました。

作ったものはこれです。

https://github.com/crazyathlete220-stack/oldmac-display

## 正攻法が全部ダメだった

まず、2011 iMacには映像入力がない。HDMIモニター化はできません。

じゃあApple公式の **Target Display Mode** は？ と思ったけど、対応機種とOSの条件が厳しくて、今のメインMacからは使えませんでした。

次に **VNC（画面共有）**。これは一応映る。映るんだけど、ラグが酷すぎて「ディスプレイ」としては正直きつい。マウスを動かすたびにモタつくモニタは、もう要らない。

そんなわけで、普通のやり方は全滅でした。

## OS内に「存在しない画面」を作る

発想を変えて、物理的に繋ぐのを諦めました。

代わりに、**メインMacの中に仮想のディスプレイを作る**ことにします。macOSには `CGVirtualDisplay` という非公開APIがあって、これを使うと「実在しないけど、OSからは本物の外部モニタに見える」画面を作れます。

```objc
@interface CGVirtualDisplay : NSObject
- (id)initWithDescriptor:(id)descriptor;
- (BOOL)applySettings:(id)settings;
@end
```

これさえ作れれば、あとはその中身を古いiMacに送るだけ。最終的にこういう構成に落ち着きました。

```
メインMac
  oldmac-display   ← 仮想ディスプレイを作る
  Sunshine         ← その画面を低遅延で配信
        │
        ↓
2011 iMac (macOS 10.13)
  Moonlight        ← 受信してフルスクリーン表示
```

メインMacの「システム設定 > ディスプレイ」に **Old Mac Display** が増えて、普通の外部モニタと同じようにウィンドウを並べられます。それを古いiMacに映してる、という仕組み。

## 詰まったところ

**画面が迷子になる。** 仮想ディスプレイの配置をミスると、ウィンドウが見えない場所に飛んで戻せなくなります。怖いので、デフォルトは45秒で勝手に終了するようにしました。挙動を確認してから `--keep-alive` で常駐させます。迷子になったら `pkill -f oldmac-display` で強制終了。

**VNCはやっぱり遅い。** 最初はVNCで映してたけど、前述のとおりラグが致命的。ゲーム配信で使う **Sunshine + Moonlight** に替えたら、体感が一気に実用レベルになりました。

**古いmacOSだと新しいMoonlightが動かない。** 2011 iMacの10.13では最新のMoonlightが起動せず、古いビルドが必要でした。同じ罠にハマる人がいそうなのでREADMEに書いてあります。

## 注意点

`CGVirtualDisplay` は非公開APIなので、将来のmacOSで動かなくなる可能性があります。そこは割り切って「動くうちに遊ぶ」スタンスです。

あと、仮想ディスプレイを作る部分自体は古いMacに限らず、ヘッドレス環境や好きな解像度のサブ画面とかにも応用が効くと思います。

結果としては、そこそこ使える。何より、好きなiMacがまだ現役なのが嬉しい。MITなので、同じように古いMacを持て余してる人はどうぞ。

https://github.com/crazyathlete220-stack/oldmac-display
