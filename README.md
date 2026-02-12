# +Message OTP 自動取得システム

Android の +メッセージ通知から認証コード(OTP)を自動抽出し、iAEON ログインを無人化する仕組み。

## 全体構成

```
+Message通知 → Android App (通知リスナー)
                   ↓ OTP抽出
             HTTP Server (:8765)
                   ↓ Wi-Fi LAN
             Python (ポーリング)
                   ↓
             iaeon_auth.full_login(otp_provider=...)
                   ↓
             .env にトークン保存
```

## セットアップ

### 1. ビルド環境の準備

#### Java 17 (SDKMAN)

Gradle ビルドには Java 17 が必要。SDKMAN 経由でインストール:

```bash
# SDKMAN インストール (未導入の場合)
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# Java 17 インストール
sdk install java 17.0.10-tem
```

#### Android SDK (コマンドラインツール)

Android Studio 不要。コマンドラインツールのみでビルドできる:

```bash
# コマンドラインツールをダウンロード・配置
cd /tmp
curl -sO https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip
unzip -qo commandlinetools-linux-11076708_latest.zip -d cmdline-tools-tmp
mkdir -p ~/Android/Sdk/cmdline-tools
mv cmdline-tools-tmp/cmdline-tools ~/Android/Sdk/cmdline-tools/latest
rm -rf commandlinetools-linux-11076708_latest.zip cmdline-tools-tmp

# SDK コンポーネントをインストール
export JAVA_HOME=$HOME/.sdkman/candidates/java/17.0.10-tem
printf 'y\ny\ny\ny\ny\ny\ny\ny\ny\n' | \
  ~/Android/Sdk/cmdline-tools/latest/bin/sdkmanager \
    --sdk_root=~/Android/Sdk \
    --licenses
~/Android/Sdk/cmdline-tools/latest/bin/sdkmanager \
  --sdk_root=~/Android/Sdk \
  "platform-tools" "platforms;android-35" "build-tools;35.0.0"
```

#### Gradle Wrapper

```bash
# Gradle 本体 (Wrapper 生成用、初回のみ)
sdk install gradle 8.11.1

cd android
gradle wrapper --gradle-version 8.11.1
```

### 2. Android アプリのビルド

```bash
cd android

# local.properties に SDK パスを設定
echo "sdk.dir=$HOME/Android/Sdk" > local.properties

# ビルド (初回は依存ダウンロードで数分かかる)
export JAVA_HOME=$HOME/.sdkman/candidates/java/17.0.10-tem
./gradlew assembleDebug
```

APK は `app/build/outputs/apk/debug/app-debug.apk` に出力される。

### 3. Android アプリのインストールと初期設定

```bash
adb install app/build/outputs/apk/debug/app-debug.apk
```

#### 通知アクセスの許可

Android 13 以降はサイドロードしたアプリの通知リスナー権限がブロックされる。先に制限を解除する必要がある:

**方法A: adb コマンド (推奨)**
```bash
adb shell appops set com.example.plusmessageotp ACCESS_RESTRICTED_SETTINGS allow
```

**方法B: 設定画面から**
1. **設定 → アプリ → OTP Relay** を開く
2. 右上の **⋮ (三点メニュー)** をタップ
3. **「制限付き設定を許可」** をタップ

制限解除後:
1. アプリを起動
2. 「通知アクセスを許可」ボタンを押して設定画面へ遷移
3. **OTP Relay** を有効にする
4. 画面に表示される IP アドレスを控える（例: `192.168.100.56:8765`）

#### 動作確認

PC から疎通確認:

```bash
curl http://<phone-ip>:8765/health
# {"status":"ok"}
```

+メッセージでテスト送信後:

```bash
curl http://<phone-ip>:8765/otp
# {"otp":"123456"}
```

### 4. Python パッケージ

```bash
cd /path/to/+message
pip install -e .
```

依存は `requests` と `python-dotenv`。

### 5. 環境変数

**`+message/.env`** - Android 端末の接続情報:

```env
ANDROID_HOST=192.168.100.56
ANDROID_PORT=8765
```

**`aeon/.env`** - iAEON 認証情報とトークン:

```env
PHONE_NUMBER=09012345678
PASSWORD=your_password
DEVICE_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
ACCESS_TOKEN=（auto_login.py が自動書き込み）
```

- `PHONE_NUMBER` と `PASSWORD` は必須。手動で追記すること。
- `DEVICE_ID` は初回ログイン時に自動生成・保存される。同じ値を使い続けると SMS 認証をスキップできる場合がある。
- `ACCESS_TOKEN` は `auto_login.py` 実行時に自動更新される。

## 使い方

### 自動ログイン（メイン用途）

```bash
python auto_login.py
```

処理フロー:
1. Android アプリへの疎通確認
2. 古い OTP をクリア
3. iAEON ログイン開始（SMS 送信）
4. +メッセージに届いた OTP を自動取得（最大120秒待機）
5. 認証完了、トークンを `aeon/.env` の `ACCESS_TOKEN` に保存

### cron で定期実行

トークンは約10時間有効。毎朝4時に更新する例:

```cron
0 4 * * * /path/to/python /path/to/+message/auto_login.py >> /path/to/cron.log 2>&1
```

### Python から直接使う

```python
from plusmsg_otp import PlusMessageOTP

client = PlusMessageOTP(host="192.168.100.56")

# 疎通確認
client.health()  # True

# OTP を確認（消費しない）
client.peek()  # "123456" or None

# OTP を取得して消費（1回限り）
client.consume()  # "123456" or None

# OTP が届くまで待機
code = client.wait_for_otp(timeout=120, poll_interval=2.0)
```

### デバッグ用ウォッチャー

OTP の到着をリアルタイム監視:

```bash
python -m plusmsg_otp.watcher --host 192.168.100.56
# Connected. Watching for OTP...
# [04:00:15] OTP detected: 123456
```

### iaeon_auth.py を直接使う場合

```python
from iaeon_auth import IAEONAuth
from plusmsg_otp import PlusMessageOTP

client = PlusMessageOTP(host="192.168.100.56")
client.clear()

auth = IAEONAuth(device_id="your-device-id")
token = auth.full_login(
    phone_number="09012345678",
    password="your_password",
    otp_provider=lambda: client.wait_for_otp(),
)
# otp_provider を省略すると従来通り input() で手動入力
```

## HTTP API リファレンス

| メソッド | パス | 説明 | レスポンス例 |
|---------|------|------|-------------|
| GET | `/health` | 疎通確認 | `{"status":"ok"}` |
| GET | `/otp` | OTP確認（非消費） | `{"otp":"123456"}` / 204 |
| POST | `/otp/consume` | OTP取得＆消費 | `{"otp":"123456"}` / 204 |
| POST | `/otp/clear` | OTPクリア | `{"status":"cleared"}` |

OTP 未到着時は HTTP 204 (No Content) を返す。OTP は保存から5分で自動失効する。

## 対応キャリア

| キャリア | +Message パッケージ名 |
|---------|---------------------|
| DoCoMo | `com.nttdocomo.android.msg` |
| au | `com.kddi.android.cmail` |
| SoftBank | `jp.softbank.mb.plusmessage` |

## トラブルシューティング

**「Restricted setting」と表示されて通知アクセスを許可できない**
- Android 13+ のセキュリティ制限。上記「通知アクセスの許可」の手順で制限を解除する

**Android アプリに接続できない**
- スマホとPCが同じ Wi-Fi に接続されているか確認
- スマホのファイアウォールやバッテリー最適化でアプリが停止されていないか確認
- アプリを開いて IP アドレスが正しいか確認

**`KeyError: 'PHONE_NUMBER'`**
- `aeon/.env` に `PHONE_NUMBER=09...` と `PASSWORD=...` を追記する

**OTP が検出されない**
- 通知アクセスが許可されているか確認（アプリ画面でステータス表示）
- +メッセージの通知が有効か確認（Android の通知設定）
- 認証コードが6桁の数字であることを確認

**タイムアウトする**
- `wait_for_otp()` のデフォルトは120秒。SMS の遅延がある場合は `timeout` を伸ばす
- `poll_interval` を短くすると検出が速くなるがリクエスト数が増える

**ビルドエラー: `SDK location not found`**
- `android/local.properties` に `sdk.dir=/home/<user>/Android/Sdk` を記述する

**ビルドエラー: Java バージョン**
- `sdk use java 17.0.10-tem` で Java 17 に切り替える
