# husk-patch

[Husk](https://github.com/Leviidev/Husk) 2.0 に当てる非公式パッチと、パッチ込みの IPA を作る GitHub Actions です。
Husk は GPL-2.0-or-later です。このリポジトリにはパッチ（`patches/`）とワークフローだけを置き、Husk 本体のソースはビルド時に取得します。

## パッチ

| ファイル | 内容 |
| --- | --- |
| `patches/0001-unity-claim-gles31-from-manifest.patch` | Translation Layer のネイティブ Unity 経路が、マニフェストで ES 3.1/3.2 を要求するゲームに「ドライバは対応している」と伝える。Unity が「Your device does not match the hardware requirements of this application.」を出して 0 フレームのまま止まる問題への対処。あわせて Unity 起動時に EGL トレースを有効にする。 |
| `patches/0002-egl-trace-every-call.patch` | 診断用。EGL の全呼び出し（eglGetDisplay / eglInitialize / eglQueryString / eglGetError など）とその結果をログに残す。Unity が EGL のどの段階で諦めているかを特定するため。 |
| `patches/0003-unity-vulkan-only-builds-use-moltenvk.patch` | Vulkan だけでビルドされた Unity ゲーム（GLES 用シェーダーが無い。例: Taiko 0.99.98）に MoltenVK を渡す。マニフェストが `android.hardware.vulkan.version` を必須としているときだけ有効で、GLES でも動くビルドは従来どおり ANGLE を使う。これが無いと Unity は libvulkan.so を見つけられず、「Your device does not match the hardware requirements」を出して 0 フレームで止まる。 |
| `patches/0004-unity-stall-backtrace.patch` | 診断用。Unity のフレーム数が 8 秒（と 20 秒）止まったら、エンジンのスレッドに SIGUSR2 を送り、PC・LR・フレームポインタの連鎖を「ライブラリ名 + オフセット」でログに出す。あわせて Vulkan のトレース（TL_VK_TRACE）も有効にする。 |
| `patches/0005-unity-stall-all-threads.patch` | 診断用。0004 の停止検知で、ゲームが作った全スレッド（スレッド名つき）の停止位置を出す。TL_THREAD_TRACE も有効にして、スレッドが終了していないかも分かるようにする。 |
| `patches/0006-unity-stall-stack-scan.patch` | 診断用。停止ダンプで、フレームポインタの連鎖が途切れる場合に備え、スタックをスキャンして戻りアドレスの候補（#100〜）も出す。 |
| `patches/0007-sync-event-ring.patch` | 診断用。futex の待ち・起こしと条件変数の待ち・通知を、メモリ上のリングバッファに記録し、Unity が止まったときに直近約160件を出す（誰が何を待ち、誰が起こそうとしたか）。 |
| `patches/0008-sync-ring-coalesce.patch` | 診断用。0007 のリングを 4096 件に拡大し、同じスレッド・種類・アドレスの繰り返し（vsync スレッドの 60Hz broadcast など）は 1 行にまとめて、止まる前後の出来事が埋もれないようにする。 |
| `patches/0009-unity-force-gfx-direct.patch` | 実験。Vulkan 専用の Unity ビルドに、起動引数 `-force-gfx-direct`（描画を別スレッドに分けず、メインスレッドで直接行う）を渡す。Intent の extras `unity` を返す `Intent.getExtras` / `Bundle.getString` を実装した。メインスレッドと描画ワーカーの待ち合わせで止まる問題を避けられるかを見る。 |
| `patches/0010-bundle-containskey.patch` | 0009 の続き。Unity は `Bundle.containsKey("unity")` を先に呼ぶので、起動引数があるときは `true` を返す（0009 だけでは引数が読まれていなかった）。 |
| `patches/0011-bundle-getstring-default.patch` | 0010 の続き。Unity は `Bundle.getString(key, default)` の 2 引数版で引数を読むので、そちらも実装した。 |
| `patches/0012-file-trace.patch` | 診断用。Unity の Vulkan ビルドで TL_FILE_TRACE を有効にし、ゲームが open / stat しようとしたファイルと結果（ENOENT など）をログに残す。 |
| `patches/0013-etc-fonts-xml.patch` | `/etc/fonts.xml` と `/etc/font_fallback.xml`（実機では `/etc` が `/system/etc` へのリンク）も、アプリが持つフォント一覧に繋ぐ。Unity のフォント管理は `/etc/fonts.xml` を最初に開く。 |
| `patches/0014-hide-display-timing-and-choreo-notes.patch` | 実験。Vulkan 専用の Unity ビルドには `VK_GOOGLE_display_timing` を見せず（Swappy が別のフレームペーシング実装を使う）、Choreographer のコールバック登録・実行と Looper の wake を同期リングに記録する。 |
| `patches/0015-vk-fence-notes.patch` | 診断用。Vulkan 専用の Unity ビルドで vkQueueSubmit / vkWaitForFences / vkGetFenceStatus / vkResetFences の呼び出しと結果を同期リングに記録する（Swappy のフェンス待ちが完了するかを見る）。 |
| `patches/0016-bypass-swappy-queuepresent.patch` | 修正案。Vulkan 専用の Unity 2020.3.33 で、`SwappyVk_queuePresent`（libunity +0xe0d7c8）の先頭を書き換えて MoltenVK の present を直接呼ぶ。Swappy が結果待ちで永久に止まる問題を避ける（先頭の命令が一致するビルドだけ。`TL_SWAPPY_KEEP` で無効）。 |
| `patches/0017-touchscreen-input-device.patch` | 修正案。タッチ入力が効かない件。Unity が `InputEvent.getDevice()` を呼んで null を受け取っていたので、タッチスクリーンの `InputDevice`（ソース 0x1002）を返す。あわせて最初の数回のタッチの結果をログに出す。 |

## IPA の作り方

Actions → **Build patched Husk IPA** → *Run workflow*。

- `mode=fast`（既定）: 公式 Release の `Husk.ipa` から QEMU / ANGLE / MoltenVK / ゲスト一式を借り、アプリ本体（Translation Layer を含む）だけをビルドします。
- `mode=full`: 上流の `scripts/ci_build.sh` で全部をビルドします（数時間）。

出来た IPA は Artifacts と、`patched-…` という Release に付きます。
