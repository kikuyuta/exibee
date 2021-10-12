# ExiBee 3.0 コンセプト

- Date: 08 Aug 2021
- Author: [KIKUCHI, Yutaka](https://github.com/kikuyuta)

この文書は ExiBee 3.0 の考え方をまとめたものである。

# 基本コンセプト
- 産業用に用いることができる Nerves/Elixir が稼働するハードウェアを構築する
- 単体でも SBC (Single Board Computer) として使える CPU ボードを用意して、それに入出力デバイスを追加する形でモジュールを構成する
  - コンボ：CPU ボードに基本的な入出力を少しずつ多様に
  - DIO：CPU ボードにデジタル入出力を充分な数用意する
  - AIO：CPU ボードにアナログ入出力を充分な数用意する
- CPU ボード
  - CPUも含めて、コンボ、DIO、AIOの共通部分の機能を担う基板とする
  - 単体でも使える、手のひらサイズのボードとする
- Beagle Bone Black/Green, Pocket Beagle とソフト互換性をもたせるようにする
  - Nerves で firmware を作るときに export MIX_TARGET=bbb で作製できること
    - 入出力は同じにする必要はない。例えばデジタル出力でGPIOの番号を合わせる必要はない
- 出来たハードウェアはオープンソースにする。ライセンスは CC by-sa 4.0 で
  - 他の CC by-sa 4.0 を継承する可能性が高いので必然的にこれになると思われる

# CPUボード
ExiBee のベースになる共通の機能を実現する基板。
- 単体でも利用可能とする。ただし、入出力用のピンヘッダやコネクタは SBC として使うか、他の入出力基板と合わせて一体のモジュールとするかで違ってて良い
- CPU を含む周辺は TI AM335x 系の SiP (System in Package) を用いる
- SiP は温度範囲の違う2種類があるので、産業用の温度範囲の広い方を使う。
- SiP: [OSD3358-C-SiP](https://octavosystems.com/octavo_products/osd335x-c-sip/)
  - SoC: TI AM335x (ARM Cortex-A8 1GHz, 3Dアクセラレータ, PRU)
  - RAM: 512MB DDR3L
  - eMMC: 4GB
  - EEPROM: 4KB
  - MEMS 24MHz Oscillator
  - PMIC: TPS65217C, LDO: TL5209, Passives
  - BGA 20x20 1.27mm Grid, 27mmx27mm
  - Temp: -25°C〜70°C
    - 殆どのチップは -40°C〜85°C を満たすが一部のチップがこれを満たさない可能性がある

モードが有るSiPピンについては以下とする。
- 基本リセットモード(モード番号7)で用いる
  - BeagleBone でモードを変えているのものは同様に変更する
  - 機能追加により使いたい機能が衝突するピンについては、BeagleBone とのファームウェア互換性が維持できるように割当てる

CPUボードからコンボ・DIO・AIOに接続するコネクタについては以下とする。
- フレキシブルケーブルでやりとりするためのコネクタ
- JTAG用 [cTI（試作機に実装、本番では実装しない）](http://software-dl.ti.com/ccs/esd/documents/xdsdebugprobes/emu_jtag_connectors.html)
- 参考：表面実装コネクタ
  - [基板対基板コネクタ](https://www.hirose.com/product/document?clcode=CL0537-0731-3-86&productname=DF12(3.0)-60DP-0.5V(86)&series=DF12&documenttype=Catalog&lang=en&documentid=D31693_ja)

セキュリティチップについては [Microchip ATECC608A](https://www.microchip.com/wwwproducts/en/ATECC608A)を実装する。
- I2C2 につなげる（BBB/BBG の P9 に出ている。また BBG の grove に出ている）
  - I2C0 はEPROMとPMONにつながっている
  - 参考：Raspberry Pi では I2C1 につながってる
- Nerves Hub 用の [Nerves Key](https://github.com/nerves-project/nerves_system_bbb#nerveskey) に該当

ウォッチドッグタイマについては、BBB/BBG 同様にCPUのタイマーを用いることとし、専用のチップを搭載しない。

## 電源
電源については、以下のどちらか（もしくは両方）がつながったら稼働すること。
- PoE (IEEE802.3af: DC 48V)
- DC 24V
ただし、各モジュールはPoEで稼働するものとして、DC24Vは基板にだけ用意する。

以上から内部で用いる以下の電源を準備する。
- DC5V
- DC3.3V
- DC1.8V (ADC の reference 用)
- I/O で外部へ電源供給を行うチップの電源（アナログインタフェースの4〜20mAなど）

電源ダウン時にCPUボードをシャットダウンできる容量のキャパシタをもたせる。

### 電源ダウンに対する動作
JIS B3502 の 5.1.1 （瞬時停電は 5.1.1.3 PS2）を参考にする。
これは 10ms (50Hz で半サイクル)の電源断で、キャパシタから供給する電圧が各チップの定格電源電圧の最低レベルを守ること。
- これは SiP の VIN_BAT ピンへのバッテリーDC供給があればクリアできる。

以上の条件は、外部電源の PoE ないしは DC24V に対して CPU ボードの機能が維持されれば良い。I/O で外部への電源供給を行うためのチップへの電源ラインについてあこの限りでない。

なお、電源ダウン時CPUに割り込みをかけられるようにする。これは、電源電圧があらかじめ決められた値より低くなった場合にGPIO入力を変化させることで行う。

## 共通I/O
CPUボードの持つI/Oについては以下を準備する。

- LED
  - 電源：青ないしは緑
  - 緑、黄、橙、赤 USR0〜USR3, SiP GPMC\_a5〜a8 (gpio1_21〜24)
- RS232（デバッグ用コンソールポート, SiP UART0） x1
- micro SDカード用ソケット x1 (SiP MMC0)
- 有線LAN：100BASE-TX x1 (BBB/BBGと合わせるのに1Gbpsを目指さない)
  - B14 GPIO2_0 を MMC2_CMD で使う
- 無線LAN: （BBG Wireless に倣う。BBB Wireless とは異なることに注意）
  - WiFi: MMC2 系統を使う
  - BlueTooth: BBG wireless とは異なるアプローチになる
    - BR/EDR: Audio に関しては使わない
    - LE: UART4 で RTS/CTS なしで使う
- RTC: DS3231: I2C2で内部接続
  - ボタン電池でバックアップすること
- USB 2.0 Type C (SiP USB0) x1
- Grove コネクタ
  - UART2
  - I2C2

なお、最後の USB と Grove についてはCPUボード単体の場合の用途である。Combo, DIO, AIO のケースに入れた場合に、接続できる必要はない。

### デジタル共通
デジタル入出力については↑エッジと↓エッジのどちらについても割り込みを発生させること。I2Cチップでデジタル入出力を扱う場合は、GPIOを用意してソフトウェアで割り込み入力をチェックできるようにすること。

1. I2Cの入力変化でGPIOの入力を変化させる
1. GPIOの入力変化で割り込みが発生
1. 割り込みをソフトウェアで検出
1. ソフトウェアでどのI2Cの入力が変化したかを調べる

I2Cでデジタル出力を行う場合は、ループバック用（アンサ用）機能を持たせるため、デジタル出力の↑↓エッジで割り込みを発生させること。デジタル出力のエッジで割り込みが発生しないチップの場合は、別途デジタル入力用のチップを用意して、デジタル出力のエッジを検出できるようにすること。

## CPUボードから外部へのコネクタ
CPUボードからは以下の信号・電源がやりとりできるようにする。

- SiP 電源
- SiP I2C   x1: コンボ(DIO)、DIOボード用
- SiP SPI0  x1: コンボ(AIO)、AIOボード用
- SiP GPMC  x2: コンボ、DIOボードで I2C の割り込みを受ける
- 案から外すか検討中
  - SiP Ain0〜Ain5(6): コンボA/D用（コンボのAIOはSPIかI2C経由とする）
  - SiP GPMC(12): コンボDIO/DIOボード用GPIO（DIOはI2C経由とする）
  - SiP GPMC(20): DIOボード用GPIO （DIOはI2C経由とする）

- BBB/BBG P8/P9 コネクタとの互換性
  - P8 コネクタ相当は作らない。ただし物理的な基板の支持用にあってもよい。
  - P9 コネクタは BBB/BBG とコンパチになるようにする
    - ただし外部に機能を出せないピンは No Contact にする
	  - C-SiP から出てない機能（BBB/BBG と SoC が違うので am3358 のピンが出てないものがある）
	  - CPUボードで使用してて外部で使いようがない機能
	    - UART1 が RS485, UART4 が BLE に使われるので P9 に出さない。
        - SPI0, SPI1 はアナログボードの場合に使われるので P9 に出す。

## リセット、ブート順
起動時に関係する条件を以下とする。

以下のボタン・スイッチを用意して、コンボ・DIO・AIOケースにおいても操作できるようにする。
- リセットボタン (SoC NRESET_INPUT, BBG の S1 button)
- SDブート スライドスイッチ (SoC LCD_DATA2, BBG の S2 button)
- パワーボタン (SoC PWE_BUT, BBG の S3 button)

SDブート スライドスイッチを用意しない場合は、以下の順番でブートメディアにアクセスしてブートできるメディアがあったらブートするようにする。

1. SDカード
1. eMMC

### EEPROMブート名
BBG 同様にしておく。ただし後で書き換えられるように WP は有効(L)にしておく。

# Combo、DIO、AIO ユニット

コンボ・DIO・AIO は、CPU ボードに、入出力を拡張するボードを追加して、
[Phoenix Contact のケース](https://www.phoenixcontact.com/online/portal/jp?1dmy&urile=wcm%3apath%3a/jpja/web/main/products/subcategory_pages/Multifunctional_housings_ME-PLC_P-01-12-08/8706a764-5158-4867-9fa1-f11065f1af6e) に収めたもの。

## Combo（コンボ）

- シリアル：RS422/485 x1
  - 半二重 (UART1)
- 絶縁 DI x8
- 絶縁 DO x4
- AI x6
  - AI: 4-20mA, SiP SPI
  - ADC については Analog Devices の AD7175-8 を採用する。詳細については AIO の項を参照すること。なお、AD7175-8 はコンボでは1チップなので、CSはL固定で良い。
- AO x1
  - AO: 4-20mA, SiP SPI (or I2C)
    - 給電・無給電切替可能なこと
- DI, DO, AI, AO には対応する LED をつけること

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
給電・無給電切替可能なこと、絶縁しなくてよい。対応するLED表示をつけること。
- AI x16
- AO x16

ADC については Analog Devices の AD7175-8 を採用する。
AD7175-8 は 8入力なので同一SPIバスに接続するために CS を使う必要がある。
CS (chip select) を GPIO のいずれかのピンに接続するものとする。

### AD変換の性能

SiP のADCは以下の性能を持たせること。
- 12ビットの逐次比較型(SAR) ADC
- 毎秒200kサンプル（5μs/ch）
- 入力は、8:1アナログ・スイッチにより多重化（8chの一周で40μs）
- 低レート変換で50/60Hzの影響を除去するモード

SPI を用いる高速 ADC チップを用いた場合は Linux 側からのアクセスではおそすぎる可能性がある。以下の方法が考えられ、今回は PIC を用いるものとする。

- am3358 の A7 を使って SPI 経由でアクセスする
  - ADC のサンプリングレートを落として Elixir からアクセスして取りこぼさないようにする。
- am3358 の PRU を使う
  - 開発が大変、かつノウハウを貯めてもほかで使い潰しが効かない
- 別に CPU を設ける
  - 今回は PIC を使い、Linux 側からは I2C でアクセスするようにする

