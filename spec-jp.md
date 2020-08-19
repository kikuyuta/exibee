# ExiBee 仕様（ちょっと設計も）

- Date: 19 Aug 2020
- Author: KIKUCHI, Yutaka
- Rev: 1.4

# 基本コンセプト

- PLC風な Nerves マシンを作成する
  -- コンボ：CPU ボードに基本的な入出力を少しずつ多様に
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
 
# 
