+++
title = "space cubicsでRaspberry Pi組み込み開発をした話"
date = 2023-11-14
discription = "Example post showing callouts"

[taxonomies]
categories = ["Internship"]
tags = ["spacecubics"]

[extra]
lang = "ja"
toc = true
comment = true
math = true
mermaid = true
+++

[Zennの記事](https://zenn.dev/117324/articles/3f7ec8879660ed)もよろしくお願いします!!

---

# space cubicsとは？

[**space cubics**](https://spacecubics.com/)さんはJAXA発のベンチャー企業で、ごりごりの宇宙屋さんというよりも、宇宙でも問題なく動く高品質なプロセッサを開発している会社です。

ソフトウェア・FPGA・回路設計それぞれのプロフェッショナルが在籍していて、協力しあって素晴らしい製品を開発しています。

# 何をしたか？

2023年8月からCubeSat SC-Sat1という小型衛星に乗せるRaspberry Piボード (下図：ZeroとPicoが乗っている) の開発を行うインターンをさせていただきました。

{{ figure(src="https://storage.googleapis.com/zenn-user-upload/14b357f1ffd5-20231030.jpg", alt="SC-Sat1", caption="SC-Sat1") }}

{{ figure(src="https://storage.googleapis.com/zenn-user-upload/a837e827b95d-20230927.jpg", alt="scsat1-rpi", caption="scsat1-rpi") }}

このRaspberry Piモジュールは、SC-Sat1のカメラモジュールとして機能し、下記のミッションを達成するために開発が行われました。

- 宇宙で写真撮影を行い、指定した写真をdownlinkする
- Raspberry Pi Zero[^1] / Pico[^2] の放射線耐性を確認する
- SC Busのセンサーノードとして、Raspberry Pi Picoを使用する

メインのコンピュータとZero間はCAN Bus、ZeroとPico間はUARTで接続されており、その上を[CSP (Cubesat Space Protocol)](https://github.com/libcsp/libcsp)を載せたパケットが通信を行います。

僕は主にこの通信周りに関わりました。成果は[このリポジトリ](https://github.com/spacecubics/scsat1-rpi)にあります。

## きっかけ

7月末で院試が終了し、夏休みにはインターンを経験したいと考えていました。
そのとき、[KOBA789さんのツイート](https://twitter.com/KOBA789/status/1669729522663714816)を見かけ、「基板カッコいいなぁ、触ってみたいなぁ」と感じ、応募しました！
(低レイヤーのインターンがめずらしいというのも理由の1つです笑)

正直に言うと、宇宙については全くの未経験でしたが、高専の授業や趣味でRaspberry Piに触れた経験はあったので、不安と期待が入り混じった気持ちでした笑

## 応募～採用まで

TwitterのDMが開放されていなかったため、ホームページのお問い合わせフォームを使って連絡を取りました。すぐに返信があり、meetで面談することになりました。

技術系のインターンの面接は初めての経験で、緊張しましたが、面談ではRaspberry Piを触ったことがあること、低レイヤーに興味があること、高専で作成した作品についてなどを30分ほど話したかなと思います。

面談から約1週間後、採用のメールを受け取りました！とてもうれしかったですが、応募者が4名全員採用されたようです笑。インターン初年度で、来る者拒まずという感じだったと後で聞きました笑

## 8月中

### RaspberryPi OS上での開発

Zeroはリソースが潤沢なので、Linuxを動かすのですが、組み込みLinuxを作成するための[Yocto](https://www.yoctoproject.org/)というプロジェクトを用いました。

Yoctoはレシピの作成など、ややこしい部分が多いため、まずはRaspberryPi OS上で実装し、Yoctoに移植する形で開発がスタートしました。

僕がRaspberryPi OS上で行ったことは下記の内容です。

{% detail(title="ZeroがCAN Controller (MCP2517FD) を認識できるようにする") %}
使用したいデバイス情報は[RaspberryPi OSカーネル](https://github.com/raspberrypi/linux)に組み込まれているため、使用するように `config.txt` を書き換えます。

```
dtparam=spi=on
dtoverlay=mcp251xfd,oscillator=20000000,interrupt=25
```

`MCP251xFD` というデバイスツリーを適切なパラメータ (発振周波数や割り込みGPIOピン) を与えて有効化します。

このようにすると、 `ip` コマンドなどで CANインターフェースが表示され、RaspberryPi OSからCAN通信を行うことができるようになります。
{% end %}

{% detail(title="ZeroとPC間でCAN通信が行えることを確認する") %}
通常のPCはCANインターフェースを持たないので、[MicrochipのCAN bus analyzer](https://www.microchip.com/en-us/development-tool/apgdt002)を用います。

Windows環境よりもLinux環境のほうが後々便利そうなので、WSL (Windows Subsystem for Linux) を用いて環境を整備しました。(次節)

```sh
$ cangen can0
```
```sh
$ candump can0
  can0  0ED   [8]  37 D0 12 66 BB 9F 61 0D
  can0  5EB   [8]  02 B6 4B 50 F7 CC CF 02
  can0  07E   [8]  3A E2 A8 2C 26 BE 3F 6C
  can0  68F   [8]  42 00 1E 69 E4 6A 87 37
  can0  2A7   [2]  17 FE
  can0  57B   [1]  AB
...
```

上記のログの用にCSPの下位プロトコルであるCANで疎通することを確認しました。
{% end %}

{% detail(title="Zeroでlibcspのexampleコードを動かす") %}
[CSP](https://github.com/libcsp/libcsp)はCubesatなどの小規模なネットワーク間で通信を行うためのプロトコルで、現在も開発が盛んなOSSです。(サンプルプログラムが動かなくなることもしばしば...笑)

とりあえずサンプルプログラムを動かしてみようと思い、exampleフォルダ内のコード (`csp-server-client`) を動かしてみました。

現在のバージョンでは治っていますが、当時は自身のIDを設定できないといった不具合があり、動かすまで苦戦しました。

ビルドシステムは `waf` `mason` `cmake` の３つあり、今回は `waf` を用いてビルドしました。
```
$ ./waf configure --enable-examples --with-driver-usart=linux --with-os=posix --enable-can-socketcan build
```
ビルドが成功すれば、 `csp-server-client` という実行ファイルが生成されているはずで、
`-c can0` とCANインターフェースを指定すればCANを用いたCSP通信が可能です。
{% end %}

{% detail(title="details ZeroとPC間でCSP通信が行えることを確認する") %}
RaspberryPi ZeroとWSLそれぞれでlibcspのexampleをビルドし、 `csp-server-client` を動かせる状態になれば、下記のように疎通が確認できるはずです。
```sh
$ ./build/examples/csp_server_client -c can0 -a 10 -r 20
Initialising CSPINIT CAN: device: [can0], bitrate: 0, promisc: 0
Connection table
[00 0x55a505cbebe0] S:0, 0 -> 0, 0 -> 0 (17) fl 0
[01 0x55a505cbecf0] S:0, 0 -> 0, 0 -> 0 (18) fl 0
[02 0x55a505cbee00] S:0, 0 -> 0, 0 -> 0 (19) fl 0
[03 0x55a505cbef10] S:0, 0 -> 0, 0 -> 0 (20) fl 0
[04 0x55a505cbf020] S:0, 0 -> 0, 0 -> 0 (21) fl 0
[05 0x55a505cbf130] S:0, 0 -> 0, 0 -> 0 (22) fl 0
[06 0x55a505cbf240] S:0, 0 -> 0, 0 -> 0 (23) fl 0
[07 0x55a505cbf350] S:0, 0 -> 0, 0 -> 0 (24) fl 0
Interfaces
LOOP       addr: 0 netmask: 14 mtu: 0
           tx: 00000 rx: 00000 txe: 00000 rxe: 00000
           drop: 00000 autherr: 00000 frame: 00000
           txb: 0 (0B) rxb: 0 (0B)

CAN        addr: 10 netmask: 0 mtu: 248
           tx: 00000 rx: 00000 txe: 00000 rxe: 00000
           drop: 00000 autherr: 00000 frame: 00000
           txb: 0 (0B) rxb: 0 (0B)

Route table
0/0 CAN
Client task startedPing address: 20, result -1 [mS]
reboot system request sent to address: 20
Ping address: 20, result 10 [mS]
reboot system request sent to address: 20
Ping address: 20, result 50 [mS]
reboot system request sent to address: 20
Ping address: 20, result -1 [mS]
reboot system request sent to address: 20
Ping address: 20, result -1 [mS]
reboot system request sent to address: 20
Ping address: 20, result 98 [mS]
reboot system request sent to address: 20
Ping address: 20, result -1 [mS]
reboot system request sent to address: 20
Ping address: 20, result -1 [mS]
reboot system request sent to address: 20
...
```

```sh
$ ./build/examples/csp_server_client -c can0 -a 20
Initialising CSPINIT CAN: device: [can0], bitrate: 0, promisc: 0
Connection table
[00 0x2d438] S:0, 0 -> 0, 0 -> 0 (17) fl 0
[01 0x2d4e8] S:0, 0 -> 0, 0 -> 0 (18) fl 0
[02 0x2d598] S:0, 0 -> 0, 0 -> 0 (19) fl 0
[03 0x2d648] S:0, 0 -> 0, 0 -> 0 (20) fl 0
[04 0x2d6f8] S:0, 0 -> 0, 0 -> 0 (21) fl 0
[05 0x2d7a8] S:0, 0 -> 0, 0 -> 0 (22) fl 0
[06 0x2d858] S:0, 0 -> 0, 0 -> 0 (23) fl 0
[07 0x2d908] S:0, 0 -> 0, 0 -> 0 (24) fl 0
Interfaces
LOOP       addr: 0 netmask: 14 mtu: 0
           tx: 00000 rx: 00000 txe: 00000 rxe: 00000
           drop: 00000 autherr: 00000 frame: 00000
           txb: 0 (0B) rxb: 0 (0B)

CAN        addr: 20 netmask: 0 mtu: 248
           tx: 00000 rx: 00000 txe: 00000 rxe: 00000
           drop: 00000 autherr: 00000 frame: 00000
           txb: 0 (0B) rxb: 0 (0B)

Route table
0/0 CAN
Server task started
Packet received on MY_SERVER_PORT: Hello world A
Packet received on MY_SERVER_PORT: Hello world B
Packet received on MY_SERVER_PORT: Hello world C
Packet received on MY_SERVER_PORT: Hello world D
Packet received on MY_SERVER_PORT: Hello world E
Packet received on MY_SERVER_PORT: Hello world F
Packet received on MY_SERVER_PORT: Hello world G
Packet received on MY_SERVER_PORT: Hello world H
...
```
同時に送っているpingのRTTがおかしなことになっている場所がいくつかありますが、CSPで疎通が行えていることが確認できました。

また、このサンプルコードは `reboot` コマンドを強制的に送りつけるので送信側に管理者権限があれば再起動をし続けるということに...笑
{% end %}

### WSLからCAN通信を行うセットアップ

長くなりそう (+ 需要が多少はありそう笑) なので、別記事でまとめたいと思います。

簡単に書くとCAN関連のカーネルモジュールを入れたWSLのカスタムカーネルをビルドし、WSLからCAN bus analyzerを認識できるようにすればうまくいきました。

### Yoctoへの移植

Yoctoはbitbakeというパッケージ管理システムを用いて組み込みLinuxをビルドします。
Linuxにほしい機能を追加するには、bitbakeが認識できるレシピを作成する必要があります。
今回作成したレイヤー・レシピは次のようになります。

{% detail(title="meta-mcp2517fd") %}
[このレイヤ](https://github.com/spacecubics/scsat1-rpi/tree/main/zero/meta-mcp2517fd)は、MCP2517FD用のデバイスツリーをYoctoでビルドするカーネルに組み込むものです。


実際に追加するデバイス情報は下記の内容です。
```
/*
 * Device tree overlay for mcp2517fd/can0 on spi0.0
 */

/dts-v1/;
/plugin/;

/ {
    compatible = "brcm,bcm2835";

    /* enable spi for spi0.0 */
    fragment@0 {
        target = <&spi0>;
        __overlay__ {
            status = "okay";
        };
    };

    /* disable spi-dev for spi0.0 */
    fragment@1 {
        target = <&spidev0>;
        __overlay__ {
            status = "disabled";
        };
    };

    /* the interrupt pin of mcp2517fd */
    fragment@2 {
        target = <&gpio>;
        __overlay__ {
            mcp2517fd_int_pin: mcp2517fd_int_pin {
                brcm,pins = <25>;       // use GPIO25 for interrupt pin
                brcm,function = <0>;    // BCM2835_FSEL_GPIO_IN
            };
        };
    };

    /* the clock/oscillator of mcp2517fd */
    fragment@3 {
        target-path = "/clocks";
        __overlay__ {
            /* external oscillator of mcp2517fd on SPI0.0 */
            clk_mcp2517fd_osc: clk_mcp2517fd_osc {
                compatible = "fixed-clock";
                #clock-cells = <0>;
                clock-frequency  = <20000000>;  // use 20MHz oscillator
            };
        };
    };

    /* the spi config of the can-controller itself binding everything together */
    fragment@4 {
        target = <&spi0>;
        __overlay__ {
            /* needed to avoid dtc warning */
            #address-cells = <1>;
            #size-cells = <0>;
            can0: mcp2517fd@0 {
                reg = <0>;
                compatible = "microchip,mcp2517fd";
                pinctrl-names = "default";
                pinctrl-0 = <&mcp2517fd_int_pin>;
                spi-max-frequency = <20000000>;     // up to 20MHz SPI clock speed on mcp2517fd
                interrupt-parent = <&gpio>;
                interrupts = <25 8>;                // IRQ_TYPE_LEVEL_LOW : active low
                clocks = <&clk_mcp2517fd_osc>;
            };
        };
    };

    __overrides__ {
        oscillator = <&clk_mcp2517fd_osc>,"clock-frequency:0";
        spimaxfrequency = <&can0>,"spi-max-frequency:0";
        interrupt = <&mcp2517fd_int_pin>,"brcm,pins:0",<&can0>,"interrupts:0";
    };
};
```
このコードは、Linuxカーネル内の `mcp2515-overlay.dts` や `mcp251xfd-overlay.dts` を参考にして、今回必要な部分のみを取り出したものです。

このファイルをRaspberryPiのLinuxをビルドするレシピ(`linux-raspberrypi_*.bb`)を実行する際に追加するように `.bbappend`ファイルを作成しました。
```
SRC_URI += "file://mcp2517fd-overlay.dts;subdir=git/arch/${ARCH}/boot/dts/overlays"
...
```
対象のファイルをsubdirに配置できます。他にも、patchを用いた追加方法もあります。
{% end %}

{% detail(title="meta-libcsp") %}
[このレイヤ](https://github.com/spacecubics/scsat1-rpi/tree/main/zero/meta-libcsp)は、[libcsp](https://github.com/libcsp/libcsp) を使用するレシピのために共有ライブラリ・ヘッダファイルを作成するためのものです。

今回ビルド対象は、github上にあるので、下記のようにレシピを書きgithubからクローンするようにしました。
```
SRCREV = "${AUTOREV}"
SRCBRANCH = "develop"
SRC_URI = "git://github.com/libcsp/libcsp.git;protocol=https;branch=${SRCBRANCH};"
```
このレシピは、URLが示すリポジトリの `develop` ブランチの最新のコミットを取って来ます。
そして、ビルド方法にcmakeを指定することで、build → installまで行ってくれます (installターゲットがなければ当然実行されない)。
```
inherit cmake ...
```
ビルド後のファイルの置き場所を指定する必要があるので、下記のように目的別に指定しました。
```
FILES:${PN}-dbg += "${libdir}/.debug"
FILES:${PN}     += "${libdir}/libcsp.so"
FILES:${PN}-dev += "${includedir}/csp"
```
それぞれ、debug用、共有ライブラリ使用、ヘッダを使用する開発用と分けています。
{% end %}

{% detail(title="meta-csp (wip)") %}
[このレイヤ](https://github.com/KantaTamura/scsat1-rpi/tree/wip/csp-app/zero/meta-csp)は、libcspを使用したレシピのためのもので、現在開発途中です。

現状は、`csp-server-client`のサーバ側と同様の機能を持つプログラムをsystemdで起動するところまで実装しています。
今後、ルータ機能やカメラコマンドを発火させる機能を増やしていく予定です。
{% end %}

{% detail(title="meta-arducam (wip)") %}
[このレイヤ](https://github.com/spacecubics/scsat1-rpi/tree/wip/arducam/zero/meta-arducam)は、Yoctoでビルドするカーネルでカメラモジュールを認識できるようにするための、カーネルモジュールとデバイスツリーを追加するためのもので、現在開発途中です。
{% end %}

## 9月中

### 札幌オフィス訪問

インターン期間中space cubics札幌オフィスに招待されたので、3日間現地で作業しました！

札幌オフィスではlibcspをyoctoに移植する作業をしながら、"普段どんなソフトウェアを使って開発しているのか"や"pull requestを整理してマージする方法"など、現場で開発しているならではの話を聞いたり、僕がおすすめしているソフトウェアを紹介したりしていました。
個人開発では触ったことの無いgitコマンドなど、色々なことを教えていただき、とても楽しかったです。

オフィスは机の上にpic-kitなどが無造作においてあったり、オシロスコープや4Kディスプレイがそれぞれの机の上に備え付けられていたりと、個人的にとても理想的な作業環境だな...と憧れました笑

高専では同じような環境で作業していたのですが、大学に進学してからはハードウェアに触る機会は減って行ってしまったのでとても楽しかったです！

ご飯にもたくさん誘って頂いたので一部の記録を紹介します。

{% detail(title="オフィス訪問している間の写真など... (飯テロ注意！)") %}
{{ figure(src="https://storage.googleapis.com/zenn-user-upload/34bd1d504d79-20231003.jpg", alt="札幌オフィス", caption="space cubics札幌オフィス - 3部屋ほど借りられていて、今後部屋を合併するらしいです") }}

{{ figure(src="https://storage.googleapis.com/zenn-user-upload/2c3fcb2634ed-20231114.jpg", alt="ジンギスカン", caption="初日はジンギスカンを食べに行きました！ 初ジンギスカン美味しかったです！") }}

{{ figure(src="https://storage.googleapis.com/zenn-user-upload/bfdb0d5dd56c-20231114.jpg", alt="linuxビルド", caption="持ち込んだノートPCにWSL(enable CAN)をセットアップしてなかったので夜中にビルド") }}

{{ figure(src="https://storage.googleapis.com/zenn-user-upload/fe790407a60a-20231114.jpg", alt="スープカレー", caption="2日目お昼はスープカレーを食べました。スパイシーで美味しかったです！") }}

{{ figure(src="https://storage.googleapis.com/zenn-user-upload/0ec8be27ff3e-20231114.jpg", alt="サンマ炊き込みご飯", caption="3日目(最終日)の晩ごはんで旬のサンマを食べました！") }}
{% end %}

札幌オフィスでの作業は水木金の３日間だけでしたが、なんと追加で数日間のホテルも予約していただき、観光もすることができました！とてもありがたかったです！

{% detail(title="札幌観光") %}
{{ figure(src="https://storage.googleapis.com/zenn-user-upload/304d1d7a68b7-20231114.jpg", alt="夜パフェ", caption="札幌では夜パフェ(締めパフェ)が流行っているらしいです(22:00頃)") }}

{{ figure(src="https://storage.googleapis.com/zenn-user-upload/be932a19d3c6-20231114.jpg", alt="AOAO", caption="AOAOというきれいな水族館がホテルの近くにあったので観光") }}

{{ figure(src="https://storage.googleapis.com/zenn-user-upload/2582511a75bb-20231114.jpg", alt="藻岩山夜景", caption="新日本3大夜景に選ばれた藻岩山山頂展望台からの景色です(極寒でした...)") }}
{% end %}

とても充実した3日間(+1日)で、一瞬で過ぎ去ってしまってもっと作業していたかったな...と帰りの飛行機で思っていました。

### libcspへのコミット

yoctoのビルドではcmakeを使用するのですが、libcspのcmakeにはヘッダ・共有ライブラリをインストールする機能がなかったので、この部分を追加し、[マージ](https://github.com/libcsp/libcsp/pull/446)されました。

実際に書いたコード量は短いですが、linuxデフォルトの共有ライブラリロード方法やcmakeの複雑な記法など、ためになることがたくさんありました。

OSSへのコントリビューションの経験がなかったので、非常に良い経験をさせていただいてありがたかったです！

# 学んだこと

今回のインターンでは本当に知らないことが多く、下に調べて学んだことの一部を示します。

- DeviceTree (Overlay)
- Kernel Module
- Yocto (bitbake)
- CAN

他にも細々としたことを調査したりしていました。

インターン初日のミーティングで「はじめはわからないことが多いと思うから一日調査に使ってもいいよ」と言ってくださったので、わからないことは基本的に調べて実践を繰り返して学びました。
おそらく、ミーティングなどで直接聞けばみなさん既知の内容なので、快く教えてくださると思いますが、個人的に"調べる力"や"調べたい内容以外の知識を得られる[^3]"ということが大切だと思い、"わからないことも自分で十分に調べて、それでも分からなければ聞く"ということを心がけました。

8月中旬などはずっと調べていて、あまり進捗がなかったですが、その経験が8月後半・9月での実装で非常に役立ちました。

# 感想

蓋を開けてみれば調べては実践の繰り返しをしていた2ヶ月間でしたが、知らない知識がたくさんあることやそれを学ぶ楽しさがたくさんありました。
カーネルや通信の仕組みなど、まだまだ知らないことが多く残っているので、これからも勉強しながら低レイヤー関連を触っていきたいです。

インターン期間中は、研究室のミーティングなどで予定変更が多かったのにも関わらず、調整してくださり、とてもありがたかったです。

ありがたいことに10月以降もお手伝いという形で続けることになりましたので、残りの実装を行っていきたいと思います！ (10月はCSP用のwireshark dissectorを書いたりしていました)

来年以降に和歌山で打ち上げ予定とのことなので、ぜひ実際に見てみたいです！

### 追記 - 困ったこと

困ったことはほとんどありませんでしたが、強いて言うなら来年以降、インターン用の応募欄をホームページに作成していただけると、応募の際のハードルが下がって良いかな？と思いました。

# おわりに

来年もサマーインターンを募集するみたいなので、興味を持っていただいた方はぜひ応募してみてください！

最後になりますがインターンを受け入れていただいたspace cubicsの皆様、本当にありがとうございます！もう少しお世話になります！

---

[^1]: RaspberryPi Zeroはカメラ撮影とメインのコンピュータからの通信を自身とPicoに振り分けるルータ的役割を持っている。

[^2]: RsapberryPi Picoは温度センサを監視する役割を持っている。

[^3]: 例えば、DeviceTreeを調べている過程でKernelがどのようにロードしていくのか...といったことが学べました。
