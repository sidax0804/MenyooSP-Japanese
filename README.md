# Menyoo SP - Japanese Translation & ScriptHookVDotNet Compatibility Fix

> **⚠ WIP (Work In Progress):** Translation is not yet complete. Some menu items may still appear in English.
>
> Note: If you are using Menyoo 2.3.0a4 or later, this patch is not needed. The language system crash fix has been merged into the official build, so language files and SHVDN can now coexist without issues. See itsjustcurtis/MenyooSP#478 for details. This patch is intended for users still on versions prior to 2.3.0a4 (e.g. 2.2.2).​​​​​​​​​​​​​​​​

A Japanese language pack and crash fix patch for [Menyoo SP](https://github.com/itsjustcurtis/MenyooSP) (GTA V single-player trainer).

## The Problem

Menyoo 2.2.2's language system crashes when [ScriptHookVDotNet](https://github.com/scripthookvdotnet/scripthookvdotnet) (SHVDN) is also installed. The error looks like this:

```
CORE: An exception occurred while executing 'Menyoo.asi', id 3
Exception addr 0x00007FFEBEFE8BC0 is clr.dll+0x005F8BC0
Last called native 0x0000000000000000
```

This affects **all** language files, not just Japanese.

### Root Cause

In `Language.cpp`, the `Lang::Translate()` method uses `std::map::at()` to look up translation keys. For any key not found in the JSON file, this throws a `std::out_of_range` C++ exception. Normally this is caught and handled gracefully.

However, when SHVDN is loaded, the .NET CLR installs a process-wide **Vectored Exception Handler**. This handler intercepts the C++ exception before Menyoo's `catch` block can handle it. Because ScriptHookV runs scripts inside **fibers** (not regular threads), the CLR's stack space check fails (fiber stack != cached thread stack), causing a fatal crash.

See: [SHVDN Issue #976 - Usage of fibers causes random .NET runtime crashes](https://github.com/scripthookvdotnet/scripthookvdotnet/issues/976)

### The Fix

Replace `std::map::at()` with `std::map::find()` in `Language.cpp`. This eliminates the C++ exception entirely, so the CLR vectored exception handler never fires.

```diff
- try {
-     auto& ret = this->pairs.at(text);
-     return ret;
- }
- catch (std::out_of_range)
- {
-     ...
- }
+ auto it = this->pairs.find(text);
+ if (it != this->pairs.end()) {
+     return it->second;
+ }
+ // fallback: cache and return original text
+ this->pairs[text] = text;
+ return text;
```

## Contents

```
patch/
  Menyoo.asi           - Patched binary (built from MenyooSP 2.2.2 + fix)
  language-fix.patch   - Git diff of the source code change
language/
  Japanese.json        - Japanese translation (5,065 entries)
```

## Installation

1. **Back up** your existing `Menyoo.asi` in your GTA V directory
2. Copy `patch/Menyoo.asi` to your GTA V root directory (overwrite the existing one)
3. Copy `language/Japanese.json` to `menyooStuff/Language/`
4. Edit `menyooStuff/menyooConfig.ini`:
   - Find `language =` under `[settings]`
   - Change it to `language = Japanese`
5. **(Recommended)** Change fonts to support Japanese characters in the same file:
   - Under `[fonts]`, change `options`, `selection`, and `breaks` to `0`

```ini
[settings]
language = Japanese

[fonts]
title = 7
options = 0
selection = 0
breaks = 0
```

6. Launch GTA V

## Building from Source

If you want to build the patched Menyoo.asi yourself:

1. Clone the MenyooSP repository:
   ```
   git clone --recursive https://github.com/itsjustcurtis/MenyooSP.git
   ```
2. Apply the patch:
   ```
   cd MenyooSP
   git apply path/to/language-fix.patch
   ```
3. Generate the Visual Studio solution:
   ```
   Solution\external\premake\premake5.exe vs2022
   ```
4. Build with Visual Studio 2022 or MSBuild:
   ```
   MSBuild Solution\Menyoo.sln -p:Configuration=Release -p:Platform=x64
   ```
5. Output: `Solution\source\_Build\bin\Release\Menyoo.asi`

## Translation Coverage

The Japanese translation covers 5,065 menu strings including:

- All main menu items and toggles
- UI text, error messages, and help text
- Player appearance options (face features, overlays, etc.)
- All weapon names and customization options
- Teleport locations
- Object Spooner features
- Weather, gravity, and visual effects
- Ped model names (animals, citizens, story characters)
- DLC ped models
- Vehicle options (paint, speed, doors, etc.)

Some strings may remain in English if they were added in Menyoo 2.2.2 after the translation was created.

## Credits

- [MAFINS](https://github.com/MAFINS) - Original Menyoo PC creator
- [itsjustcurtis](https://github.com/itsjustcurtis) - Menyoo 2.0+ maintainer
- [lucienlmy](https://github.com/lucienlmy/Menyoo-Translation) - Chinese translation (used as reference for complete key extraction)

## License

This project is licensed under the GNU General Public License v3.0 - see [LICENSE](LICENSE) for details.

Menyoo PC is originally by MAFINS, continued by itsjustcurtis.

---

# Menyoo SP - 日本語翻訳 & ScriptHookVDotNet互換性修正パッチ

> **⚠ 作業中（WIP）：** 翻訳はまだ完全ではありません。一部のメニュー項目が英語のまま表示される場合があります。
> 
> ※ Menyoo 2.3.0a4以降をお使いの方は、このパッチは不要です。言語システムのクラッシュ修正が本体に取り込まれたため、言語ファイルとSHVDNの共存が公式にサポートされています。詳細は itsjustcurtis/MenyooSP#478 をご覧ください。このパッチは Menyoo 2.2.2 など、2.3.0a4より前のバージョンをお使いの方向けです。

[Menyoo SP](https://github.com/itsjustcurtis/MenyooSP)（GTA Vシングルプレイヤー用トレーナー）の日本語言語パックとクラッシュ修正パッチです。

## 問題

Menyoo 2.2.2の言語システムは、[ScriptHookVDotNet](https://github.com/scripthookvdotnet/scripthookvdotnet)（SHVDN）が同時にインストールされている環境でクラッシュします。

```
CORE: An exception occurred while executing 'Menyoo.asi', id 3
Exception addr 0x00007FFEBEFE8BC0 is clr.dll+0x005F8BC0
Last called native 0x0000000000000000
```

これは日本語だけでなく、**全ての言語ファイル**で発生します。

### 原因

`Language.cpp`の`Lang::Translate()`メソッドが`std::map::at()`を使用しており、JSONに存在しないキーに対して`std::out_of_range`例外をスローします。通常はcatchブロックで処理されますが、SHVDNが読み込む.NET CLRのVectored Exception Handlerがこの例外を横取りし、ScriptHookVのfiber上で実行されているためスタック境界の不一致でクラッシュします。

### 修正内容

`std::map::at()` を `std::map::find()` に置き換え、C++例外が一切発生しないようにしました。

## 内容物

```
patch/
  Menyoo.asi           - パッチ適用済みバイナリ（MenyooSP 2.2.2 + 修正）
  language-fix.patch   - ソースコード変更のdiffファイル
language/
  Japanese.json        - 日本語翻訳ファイル（5,065エントリ）
```

## インストール方法

1. GTA Vディレクトリにある既存の`Menyoo.asi`を**バックアップ**
2. `patch/Menyoo.asi`をGTA Vのルートディレクトリにコピー（上書き）
3. `language/Japanese.json`を`menyooStuff/Language/`にコピー
4. `menyooStuff/menyooConfig.ini`を編集:
   - `[settings]`セクションの`language =`を`language = Japanese`に変更
5. **（推奨）** 同ファイルのフォント設定で日本語対応フォントに変更:
   - `[fonts]`セクションの`options`、`selection`、`breaks`を`0`に変更

```ini
[settings]
language = Japanese

[fonts]
title = 7
options = 0
selection = 0
breaks = 0
```

6. GTA Vを起動

## 翻訳カバー範囲

5,065のメニュー文字列を翻訳済み:

- メインメニュー項目、トグル設定
- UIテキスト、エラーメッセージ、ヘルプテキスト
- プレイヤー外見オプション（顔のパーツ、オーバーレイ等）
- 全武器名とカスタマイズオプション
- テレポート先ロケーション
- Object Spoonerの全機能
- 天候、重力、視覚エフェクト設定
- 歩行者モデル名（動物、市民、ストーリーキャラクター）
- DLC歩行者モデル
- 車両オプション（塗装、速度、ドア等）

Menyoo 2.2.2で新規追加された文字列の一部は英語のまま残る場合があります。

## ライセンス

GNU General Public License v3.0 - 詳細は[LICENSE](LICENSE)を参照してください。
