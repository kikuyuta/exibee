# ExiBee 仕様（ちょっと設計も）

- Date: 19 Aug 2020
- Author: KIKUCHI, Yutaka
- Rev: 1.4

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
	  - [80pin x1 にするか、50pin x2 もしくは 60pin x2 とする](https://www.hirose.com/product/document?clcode=CL0537-0731-3-86&productname=DF12(3.0)-60DP-0.5V(86)&series=DF12&documenttype=Catalog&lang=en&documentid=D31693_ja)
      - [Armadillo 840m の DIMM コネクタも参照すること](https://manual.atmark-techno.com/armadillo-840/armadillo-840_product_manual_ja-1.10.0/ch04.html#sct.interface-layout-a840m)
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
	- 緑、黄、橙、赤 USR0〜USR3, SiP GPMC_a5〜a8 (gpio1_21〜24)
  - RS232（デバッグ用コンソールポート, SiP UART0） x1
  - USB 2.0 Type C (SiP USB0) x1
  - micro SDカード用ソケット x1 (SiP MMC0)
  - 無線LAN (WiFi/BLE) x1
	- TI [WL1835MOD](https://www.ti.com/product/WL1835MOD)
	  - レベル変換(1.8V <-> 3.3V)をして SPI1, UART3 GPIO
	    - これは BBG wireless の場合を踏襲
      - あるいは SiP の SDIO を使うか
- 電源
  - SiP VIN_AC: ドータボードコネクタからのDC5V
  - SiP VIN_USB: ボードの micro USB C コネクタからのDC5V
  - SiP VIN_BAT: ドータボードコネクタからのバッテリーDC
- コネクタ（SiP の信号を出す、GPIOを除いて46、GPIOは12か32）
  - SiP USB1( 6): 各ボードの USB 2.0 x1
  - SiP MII1(15): 各ボードの 100base-TX x1
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
