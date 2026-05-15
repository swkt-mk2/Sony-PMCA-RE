# DSC-T2等への言語アンロック適用可能性メモ

## 調査目的
- `DSC-TX7`向けに見つかった「言語アンロック」実装が、`DSC-T2`など別機種にも適用できそうかを、**コード上の根拠のみ**で確認する。

## 結論（現時点）
- このリポジトリには `DSC-T2` / `DSC-TX7` の機種個別実装は見当たらない。
- 言語アンロックは、機種分岐ではなく**共通のBackupプロパティ書き換え**として実装されているため、
  - 対象機がこのツールのバックアップアクセス経路に乗る
  - かつ言語配列プロパティID/サイズが互換
 であれば、理屈上は適用可能。
- 一方で、`language`は固定で `0x010d008f..+34`（35バイト）を前提にしているため、古い機種で構造が違う場合は失敗または不正書換のリスクがある。

## 根拠（コード）

### 1. 言語アンロックは共通tweakとして実装
- `TweakInterface` に `language` が `Unlock all languages` として登録されている。  
  (`self.addTweak('language', 'Unlock all languages', LanguageTweak(backup))`)

### 2. 実際の書換対象が固定
- `BackupInterface` で `language` は以下の固定アドレス列で定義。  
  `CompoundBackupProp(dataInterface, [(0x010d008f + i, 1) for i in range(35)])`

### 3. アンロックON/OFFの意味
- ON: `ALLLANG` マスク（35個すべて有効）を使う。
- OFF: `backupRegion`由来のリージョン文字列を使って、地域ごとのデフォルトマスクへ戻す。

### 4. バックアップフォーマットの前提
- Backup parser は `BK2/BK4` リビジョンのみ受理（`revision in [2,4]`）。
- つまり、古い/異なる世代の機種でバックアップ形式が異なると、この経路自体が使えない可能性がある。

### 5. 公式README上の互換性注意
- 対応機種は外部一覧への参照で管理され、さらに一部アーキテクチャ（CXD90045/CXD90057）は非対応と明記。
- 個別機種の可否は最終的に実機接続で判定する運用になっている。

## DSC-T2への適用可能性評価
- **可能性あり（条件付き）**:
  - Backupアクセスが動作する
  - 当該機の言語テーブルが35バイト互換
  - `region`文字列が既知キー（`ALLLANG`, `J1`, `CE` など）に整合
- **不確実要素**:
  - 機種固有のプロパティID差分
  - Backupサイズ/属性差分
  - 保護属性の扱い差異（`bk unlock`やpatch経由が必要な場合）

## 実機検証時の最小チェック項目（変更は加えない）
1. `info` で `backupRegion` が読めるか。
2. `tweak` 一覧に `Unlock all languages` が出るか。
3. `language` 読み出し値が35バイトか（利用可能判定を通るか）。
4. ON/OFFトグル後に再読込して `1/2` の配列として整合するか。

## 補足
- 本メモは**実装読み取りのみ**で、`DSC-T2`実機での成功を保証するものではない。


## 追記: 実機エラー（Sense 0x5 0x20 0x0）の解釈

### 観測された事象
- `pmca-gui` の Tweaking 起動時、`SonyExtCmdCamera(device).getCameraInfo()` 呼び出しで失敗。
- 例外: `pmca.usb.InvalidCommandException: Mass storage error: Sense 0x5 0x20 0x0`

### コード上の意味
- `Sense 0x5 0x20 0x0` は `MSC_SENSE_InvalidCommandOperationCode` として定義され、
  「そのMSCデバイスが受け取ったコマンドをサポートしていない」扱いで `InvalidCommandException` を投げる実装。
- 失敗箇所は `updaterShellCommand` 内のモデル取得処理（`getCameraInfo`）で、
  ここでは Sony拡張コマンド（`sendSonyExtCommand`）を送っている。
- したがって今回の失敗は、**言語tweak本体以前**に「Updater shellへ入るための拡張コマンド経路」がT2で通っていないことを示す。

### 判断（T2に使えるか）
- 現状ログだけで判断すると、`pmca-gui` の Tweaking（updater shell経由）は **DSC-T2では非対応の可能性が高い**。
- 少なくとも現行実装の `getCameraInfo` / Sony拡張MSCコマンドが通らないため、
  本リポジトリの通常手順で language tweak を適用するのは難しい。

### ただし「完全に不可能」と断定しない理由
- このエラーは「この経路のコマンドが無効」という事実を示すもので、
  機種固有の別経路（別モード/別プロトコル）まで否定するものではない。
- ただし本リポジトリ内には、DSC-T2向けにその代替経路を実装した痕跡は見当たらない。

### 実務的な結論
- **このツールのTweaking機能に関しては、DSC-T2は“使えない寄り”と見るのが妥当**。
- 対応させるには、T2が受理する通信経路（サービスモード等）の追加解析/実装が別途必要。


## T2対応実装を見据えた準備項目（提案）

### 1) 実機情報の収集基盤を先に作る
- 目的: 「どのコマンドが通る/通らないか」を再現可能にする。
- 準備:
  - USBトランザクションログ取得環境（WindowsならUSBPcap + Wireshark）を整備。
  - PMCA実行ログを保存（コマンド、ドライバ種別、接続モード、例外スタックトレース）。
  - 取得したSenseコードを機種/モード別に表にして蓄積。

### 2) 接続モードの切り分け計画
- 目的: updater shell経路が不可でも、代替経路があるか確認する。
- 準備:
  - MSC / MTP / （可能なら）サービスモードの到達可否を個別に確認する手順書化。
  - `getCameraInfo` に依存しない最小疎通コマンド群を用意（読み取り専用の軽量クエリ）。
  - 「モデル判定失敗時に手動モデル指定で続行」する運用・コード方針を設計。

### 3) 機種別互換レイヤー設計
- 目的: 既存の“固定ID前提”を崩さず、機種差分を吸収する。
- 準備:
  - `language`プロパティの定義（ID範囲、サイズ、値意味）を機種プロファイル化する設計案を作る。
  - 最低限 `auto`（現行）+ `legacy_t_series`（仮）等の分岐キーを設計。
  - 書き込み前に「サイズ一致」「値ドメイン(1/2)」「regionキー整合」を検証するガード仕様を策定。

### 4) 安全性設計（ロールバック前提）
- 目的: 失敗時に復旧できるようにする。
- 準備:
  - 変更前バックアップダンプの自動取得。
  - 書き換え対象IDの差分表示（before/after）と、適用前確認ステップ。
  - 失敗時に元値へ戻すワンコマンド復元フローを設計。

### 5) 実装前の受け入れ条件（DoD）を明確化
- 目的: “動いた気がする”を排除する。
- 最低条件:
  - T2で接続・識別・バックアップ読み出しが安定。
  - language相当領域の読み出しが再現性を持って成功。
  - 1回以上のON/OFF往復で整合（再起動後も値保持）。
  - 既知リスク（非対応モード、失敗時の復元不能）がドキュメント化済み。

### 6) 実装の進め方（小さく段階的に）
- Phase A: 読み取り専用の“診断コマンド”追加（書き込みなし）。
- Phase B: T2プロファイルを導入して、dry-run（書き込み予定値のみ表示）。
- Phase C: 明示フラグ付きで書き込み有効化（`--i-know-what-im-doing`相当）。
- Phase D: GUI導線は最後（CLIで十分検証後）。

### 7) 必要な検証リソース
- 可能ならT2実機を2台（検証機/予備機）。
- バッテリー劣化の少ない電源環境（途中断電回避）。
- 既知の対応機（例: 現行でtweak可能機）を比較対照として同時検証。

## まとめ（準備方針）
- まずは「T2でSony拡張MSCコマンドが通らない」事実を前提に、
  **診断→代替経路探索→機種別プロファイル化→安全な書き込み** の順で進めるのが現実的。
- いきなりlanguage書き込みを実装するより、ログ基盤とロールバック設計を先に作るほうが成功率が高い。


## 追記2: `infoCommand` でも同一エラーを確認

### 新しい観測ログ
- `infoCommand` 実行時にも `SonyExtCmdCamera.getCameraInfo()` で失敗。
- 例外は前回と同じ: `InvalidCommandException: Mass storage error: Sense 0x5 0x20 0x0`。

### 意味
- `Tweaking` だけでなく、`info` 系の初期情報取得でも同じ失敗が再現している。
- つまり問題はtweak固有ではなく、**T2に対する Sony拡張MSCコマンド経路そのもの**が通っていない可能性がより高い。

### コード上の整合
- `infoCommand` でも `dev.getCameraInfo()` が直接呼ばれる。
- `getCameraInfo()` は `_sendCommand()` を通じて `sendSonyExtCommand()` を使う。
- `MscDevice._checkResponse()` で `0x5/0x20/0x0` は `InvalidCommandException` として処理される。

### 実務判断の更新
- これで「Tweaking時だけの一時不具合」の可能性は下がった。
- 現行PMCAのMSC拡張コマンド路線では、`DSC-T2` は非対応とみなすのが妥当。
- 次の実作業は、MTP/サービスモード等の**別経路探索**を優先する。


## 追記3: PictBridge接続時の観測（MTP列挙失敗）

### 観測ログ
- カメラをPictBridgeモードで接続して試行。
- PMCAログ:
  - `Using drivers Windows-MSC, Windows-MTP`
  - `Querying MTP device`
  - `No devices found. Ensure your camera is connected.`

### 解釈
- この試行では、PMCAがMTPデバイスとしてT2を列挙できていない。
- したがって、少なくとも現環境/現手順では「MSC拡張経路が失敗」した際の代替として
  MTP経路へ逃がすこともできていない。

### 実務上の意味
- 現時点で確認できた経路は以下。
  1. **MSC経路**: `getCameraInfo` で `Sense 0x5 0x20 0x0`（未対応コマンド）
  2. **PictBridge(MTP想定)経路**: PMCA側でデバイス未検出
- このため、現行PMCAの標準導線ではT2に対して有効な制御チャネルが未確立。

### 次に優先する確認
- Windowsデバイスマネージャ上で、PictBridge接続時にMTP/PTPデバイスとして認識されているか確認。
- 別USBポート/別PCでも同様にMTP列挙可否を確認。
- 可能ならサービスモード経路の検証へ進む（別ドライバ前提）。

## 追記4: 接続ログ取得に使うソフトとコマンド例

### 推奨ソフト（Windows）
1. **USBPcap**
   - USBトラフィックをキャプチャするためのドライバ。
2. **Wireshark**
   - USBPcapで取得した`pcapng`を解析するため。
3. **Zadig**（サービスモード検証時のみ）
   - libusb系ドライバ差し替えが必要な場合に使用。
4. **Device Manager（標準）**
   - MTP/PTPとして認識されているかを確認。

### まず取るべきログ（安全・非破壊）
- A. PMCA実行ログ（標準出力/標準エラー）
- B. USBPcapログ（失敗再現の前後30秒程度）
- C. Windowsデバイス認識情報（デバイスマネージャのクラス/VID/PID）

### コマンド例（PowerShell）

#### 1) PMCAログをファイル保存
```powershell
# 先にログディレクトリを作成（既に存在していれば何もしない）
New-Item -ItemType Directory -Path .\logs -Force | Out-Null

# 例: info 実行ログ
python .\pmca-console.py info *>&1 | Tee-Object -FilePath .\logs\t2_info_$(Get-Date -Format yyyyMMdd_HHmmss).log

# 例: updatershell 実行ログ
python .\pmca-console.py updatershell *>&1 | Tee-Object -FilePath .\logs\t2_updatershell_$(Get-Date -Format yyyyMMdd_HHmmss).log
```

#### 2) PnPデバイス情報の取得（MTP/PTP確認）
```powershell
# 接続中デバイスから Sony を抽出
Get-PnpDevice | Where-Object { $_.FriendlyName -match 'Sony|DSC|Camera|MTP|PTP' } |
  Format-Table -AutoSize Status, Class, FriendlyName, InstanceId

# USBデバイス詳細（VID/PID確認）
Get-PnpDevice -PresentOnly | Where-Object { $_.InstanceId -match '^USB' -and $_.FriendlyName -match 'Sony|DSC|Camera' } |
  Format-Table -AutoSize Status, Class, FriendlyName, InstanceId
```

#### 3) USBPcap + Wiresharkでの取得手順（要GUI操作）
1. USBPcapをインストール（Wireshark同梱/別導入）。
2. Wireshark起動 → `USBPcapX` インターフェースを選択。
3. キャプチャ開始後、PMCAコマンドを1回だけ実行して失敗を再現。
4. キャプチャ停止し、`t2_msc_fail_YYYYMMDD_HHMMSS.pcapng` として保存。
5. Wiresharkフィルタ例:
   - `usb.capdata contains 5a:00`（状況により調整）
   - `scsi` / `usb.mass_storage` で絞り込み、Sense返却フレームを確認。

### ログ採取時の注意
- 1回のキャプチャは短く（30〜90秒）し、再現1ケースごとに保存。
- ケーブル/ポート/モードを変えたら、ファイル名に条件を含める。
- 書き込み系コマンド（tweak適用、backup patch）はまだ実行しない。

### 共有してほしい最低セット
- `t2_info_*.log`
- `t2_updatershell_*.log`
- `t2_msc_fail_*.pcapng`（可能なら）
- PnP一覧のテキスト出力



### 補足: `DirectoryNotFoundException` が出る場合
- 原因: `Tee-Object -FilePath .\logs\...` の保存先フォルダ `logs` が未作成。
- 対処: 上記の `New-Item -ItemType Directory -Path .\logs -Force` を先に1回実行。
- ワンライナー例（フォルダ作成 + 実行）:
```powershell
New-Item -ItemType Directory -Path .\logs -Force | Out-Null; python .\pmca-console.py info *>&1 | Tee-Object -FilePath .\logs\t2_info_$(Get-Date -Format yyyyMMdd_HHmmss).log
```


## 追記5: `Cryptodome` / `Crypto` モジュール不足エラーへの対処

### 症状
- `python .\pmca-console.py info` 実行時に以下で失敗する:
  - `ModuleNotFoundError: No module named 'Cryptodome'`
  - `ModuleNotFoundError: No module named 'Crypto'`

### 原因
- PMCA実行に必要な暗号ライブラリ（PyCryptodome系）が未インストール。
- `pmca-console.py` 起動時に `pmca.xpd` が `Cryptodome` / `Crypto` をimportするため、
  接続テスト以前にプロセスが終了する。

### 推奨セットアップ（Windows / PowerShell）
```powershell
# 1) リポジトリ直下で仮想環境を作成
python -m venv .venv

# 2) 仮想環境を有効化
.\.venv\Scripts\Activate.ps1

# 3) pip を更新
python -m pip install --upgrade pip

# 4) 依存関係を導入（まず requirements があればそれを使用）
if (Test-Path .\requirements.txt) {
  pip install -r .\requirements.txt
} else {
  # 最低限必要になりやすいパッケージ
  pip install pycryptodome
}

# 5) 追加で pycryptodomex も必要なら導入（環境差分対策）
pip install pycryptodomex
```

### 動作確認コマンド
```powershell
# どちらか片方が成功すればOK（pmcaは Cryptodome を優先し、失敗時のみ Crypto にフォールバック）
python -c "from Cryptodome.Hash import HMAC, SHA256; print('Cryptodome OK')"
# 上が成功した場合、下は失敗しても問題ありません
python -c "from Crypto.Hash import HMAC, SHA256; print('Crypto OK (optional)')"
python .\pmca-console.py info
```

### ログ採取コマンド（依存解決後）
```powershell
New-Item -ItemType Directory -Path .\logs -Force | Out-Null
python .\pmca-console.py info *>&1 | Tee-Object -FilePath .\logs\t2_info_$(Get-Date -Format yyyyMMdd_HHmmss).log
python .\pmca-console.py updatershell *>&1 | Tee-Object -FilePath .\logs\t2_updatershell_$(Get-Date -Format yyyyMMdd_HHmmss).log
```

### それでも失敗する場合
- 複数Pythonが混在している可能性があるため、以下で実行バイナリを固定する:
```powershell
.\.venv\Scripts\python.exe .\pmca-console.py info
```


## 追記6: `usb.core.NoBackendError: No backend available` への対処

### 症状
- `info` / `updatershell` 実行時に `libusb-*` ドライバ列挙フェーズで停止し、
  `usb.core.NoBackendError: No backend available` が発生する。

### 原因
- `pyusb` は見つかっているが、Windows上で使うlibusbバックエンドDLL（libusb-1.0等）が見つからない。
- PMCAは複数ドライバ（Windows系 + libusb系）を順に列挙するため、環境によってはlibusb側例外で全体が中断する。

### まずの回避（推奨）
- **Windowsドライバを明示指定**して実行し、libusb列挙を回避する。

```powershell
# info（Windows MSCのみ使用）
python .\pmca-console.py info -d native *>&1 | Tee-Object -FilePath .\logs\t2_info_native_$(Get-Date -Format yyyyMMdd_HHmmss).log

# updatershell（Windows MSCのみ使用）
python .\pmca-console.py updatershell -d native *>&1 | Tee-Object -FilePath .\logs\t2_updatershell_native_$(Get-Date -Format yyyyMMdd_HHmmss).log
```

> 注: この版のPMCAではドライバ指定は各サブコマンドの `-d` で行う。
> 例: `info -d native` / `updatershell -d native`（選択肢は `native`, `libusb`, `qemu`）。

### libusb経路も使いたい場合（任意）
- Zadig等で適切なlibusbドライバを導入し、`libusb-1.0.dll` が参照可能な状態を作る。
- ただし現段階（T2の非破壊調査）では、まずWindowsドライバ固定で十分。

### 実務判断
- 今回の `NoBackendError` は「T2非対応の本質原因」ではなく、**PC側のlibusb実行環境不足**。
- まず `--driver windows` で再実行して、以前の本質エラー（Sense `0x5 0x20 0x0`）再現有無を確認する。
