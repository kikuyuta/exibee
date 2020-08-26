# ExiBee 仕様（ちょっと設計も）

- Date: 26 Aug 2020
- Author: [KIKUCHI, Yutaka](https://github.com/kikuyuta)
- Rev: 1.5

# 課題点
- CPUボードをBP2.0 (= Exineris Armadillo = ExiA) 同様の構成にする
  - DIMMボード同様の Pocket Exineris
  - それをマウントした CPU ボード
- 基板の入出力と基板同士の接続をどうするか
　- Pocket Exineris の入出力とCPUボードへのコネクタピン
    - 特に無線チップをどうするか
  - CPUボードの入出力とコンボ・DIO・AIOへのコネクタピン
- 電源の構成をどうするか
  - ExiA のコンボ・DIO・AIOへのDC3.3Vは[Armadillo840mの3.3V](https://manual.atmark-techno.com/armadillo-840/armadillo-840_product_manual_ja-1.10.0/ch18.html#sct.power-a840m)から供給されてる
    - 840m の供給能力は 1.4A max もあるが C-SiP の TI TPS65217C は 3.3V 用 LDO4 は400mAしか供給できない


# 基本コンセプト

- PLC風な Nerves マシンを作成する
  - コンボ：CPU ボードに基本的な入出力を少しずつ多様に
  - DIO：CPU ボードにデジタル入出力をたくさん
  - AIO：CPU ボードにアナログ入出力をたくさん
- CPU ボード
  - Pocket Exineris に必要な基本入出力を構成
  - 電源はこのボードで主に扱う
- Pocket Exineris
  - 単体でも使える小さなボード
  - PocketBeagle やラズパイ0W同程度の大きさで、より使いやすいのを目指す
- Beagle Bone Black/Green, Pocket Beagle とソフト互換性をもたせるようにしたい
  - すなわち Nerves で firmware を作るときに export MIX_TARGET=bbb で作製できると良い
- 出来たものはオープンソースにする。ライセンスは CC by-sa 4.0 で
  - 他の CC by-sa 4.0 を継承する可能性が高いので必然的にこれになるかと
 
# Pocket Exineris

- ExiBee のベースになるほか、単体でも遊べるようにする
  - モードが有るピンについて
    - 基本リセットモード(モード番号7)で用いる
    - BeagleBone でモードを変えているのものは同様に変更しても良い
    - ヤムを得ない場合は別モードで
    - 使いたいピン機能がバッティングして仕様を満たせいない可能性あり
      - これについては一旦考慮せずに仕様をつくって、ぶつかったらあとで検討
- コネクタ
  - ドータボード用コネクタ
    - 表面実装コネクタ
	  - [基板対基板コネクタ](https://www.hirose.com/product/document?clcode=CL0537-0731-3-86&productname=DF12(3.0)-60DP-0.5V(86)&series=DF12&documenttype=Catalog&lang=en&documentid=D31693_ja)
	  の80pin x1 にするか、50pin x2 もしくは 60pin x2 とする
      - [Armadillo 840m の DIMM コネクタ](https://manual.atmark-techno.com/armadillo-840/armadillo-840_product_manual_ja-1.10.0/ch04.html#sct.interface-layout-a840m)も参照すること
  - [JTAG用 cTI（試作機に実装、本番では実装しない）](http://software-dl.ti.com/ccs/esd/documents/xdsdebugprobes/emu_jtag_connectors.html)
- SiP: [OSD3358-C-SiP](https://octavosystems.com/octavo_products/osd335x-c-sip/)
  - SoC: TI AM335x (ARM Cortex-A8 1GHz, 3Dアクセラレータ, PRU)
  - MEMS 24MHz Oscillator
  - PMIC: TPS65217C, LDO: TL5209, Passives
  - BGA 20x20 1.27mm Grid, 27mmx27mm
- このシリーズは温度範囲の違う2種類しか出回ってない。温度範囲の広い [OSD3358-512M-ICB](https://www.digikey.com/product-detail/en/octavo-systems-llc/OSD3358-512M-ICB/1676-1005-ND/9608235) を使う。
  - Soc: TI AM3358 (ARM Cortex-A8 1GHz, 3Dアクセラレータ, PRU)
  - RAM: 512MB DDR3L
  - EEPROM: 4KB
  - eMMC: 4GB
  - Temp: -40°C to 85°C
- セキュリティ: [Microchip ATECC608A](https://www.microchip.com/wwwproducts/en/ATECC608A)
  - I2C2 につなげる（BBB/BBG の P9 に出ている。BBG の grove に出ている）
    - I2C0 はEPROMとPMONにつながっている
	- Raspberry Pi では I2C1 につながってる
  - Nerves Hub 用の [Nerves Key](https://github.com/nerves-project/nerves_system_bbb#nerveskey) に該当
- ウォッチドッグタイマ
  - BBB/BBG 同様にCPUのタイマーで良いのでは
    - あえて持つなら MAX6359 とかで
- 入出力（ボード上にあるもの）
  - LED
    - 電源：青
	- 緑、黄、橙、赤 USR0〜USR3, SiP GPMC\_a5〜a8 (gpio1_21〜24)
  - RS232（デバッグ用コンソールポート, SiP UART0） x1
  - USB 2.0 Type C (SiP USB0) x1
  - micro SDカード用ソケット x1 (SiP MMC0)
  - 無線LAN (WiFi/BLE) x1
	- TI [WL1835MOD](https://www.ti.com/product/WL1835MOD)
	  - レベル変換(1.8V <-> 3.3V)をして SiP SDIO を使う

- 電源
  1. SiP VIN_AC: ドータボードコネクタからのDC5V
  1. SiP VIN_USB: ボードの micro USB C コネクタからのDC5V
  1. SiP VIN_BAT: ドータボードコネクタからのバッテリーDC
- コネクタ（SiP の信号を出す、GPIOを除いて46、GPIOは12か32）
  - SiP USB1( 6): 各ボードの USB 2.0 x1 用
  - SiP MII1(15): 各ボードの 1000base-T/100base-TX x1 用
    - MII を持ち回るより PHY (LAN8710A等) を通した後が良いかも
  - SiP UART1(4): 各ボードの RS422/485 x1
  - SiP Ain0〜Ain5(6): コンボA/D用
  - SiP I2C2(2): DIOボード用
  - SiP SPI0(5): AIOボード用
  - SiP GPMC(12): コンボDIO/DIOボード用GPIO
  - SiP GPMC(20): DIOボード用GPIO
	- DIOボードはCPUボードのGPIOではなくI2C制御が良いかも
  - SiP PMIC(14): 電源管理用
  - ？ SiP RTC(2): 内部RTC制御用（要る？）
- 形状：チョコベビーのケースに収まること
  - 参考：
	- 56mm x 35mm x 5mm : pocket Beagle
	- 65mm x 30mm x 5mm : raspberry Pi zero

# CPUボード
pocket Exineris を用いて構成する。

## 共通 電源
- SiP VIN_AC（以下のどちらか（もしくは両方）がつながったら稼働すること）
  - PoE (IEEE802.3af: DC 48V)
  - DC 24V
- JIS B3502 の 5.1.1 に準拠、特に瞬時停電は 5.1.1.3 PS2 に準拠。
  - 要は 10ms (50Hz で半サイクル)の電源断で、電解コンデンサないしはキャパシタから
  - 供給する電圧が各チップの定格電源電圧の最低レベルを守ること
- 電源ダウン時にGPIOのどれかを叩く
  - CPUに割り込みをかけるため
- CPU ボードの SiP VIN_BAT に供給する電源をもつこと（以下から選択）
  1. 電源ダウン時にCPUボードをシャットダウンできる容量のキャパシタを持つ
  1. CPUボードをリチウムイオン二次電池等である程度持たせるようにする
  1. 拡張ボードも含めてリチウムイオン二次電池等である程度持たせるようにする

## 共通I/O
- LED 入出力に従って
- シリアル：RS422/485 x1
- シリアル：USB 2.0 type C x1 (SiP USB1) 
- 有線LAN：1000BASE-T x1
- RTC: DS3231（バッテリーバックアップ）

## ブート順
1. pocket Exineris に SDカードがあったらそれから
1. eMMC から
1. （ネットブート）

# Combo、DIO、AIO

コンボ・DIO・AIO は CPU ボードを入出力を拡張するボードに装着して 
[Phoenix Contact のケース](https://www.phoenixcontact.com/online/portal/jp?1dmy&urile=wcm%3apath%3a/jpja/web/main/products/subcategory_pages/Multifunctional_housings_ME-PLC_P-01-12-08/8706a764-5158-4867-9fa1-f11065f1af6e) に収めたもの。

## Combo（コンボ）

- 絶縁 DI x8
- 絶縁 DO x4
- AI x5
  - AI: 4-20mA, SiP Ain1〜Ain5
- AO x1
  - AO: 4-20mA, SiP Ain0 （給電・無給電切替可能なこと）
- 対応する LED をつけること。
	
## DIO（デジタル入出力）
- 絶縁 DI x16
- 絶縁 DO x16
- 対応する LED をつけること。
  - ？	SiP の GPIO を使うのでも良いが32pinを引き出すことになる
  - ？	SiP の I2C2 で MCP23017 等を使っても良い
    - ？ I2C で接続したときに割り込みを受けるようにするかどうか

## AIO（アナログ入出力）
- AI x16
- AO x16

### ADCチップの選択について
コンボ用の ADC には CPU ボード自身の SiP Ain を使うつもり。
- 12ビットの逐次比較型(SAR) ADC
- 毎秒200kサンプル（5μs/ch）
- 入力は、8:1アナログ・スイッチにより多重化（8chの一周で40μsぐらい）

なので 12bit 25kS/s 程度と思って良い。これがコンボ用。

AIOボードの能力は越えてほしい気がするので、
解像度が 12bit を越えて 1chあたり40μs未満の
ポピュラーなチップとかがあると嬉しい。


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
