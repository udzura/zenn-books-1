---
title: "Runtimeの実装"
---

前章ではWasmバイナリのデコード処理を実装して、足し算する関数をデコードできるところまで実装した。
本章ではデコードされた関数をどのように実行していくかについて解説する。

## 前提知識

[Wasmの入門](/books/writing-wasm-runtime-in-rust/03_intro_wasm%252Emd)の「スタックマシンの補足」節で少しだけ説明したが、関数を実行するということは関数が持つ命令をループで処理していくということになる。
しかし、命令の処理を実行する上でいくつか前提知識があるので、まずそれらについて解説する。

### プログラムカウンタ

プログラムカウントは次に実行する命令の番地を指す値のこと。
Wasm Runtimeでは命令は配列となっているので、プログラムカウンタは配列のインデックスの値となる。

プログラムカウンタはループを1周するごとにカウントアップされていく。

### コールスタック

関数の情報であるフレーム（後述する）を保持するためのスタック領域のこと。
関数が呼ばれるとフレームがスタックに`push`され、呼び出しが終わると`pop`され、呼び出し元の関数の処理に戻る。
このようにコールスタックを使って実行中の関数の切り替えを行う。

### フレーム

関数の実行に必要な次の情報を持つデータ構造のこと。

- プログラムカウンタ
- スタックポインタ（後述する）
- 命令列
- 引数・ローカル変数
- 戻り値の数

基本的にこれらの情報を使って関数の命令を実行していくが、スタックだけは関数ごとではなく共通である。

### スタックポインタ

フレームは関数ごとに作成されコールスタックに積まれていくが、スタックは関数ごとに作成されず共通で使用する。
共通のため、呼び出し元に戻る際にはスタックに不要な値が溜まっている可能性があるため、スタックも巻き戻す必要がある。
その際にスタックをどこまで巻き戻すかの情報が必要で、それがスタックポイントである。