# husk-patch

[Husk](https://github.com/Leviidev/Husk) 2.0 に当てる非公式パッチと、パッチ込みの IPA を作る GitHub Actions です。
Husk は GPL-2.0-or-later です。このリポジトリにはパッチ（`patches/`）とワークフローだけを置き、Husk 本体のソースはビルド時に取得します。

## パッチ

| ファイル | 内容 |
| --- | --- |
| `patches/0001-unity-claim-gles31-from-manifest.patch` | Translation Layer のネイティブ Unity 経路が、マニフェストで ES 3.1/3.2 を要求するゲームに「ドライバは対応している」と伝える。Unity が「Your device does not match the hardware requirements of this application.」を出して 0 フレームのまま止まる問題への対処。あわせて Unity 起動時に EGL トレースを有効にする。 |

## IPA の作り方

Actions → **Build patched Husk IPA** → *Run workflow*。

- `mode=fast`（既定）: 公式 Release の `Husk.ipa` から QEMU / ANGLE / MoltenVK / ゲスト一式を借り、アプリ本体（Translation Layer を含む）だけをビルドします。
- `mode=full`: 上流の `scripts/ci_build.sh` で全部をビルドします（数時間）。

出来た IPA は Artifacts と、`patched-…` という Release に付きます。
