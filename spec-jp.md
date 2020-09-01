# ExiBee 仕様（ちょっと設計も）

- Date: 01 Sep 2020
- Author: [KIKUCHI, Yutaka](https://github.com/kikuyuta)
- Rev: 2.4

# 思いつく課題点
- 電源の構成をどうするか
  - ExiA (BP2.0) のコンボ・DIO・AIOへのDC3.3Vは[Armadillo840mの3.3V](https://manual.atmark-techno.com/armadillo-840/armadillo-840_product_manual_ja-1.10.0/ch18.html#sct.power-a840m)から供給されてる
    - Armadillo 840m の供給能力は 1.4A max もあるが C-SiP の TI TPS65217C では 3.3V 用 LDO4 を400mAしか供給できない
  - SiP からの電源供給で足りなければ CPU ボード上に電源を準備する

# 基本コンセプト
- PLC風な Nerves マシンを作成する
  - コンボ：CPU ボードに基本的な入出力を少しずつ多様に
  - DIO：CPU ボードにデジタル入出力をたくさん
  - AIO：CPU ボードにアナログ入出力をたくさん
- CPU ボード
  - CPUも含めてコンボ、DIO、AIOの共通部分の機能を担うボード
  - 電源はこのボードで主に扱う
  - 単体でも使えるような、手のひらボードに出来ることも検討する（オプション）
- Beagle Bone Black/Green, Pocket Beagle とソフト互換性をもたせるようにしたい
  - すなわち Nerves で firmware を作るときに export MIX_TARGET=bbb で作製できると良い
    - 入出力は同じにする必要はない。例えばデジタル出力でGPIOの番号を合わせる必要はない
- 出来たものはオープンソースにする。ライセンスは CC by-sa 4.0 で
  - 他の CC by-sa 4.0 を継承する可能性が高いので必然的にこれになるかと

# CPUボード
ExiBee のベースになる。また、できれば単体でも遊べるようにする。
- SiP: [OSD3358-C-SiP](https://octavosystems.com/octavo_products/osd335x-c-sip/)
  - SoC: TI AM335x (ARM Cortex-A8 1GHz, 3Dアクセラレータ, PRU)
  - MEMS 24MHz Oscillator
  - PMIC: TPS65217C, LDO: TL5209, Passives
  - BGA 20x20 1.27mm Grid, 27mmx27mm

SiP の調達について。
このシリーズは温度範囲の違う2種類しか出回ってない。
温度範囲の広い方を使う。
- [OSD3358-512M-ICB](https://www.digikey.com/product-detail/en/octavo-systems-llc/OSD3358-512M-ICB/1676-1005-ND/9608235)
  - Soc: TI AM3358 (ARM Cortex-A8 1GHz, 3Dアクセラレータ, PRU)
  - RAM: 512MB DDR3L
  - EEPROM: 4KB
  - eMMC: 4GB
  - Temp: -40°C to 85°C

モードが有るピンについては以下とする。
- 基本リセットモード(モード番号7)で用いる
- BeagleBone でモードを変えているのものは同様に変更する
- ヤムを得ない場合は別モードで（おそらくこれはないと想定）
  - 使いたいピン機能がバッティングして仕様を満たせいない可能性あり
    - これについては一旦考慮せずに仕様をつくって、ぶつかったらあとで検討

CPUボードからコンボ・DIO・AIOに接続するコネクタについては以下で検討する。
- フレキシブルケーブルでやりとりするためのコネクタ
- JTAG用 [cTI（試作機に実装、本番では実装しない）](http://software-dl.ti.com/ccs/esd/documents/xdsdebugprobes/emu_jtag_connectors.html)
- 参考：表面実装コネクタ
  - [基板対基板コネクタ](https://www.hirose.com/product/document?clcode=CL0537-0731-3-86&productname=DF12(3.0)-60DP-0.5V(86)&series=DF12&documenttype=Catalog&lang=en&documentid=D31693_ja)

セキュリティチップについては [Microchip ATECC608A](https://www.microchip.com/wwwproducts/en/ATECC608A)を実装する。
- I2C2 につなげる（BBB/BBG の P9 に出ている。BBG の grove に出ている）
  - I2C0 はEPROMとPMONにつながっている
  - 参考：Raspberry Pi では I2C1 につながってる
- Nerves Hub 用の [Nerves Key](https://github.com/nerves-project/nerves_system_bbb#nerveskey) に該当

ウォッチドッグタイマについては、BBB/BBG 同様にCPUのタイマーで良いと考える。
あえて持つなら MAX6359 を使う。

## 電源
以下が外部からの電源。これから基本的な電源を供給する。
以下のどちらか（もしくは両方）がつながったら稼働すること。
- PoE (IEEE802.3af: DC 48V)
- DC 24V

これから以下の電源を準備する。
- DC5V
- DC3.3V
- DC1.8V (ADC の reference 用)
- I/O で外部へ電源供給を行うチップの電源（例：4〜20mA用）

SiP には以下の電源ラインがある。
1. SiP VIN_AC: ボードのDC5V
1. SiP VIN_USB: ボードの micro USB C コネクタからのDC5V
1. SiP VIN_BAT: ドータボードコネクタからのバッテリーDC
   1. ○ 電源ダウン時にCPUボードをシャットダウンできる容量のキャパシタを持つ
   1. × CPUボードをリチウムイオン二次電池等である程度持たせるようにする

### 電源ダウンに対する動作
JIS B3502 の 5.1.1 に準拠、特に瞬時停電は 5.1.1.3 PS2 に準拠。
要は 10ms (50Hz で半サイクル)の電源断で、
電解コンデンサないしはキャパシタから供給する電圧が
各チップの定格電源電圧の最低レベルを守ること。
- これは前の節での VIN_BAT へのバッテリーDC供給があれば自動的にクリア。

この条件は、外部電源の DC24V に対して守られれば良い。
PoE に関しては今後の課題。
また、I/O で外部への電源供給を行うためのチップへの電源ラインについてもこの限りでない。

なお、電源ダウン時にGPIOのどれかを叩くこと。
これはCPUに割り込みをかけるため。

## 共通I/O
- LED
  - 電源：青
  - 緑、黄、橙、赤 USR0〜USR3, SiP GPMC\_a5〜a8 (gpio1_21〜24)
- RS232（デバッグ用コンソールポート, SiP UART0） x1
- micro SDカード用ソケット x1 (SiP MMC0)
- シリアル：RS422/485 x1
- 有線LAN：100BASE-TX x1 (BBB/BBGと合わせるのに1Gbpsを目指さない)
- RTC: DS3231: I2C2で内部接続
  - ボタン電池でバックアップ
  - （不採用）もし VIN_BAT でバッテリーバックアップするならそちらを使っても良い
- USB 2.0 Type C (SiP USB0) x1（Combo, DIO, AIO でつなげる位置でなくて良い）
- Grove コネクタ（Combo, DIO, AIO でつなげる位置でなくて良い）
  - UART2
  - I2C2

### デジタル共通
デジタル入出力については↑エッジと↓エッジのどちらについても割り込みを発生させること。I2Cチップでデジタル入出力を扱う場合は、GPIOを用意してソフトウェアで割り込み入力をチェックできるようにすること。

1. I2Cの入力変化でGPIOの入力を変化させる
1. GPIOの入力変化で割り込みが発生
1. 割り込みをソフトウェアで検出
1. ソフトウェアでどのI2Cの入力が変化したかを調べる

I2Cでデジタル出力を行う場合は、ループバック用（アンサ用）機能を持たせるため、デジタル出力の↑↓エッジで割り込みを発生させること。デジタル出力のエッジで割り込みが発生しないチップの場合は、別途デジタル入力用のチップを用意して、デジタル出力のエッジを検出できるようにすること。

## CPUボードから外部へのコネクタ

- SiP 電源
- SiP I2C   x1: コンボ(DIO)、DIOボード用
- SiP SPI0  x1: コンボ(AIO)、AIOボード用
- SiP GPMC  x2: コンボ、DIOボードで I2C の割り込みを受ける
- 案から外すか検討中
  - SiP Ain0〜Ain5(6): コンボA/D用（コンボのAIOはSPIかI2C経由とする）
  - SiP GPMC(12): コンボDIO/DIOボード用GPIO（DIOはI2C経由とする）
  - SiP GPMC(20): DIOボード用GPIO （DIOはI2C経由とする）
  - ？ SiP PMIC(14): 電源管理用（要る？）
  - ？ SiP RTC(2): 内部RTC制御用（要る？）

## ブート順
1. SDカード
1. eMMC
1. （ネットブート）

# Combo、DIO、AIO

コンボ・DIO・AIO は CPU ボードを入出力を拡張するボードに装着して 
[Phoenix Contact のケース](https://www.phoenixcontact.com/online/portal/jp?1dmy&urile=wcm%3apath%3a/jpja/web/main/products/subcategory_pages/Multifunctional_housings_ME-PLC_P-01-12-08/8706a764-5158-4867-9fa1-f11065f1af6e) に収めたもの。

## Combo（コンボ）

- 絶縁 DI x8
- 絶縁 DO x4
- AI x5
  - AI: 4-20mA, SiP SPI (or I2C)
- AO x1
  - AO: 4-20mA, SiP SPI (or I2C)
    - 給電・無給電切替可能なこと
- 対応する LED をつけること
	
## DIO（デジタル入出力）
フォトカプラ入出力とし、すべて絶縁すること。対応するLED表示をつけること。
- DI x16
- DO x16

SiP の I2C2 で制御する。候補チップMCP23017など。

### 入出力エッジによる割り込みについて
I2C の1チップで16入出力を扱うとして、入出力エッジの割り込みを発生させる。
割り込みは GPIO で受ける。
入力のほか、出力のアンサーバックも獲るので GPIO は2系統使う。

## AIO（アナログ入出力）
アナログ入出力はすべて 4〜20mA カレントループとする。
給電・無給電切替可能なこと絶縁しなくてよい。対応するLED表示をつけること。
- AI x16
- AO x16

### ADCチップの選択について
外付けのチップを使うので SiP の能力以上にしたい。

SiP のADCは以下の性能を持っている。
- 12ビットの逐次比較型(SAR) ADC
- 毎秒200kサンプル（5μs/ch）
- 入力は、8:1アナログ・スイッチにより多重化（8chの一周で40μsぐらい）

なので 12bit 25kS/s 程度と思って良い。
これを超えるとなると、解像度が 12bit を越えて 1chあたり40μs未満程度か。
この性能でポピュラーなチップとかがあると嬉しい。


# その他、検討先送り事項

## 温度環境
-40℃からにしようとするとチップを選ばないとならない。
少なくとも TI WL1835MOD は-20℃からなので、
現状の最低動作温度はこれに合わせざるをえない。

## IEEE1588 (PTP) を使うか
LAN 内のマシンの時計の同期精度があがるのは良い。
使うとしてどう使えるのか要調査 (NTP が使えなくなるので)。

## Security
AM335x に FuseFarm で固有IDがあるのを有効に使えないか

## watchdog reset
ファームウェアを更新した後に watch dog reset がかかったら
更新される前のファームウェアでブートするようにする… には何が必要？

## RTCについて
PMICのSLEEP状態で RTC 用の LDO1 のみに電源供給できそう。
C-SiP のマニュアルでは PMIC ver.C ではできないと明示。
PMIC のマニュアルでは ver に関係なくできそうに見えるが、
多くのweb上のドキュメントでうまくいかないようなことが書いてある

## 電池駆動
こんなのに乾電池を1〜3本直列にするのもありか。
http://akizukidenshi.com/catalog/g/gK-13065/

## 紫色について
今回は色に基づく名前を使うのをやめたが、Elixir 絡みということで、
コードネームに紫色を入れるべく検討した。
紫はいくつかの表現があって、Perple にするか Violet にするか Amethyst にするか調べてみた。
- [紫](https://ja.wikipedia.org/wiki/紫)
- [京紫](https://ja.wikipedia.org/wiki/京紫)
- [江戸紫](https://ja.wikipedia.org/wiki/江戸紫)

色の感じで Elixir 用に使うなら赤みがかった紫の京紫 purple が良い。
しかしながら、今回は紫をコードネームに使うのはやめてる。

あと、電源LEDやプリント基板のレジストを紫色にとも思ったが
無駄に高くなりそうなので将来の課題とする。
