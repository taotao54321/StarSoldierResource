+++
title = "敵AI"
date = 2021-06-10
updated = 2021-06-12
+++

ほとんどの敵のAIはいわゆる[バイトコード](https://ja.wikipedia.org/wiki/%E3%83%90%E3%82%A4%E3%83%88%E3%82%B3%E3%83%BC%E3%83%89)で実装されている。  
つまり、直接 6502 で書くのではなく、「ある方向へ移動する」「弾を撃つ」などの命令を備えたバイトコードで記述し、それを 6502 で書かれた仮想マシン上で動かしている。  
この方式の最大の利点は「処理の中断」が可能なことだろう。例えば「30フレーム横へ動き、60フレーム下へ動き、...」などといった処理を 6502 で直接書くと、多くの場合複雑なステートマシンを実装することになり面倒である。

バイトコードでなく専用ルーチンで動いている敵はルイド(ID: 0x11), リューク(ID: 0x14), ソープラー(ID: 0x21), スターブレイン(ID: 0x22, 0x23)のみ。  
ここではバイトコードに関する解説のみ行う。

以下、バイトコードで実装された敵のうちジェリコ、ラザロを「ボス」、それ以外を「ザコ」と総称する。

## ツール

バイトコードの[アセンブラ/逆アセンブラ](https://github.com/taotao54321/starsoldier-bytecode)を作成した。以下、命令のニーモニックはこのアセンブラを基準とする。

[playground](https://taotao54321.github.io/starsoldier-bytecode-playground/) で実際のバイトコードの動作が見られる。

## 仮想マシン

敵1体ごと(ボスの場合、1パーツごと)に仮想マシンが割り当てられる。

ここでは仮想マシンの実装をやや抽象化して記述する。また、レジスタ名などは筆者が独自に命名したものである。

### アドレス空間

仮想マシンのアドレス空間は 8bit で、ROM 上のバイトコードがマップされる。このアドレス空間は読み取り専用である。

### レジスタ

仮想マシンは以下のレジスタを持つ:

| 名前              | 型     | 用途                                           |
| --                | --     | --                                             |
| `pc`              | `u8`   | プログラムカウンタ                             |
| `loop_counter`    | `u8`   | ループカウンタ                                 |
| `loop_start_addr` | `u8`   | ループ開始アドレス                             |
| `jump_on_damage`  | `u8`   | ザコの場合:被弾時のジャンプ先<br>ボスの場合:HP |
| `sleep_timer`     | `u8`   | 仮想マシンのスリープタイマー                   |
| `homing_timer`    | `u8`   | 自機追尾タイマー                               |
| `inv_x`           | `bool` | x 方向の移動を反転                             |
| `inv_y`           | `bool` | y 方向の移動を反転                             |

ループ関連レジスタが1組しかないことからわかるように、1重ループのみサポートしている。

### 仮想マシンの動作

毎フレーム以下の動作を行う:

* `sleep_timer` レジスタが非0ならばデクリメントして戻る。
* `homing_timer` レジスタが非0ならば自機を[追尾する方向](@/direction/index.md#aim)へ移動して戻る。
* 以上いずれにも該当しなければ、`pc` の指すアドレスから 1 ステップ(実行を中断する命令まで)実行する。
* 上記とは別に、衝突処理において自機の弾と衝突したとき、
  - ザコの場合、`jump_on_damage` レジスタが非0ならば `pc` をそのアドレスに変更し、さもなくば破壊される。
  - ボスの場合、`jump_on_damage` レジスタをHPとして扱い、この値が非0ならばデクリメントし、さもなくば破壊される。  
    HPが 0 のときに被弾すると破壊されるので、`(HP)+1` 発の撃ち込みが必要であることに注意。

### 命令セット

仮想マシンは以下の命令を持つ:

| オペコード            | ニーモニック                                   | 機能                                  | 実行の中断 |
| --                    | --                                             | --                                    | --         |
| `0x00..=0x3F`         | [`move`](#op-move)                             | 敵を指定方向へ移動                    | Yes        |
| `0x40`                | [`jump`](#op-jump)                             | ジャンプ                              | No         |
| `0x41..=0x4F`         | [`set_sleep_timer`](#op-set-sleep-timer)       | `sleep_timer` を設定                  | Yes        |
| `0x50`, `0x52..=0x5F` | [`loop_begin`](#op-loop-begin)                 | 指定回数のループを開始                | No         |
| `0x51`                | [`loop_end`](#op-loop-end)                     | ループ末尾                            | No         |
| `0x60..=0x6F`         | [`shoot_direction`](#op-shoot-direction)       | 方向指定弾を撃つ (未使用)             | No         |
| `0x70..=0x7F`         | [`set_sprite`](#op-set-sprite)                 | 敵メタスプライトIDを設定              | No         |
| `0x80..=0x8F`         | [`set_homing_timer`](#op-set-homing-timer)     | `homing_timer` を設定                 | No         |
| `0x90..=0x93`         | [`set_inversion`](#op-set-inversion)           | `inv_x`, `inv_y` を設定               | No         |
| `0xA0`                | [`set_position`](#op-set-position)             | 敵座標を設定                          | No         |
| `0xA1`                | [`set_jump_on_damage`](#op-set-jump-on-damage) | `jump_on_damage` を設定               | Yes        |
| `0xA2`                | [`increment_sprite`](#op-increment-sprite)     | 敵メタスプライトIDをインクリメント    | No         |
| `0xA3`                | [`decrement_sprite`](#op-decrement-sprite)     | 敵メタスプライトIDをデクリメント      | No         |
| `0xA4`                | [`set_part`](#op-set-part)                     | 敵パーツ番号を設定                    | No         |
| `0xA5`                | [`randomize_x`](#op-randomize-x)               | 敵座標x をランダムに変更              | No         |
| `0xA6`                | [`randomize_y`](#op-randomize-y)               | 敵座標y をランダムに変更              | No         |
| `0xB0`                | [`bcc_x`](#op-branch)                          | `(敵座標x) <  (自機座標x)` ならば分岐 | No         |
| `0xB1`                | [`bcs_x`](#op-branch)                          | `(敵座標x) >= (自機座標x)` ならば分岐 | No         |
| `0xB2`                | [`bcc_y`](#op-branch)                          | `(敵座標y) <  (自機座標y)` ならば分岐 | No         |
| `0xB3`                | [`bcs_y`](#op-branch)                          | `(敵座標y) >= (自機座標y)` ならば分岐 | No         |
| `0xC0..=0xCF`         | [`shoot_aim`](#op-shoot-aim)                   | 自機狙い弾を撃つ                      | No         |
| `0xF0`                | [`restore_music`](#op-restore-music)           | BGMをデフォルトに戻す                 | No         |
| `0xF1..=0xFF`         | [`play_sound`](#op-play-sound)                 | 指定した効果音を鳴らす                | No         |

#### `0x00..=0x3F` (`move`) {#op-move}

1Byte 命令。

敵をオペコードの表す[方向](@/direction/index.md)へ移動させる。  
`inv_x`, `inv_y` の設定に従って方向が反転する。

敵が[高速化フラグ](@/enemy-group/index.md#group-accel)を持つとき、特定条件下でスピードが増す。

実行を**中断**する。  
ただし、敵が[再行動フラグ](@/enemy-group/index.md#group-extra-act)を持つとき、特定条件下でさらに 1 ステップ実行する。

#### `0x40` (`jump`) {#op-jump}

2Byte 命令。`0x40 <addr>` の形式。

プログラムカウンタを `<addr>` に設定する。

実行を継続する。

#### `0x41..=0x4F` (`set_sleep_timer`) {#op-set-sleep-timer}

1Byte 命令。

`sleep_timer` の値を `4*(オペコード下位4bit)` にする。

実行を**中断**する。

#### `0x50`, `0x52..=0x5F` (`loop_begin`) {#op-loop-begin}

1Byte 命令。

`loop_counter` の値をオペコード下位4bitに、`loop_start_addr` の値を次命令のアドレスにする。  
オペコード `0x50` は 256 回のループとなることに注意。

実行を継続する。

#### `0x51` (`loop_end`) {#op-loop-end}

1Byte 命令。

`loop_counter` をデクリメントし、その結果が 0 なら次命令のアドレスへ、さもなくば `loop_start_addr` へジャンプする。

実行を継続する。

#### `0x60..=0x6F` (`shoot_direction`) {#op-shoot-direction}

1Byte 命令。

ゲーム内では未使用(TODO: 詳細)。

実行を継続する。

#### `0x70..=0x7F` (`set_sprite`) {#op-set-sprite}

1Byte 命令。

敵メタスプライトIDを `(基本値)+(オペコード下位4bit)` にする。

実行を継続する。

#### `0x80..=0x8F` (`set_homing_timer`) {#op-set-homing-timer}

1Byte 命令。

オペコード 0x80 なら `homing_timer` の値を `252` にする。  
それ以外の場合、`homing_timer` の値を `4*(オペコード下位4bit)` にする。

自機追尾処理に戻って実行を継続する。

#### `0x90..=0x93` (`set_inversion`) {#op-set-inversion}

1Byte 命令。

`inv_x` をオペコードの bit0 に、`inv_y` をオペコードの bit1 にする。

実行を継続する。

#### `0xA0` (`set_position`) {#op-set-position}

3Byte 命令。`0xA0 <x> <y>` の形式。

敵座標x を `<x>` に、敵座標y を `<y>` にする。

実行を継続する。

#### `0xA1` (`set_jump_on_damage`) {#op-set-jump-on-damage}

2Byte 命令。`0xA1 <addr>` の形式。

`jump_on_damage` を `<addr>` にする。

実行を**中断**する。

#### `0xA2` (`increment_sprite`) {#op-increment-sprite}

1Byte 命令。

敵メタスプライトIDをインクリメントする。

実行を継続する。

#### `0xA3` (`decrement_sprite`) {#op-decrement-sprite}

1Byte 命令。

敵メタスプライトIDをデクリメントする。

実行を継続する。

#### `0xA4` (`set_part`) {#op-set-part}

2Byte 命令。`0xA4 <part>` の形式。

敵パーツ番号を `<part>` にする。

実行を継続する。

#### `0xA5` (`randomize_x`) {#op-randomize-x}

2Byte 命令。`0xA5 <mask>` の形式。

敵座標x の `<mask>` で指定されたbitたちをランダムに変更する。

実行を継続する。

#### `0xA6` (`randomize_y`) {#op-randomize-y}

2Byte 命令。`0xA6 <mask>` の形式。

敵座標y の `<mask>` で指定されたbitたちをランダムに変更する。

実行を継続する。

#### `0xB0..=0xB3` (`bcc_x`, `bcs_x`, `bcc_y`, `bcs_y`) {#op-branch}

2Byte 命令。`0xB0..=0xB3 <addr>` の形式。

敵座標と自機座標を比較し、条件を満たしていれば `<addr>` へ、さもなくば次命令のアドレスへジャンプする。

実行を継続する。

#### `0xC0..=0xCF` (`shoot_aim`) {#op-shoot-aim}

1Byte 命令。  
オペコード下位4bitは意味を持たない(何らかの拡張予定があったのかも)。

自機狙い弾を撃つことを試みる。  
[弾抑制フラグ](@/enemy-group/index.md#group-noshot)、[弾高速化フラグ](@/enemy-group/index.md#group-accel-shot)、[誘導弾化フラグ](@/enemy-group/index.md#group-homing-shot)の影響を受ける。

TODO: 発射間隔など

#### `0xF0` (`restore_music`) {#op-restore-music}

1Byte 命令。

BGMをデフォルトに戻す。ラザロのBGMを元に戻すのに使われる。  
ただし、現在ゲームオーバーBGMが流れている場合は何もしない。

実行を継続する。

#### `0xF1..=0xFF` (`play_sound`) {#op-play-sound}

1Byte 命令。

オペコード下位4bitの効果音を鳴らす。

実行を継続する。
