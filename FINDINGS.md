# Taiko 0.99.98 (Unity 2020.3.33f1 / IL2CPP) が Husk 2.0 の Translation Layer で動かない件: 調査メモ

- 対象: `taiko_0.99.98.apk`（`com.Shanzhai.TaikoUnity`）, Husk 2.0 (build 109, `e02ac68`), iPhone12,1 / iOS 18.3.2
- 実機の Android では動く（= ゲーム側の問題ではない）
- 最初の症状: `Started` のまま 0 フレーム。ログは `DIALOG "Warning": Your device does not match the hardware requirements of this application.`

## 1. 原因が分かったもの（直した）

**このビルドは Vulkan 専用。** 根拠:

- ゲームデータ内の Shader 8 本すべてが `platforms=[18]`（= Vulkan）。GLES 用のバリアントが無い。
- マニフェストに `uses-feature android.hardware.vulkan.version`（`required` 指定なし）。`glEsVersion` は 0x20000（GLES2）。
- `libunity.so` の `PlayerInitEngineGraphics`（`+0x53e3f4`）→ EGL 初期化直後の関数（`+0x3f52xx`）が、許可された API のリスト（GLES2=8 / GLES3=11）から EGL の候補を作る。リストに GLES が無いので候補が空になり、`Your device does not match the hardware requirements` を出す（ログに `eglChooseConfig` が一度も出ないのはこのため）。
- Husk の native Unity 経路（`husk-tl-unity.c`）は ANGLE（GLES）しか渡しておらず、MoltenVK は Unreal / SDL 経路にしか渡していなかった。

→ `patches/0003`: マニフェストが Vulkan を必須としている Unity ビルドにだけ MoltenVK を渡す（`tl_manifest_vulkan_required`）。これで MoltenVK の `VkInstance` / `VkDevice` / スワップチェーンまで作れ、フレームが 3〜4 描画される。

外れていた仮説（参考）: マニフェストの ES 3.1 要求 / OBB の欠落 / ゲーム用ファイルの不足（`AssetBundles` 等を置いても結果は同じ）。

## 2. 今も止まっているところ

3〜4 フレーム描画したあと、`nativeRender` が戻らなくなる（`unity: alive: 3 frames (+0)` が続く）。

### 観察したこと

- 停止時、ゲームの全スレッドが `pthread_cond_wait` / futex 待ち。動いているスレッドは無い。Vulkan のエラー・警告は無い。
- メインスレッド（`UnityMain`）の呼び出し履歴（フレームポインタが途切れるため、スタックを走査して復元）:
  - 最上位は `libunity.so +0x53e9e0〜+0x53ea24` 付近 = 文字列 `level0` / `sharedassets0.assets` / `First scene is required to be loaded asynchronously, but no AsyncOperation was returned.` を参照する関数。つまり**最初のシーン（level0）の読み込み中**。
  - 待ちの本体は `+0xe08484`（`pthread_cond_wait` の呼び出し元）、その呼び出し元に `+0xe0d7f8`（`SwappyVk_queuePresent` などの文字列の近く）、`+0xe053b0` / `+0xe05970` / `+0xe0d3ac`。
- `Filter0` / `Filter1` / `SwappyChoreographer` という名前のスレッドがあり、これは **Swappy（Android Games SDK のフレームペーシング）**のスレッド名と一致する（`SwappyVk initialized ... using VK_GOOGLE_display_timing` もログに出る）。
- 停止直前の同期イベント（Husk 側でリングバッファに記録）:
  - `UnityMain  cond-broadcast 0x3b02379c8` → その約 22ms 後に名前なしのスレッドが同じ条件変数で `cond-wait`。
  - `UnityMain  cond-wait 0x3d0365cc0`（以後、起こされない）。
  - `Filter0/Filter1` が約 16ms ごとに `cond-broadcast 0x3d082c160` を交互に出していたが、停止の少し前に両方が `cond-wait 0x3d080a870` に入り、`SwappyChoreographer` の `cond-broadcast 0x3d080a870` も止まる。
- 描画ワーカー（`UnityGfxDeviceWorker`）有り（既定）でも、無し（`-force-gfx-direct`、`patches/0009〜0011`）でも、**同じ条件変数 `0x3d0365cc0`** で待って止まる。= 描画スレッドの有無は無関係。
- ファイルアクセスも原因ではない: `config.json` / `record.json` / `translation_*.json` / `Songs/…` は見つかっている。`/etc/fonts.xml` の不一致は直した（`patches/0013`）が、停止は変わらない。

### 現在の見立て（未確定）

Swappy の frame pacing（Choreographer ↔ Filter スレッド ↔ present を呼ぶスレッド）の間の受け渡しで、起こす合図（`AChoreographer` のフレームコールバック、または present 完了）が途切れ、`present` 側が永久に待つ。

- Husk の `AChoreographer_*`（`husk-tl-bionic-ndk.c`）は Flutter 向けに実装済みで、`SwappyChoreographer` スレッドは 60Hz で動いていた。停止直前にそれが止まる理由を調べる必要がある。
- `SwappyDisplayManager`（Java 側）のメソッドは全部 `UNIMPLEMENTED (returning zero)`。
- `VK_GOOGLE_display_timing` の present は「要求時刻を無視して即表示」（`present #1 asked for time ... shown at once`）。Swappy は `vkGetPastPresentationTimingGOOGLE` の結果を使って待ち時間を決めるため、その返り値を確認したい。

### 次に試すとよいこと（作者向け）

1. Swappy を避ける: Unity の Vulkan 初期化で `SwappyVk_initAndGetRefreshCycleDuration` が失敗する条件を作る（拡張を隠すだけでは `SwappyVkFallback` が Choreographer を使うので不十分な可能性がある）。
2. `vkGetPastPresentationTimingGOOGLE` / `vkGetRefreshCycleDurationGOOGLE` の呼び出しと返り値を記録する。
3. `AChoreographer_postFrameCallback*` の呼び出しとコールバックの発火を記録し、停止直前に何が最後に post されたかを見る。

## 3. この調査のために入れた診断

`patches/0002, 0004〜0008, 0012`: EGL 全呼び出しの記録 / フレームが 8 秒止まったときに全スレッドの停止位置（`ライブラリ+オフセット`）と同期イベントのリングバッファを出す / ファイルアクセスの記録。いずれも Unity の Vulkan 専用ビルドでだけ有効。

## 4. ライブラリ側のオフセットの調べ方

ログの `libunity.so +0x…` は APK 内 `lib/arm64-v8a/libunity.so` のファイルオフセット（= 仮想アドレス）。`capstone` / `pyelftools` で逆アセンブルし、`adrp+add` が指す `.rodata` の文字列を近くから拾うと、関数の役割が分かる。
