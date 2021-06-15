+++
title = "敵弾"
date = 2021-06-15
+++

敵弾には自機狙い弾と方向指定弾がある。両者には以下の共通点がある:

* 自機が通常状態でない(例えば、死亡中やワープ中である)場合、敵弾は発射されない。
* 敵弾ランクにより敵弾の最大数と発射間隔が決まる。

## 敵弾ランク {#rank}

敵弾の最大数および発射間隔は敵弾ランクによって決まる([敵編隊ランク](@/enemy-group/index.md#group-rank)とは異なる)。  

敵弾ランクは周回数、面、自機パワーに依存し、次式で計算される:

`(敵弾ランク) = min(28, 2*(自機パワー) + (面) + ((2周目) ? 12 : 0))`

敵弾の最大数は `(敵弾ランク)/4 + 1` となる(切り捨て除算。値の範囲は `1..=8`)。

敵弾の発射間隔は `-((敵弾ランク)+2)/4 + 7` となる(切り捨て除算。値の範囲は `0..=7`)。  
敵が弾を撃った後、この値と同じ回数だけ敵弾の発射がキャンセルされる。

## 自機狙い弾 {#aim}

[敵編隊パラメータ](@/enemy-group/index.md)により計算されたスピード、誘導弾フラグがパラメータとして渡される。

自機狙い弾の方向は[近似計算](@/direction/index.md#aim)され、さらにパラメータで渡されたスピードが OR される。

自機狙い弾が誘導弾になることがある。この判定は以下の順に行われる:

* 自機パワーが 0 の場合、誘導弾にならない。
* パラメータで誘導弾フラグが渡された場合、必ず誘導弾になる。
* スターブレインを逃がしてから死なずに再戦し、逃走カウントが 0 になるまで待つとスターブレインのコアの自機狙い弾が必ず誘導弾になる。
* 2周目のとき、1/2 の確率で誘導弾になる。
* 1周目の13面以降で、かつ自機にバリアが付いているとき、9/16 の確率で誘導弾になる。

## 方向指定弾 {#direction}

方向指定弾を撃つのはビッグスターブレインの砲台のみ。  
[敵AIバイトコード](@/enemy-ai/index.md)にも方向指定弾命令はあるが、ゲーム内では使われていない。

方向指定弾は[敵編隊パラメータ](@/enemy-group/index.md)の影響を受けない。  
また、プログラム中には自機パワーによって方向指定弾を高速化する意図と思われるコードがあるが、実際には機能していない(`((自機パワー)<<2) & 0x30` を方向に OR しているが、これではシフト量が足りない)。

方向指定弾が誘導弾になることがある。この判定は以下の順に行われる:

* 自機パワーが 0 の場合、誘導弾にならない。
* ビッグスターブレインを逃がしてから死なずに再戦し、逃走カウントが 0 になるまで待つと 1/4 の確率でビッグスターブレインの砲台の方向指定弾が誘導弾になる。
* 2周目のとき、1/4 の確率で誘導弾になる。
* 1周目の13面以降で、かつ自機にバリアが付いているとき、1/4 の確率で誘導弾になる。