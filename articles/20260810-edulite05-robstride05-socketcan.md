---
title: "EduLite 05 / RobStride 05をSocketCANで動かしてみた"
emoji: "🦾"
type: "tech"
topics: ["robotics", "motor", "can", "socketcan", "python"]
published: false
---

# はじめに

今回は、小型QDD系アクチュエータである **EduLite 05 / RobStride 05** を、Linux の **SocketCAN** 経由で動かしてみた記録です。

前回の記事では、歩行ロボットで使われる QDD アクチュエータについて整理しました。

- [歩行ロボットとQDDアクチュエータ入門](https://zenn.dev/kaz1110/articles/20260720-qdd-actuator-for-legged-robots)

QDD アクチュエータは、低減速比によるバックドライブ性や力制御との相性の良さから、歩行ロボットやヒューマノイドでよく使われています。

今回はその流れで、実際に手元の EduLite 05 / RobStride 05 を CAN 通信で認識し、状態取得や簡単な制御ができるところまで確認しました。

この記事は、厳密なプロトコル解説というより、**「まず実機を安全に認識して、CLIから触れるようにする」** ところまでの動作メモです。

# 注意

実機モータは、意図せず回転すると危険です。

初回確認時は、以下のような安全対策を強く推奨します。

- モータをしっかり固定する
- 手やケーブルを回転部に近づけない
- 無負荷、または軽負荷で確認する
- 電流・トルク・速度・位置の上限を小さくする
- まずは状態取得やIDスキャンなど、動かないコマンドから確認する
- 異常時にすぐ電源を切れるようにする

今回の確認でも、最初は `scan id` やパラメータ読み出しなど、モータを動かさない操作から始めています。

# 今回やること

今回の記事で扱う内容は以下です。

1. EduLite 05 / RobStride 05 を SocketCAN で接続する
2. Python 製の対話CLIを起動する
3. CAN接続を確認する
4. モータIDをスキャンする
5. モータ状態を読む
6. 制御モードを設定して有効化する
7. 今後の二輪倒立振子への展開を考える

# 使用したもの

今回使ったものは以下です。

- EduLite 05 / RobStride 05
- Linux PC
- SocketCAN 対応CANインターフェース
- Python 3
- 自作 Python SDK / CLI
- 24V 系電源

CAN インターフェースは Linux 側で `can0` として認識させています。

# リポジトリ構成

今回の確認用に、Python SDK と対話CLIを用意しました。

構成イメージは以下です。

```text
edulite05_qdd/
├─ el_a3_sdk/
│  ├─ __init__.py
│  ├─ can_driver.py
│  ├─ data_types.py
│  ├─ protocol.py
│  └─ utils.py
└─ examples/
   └─ interactive_motor_cli.py
```

`el_a3_sdk` 側に CAN 通信、プロトコル、データ型などをまとめ、`examples/interactive_motor_cli.py` から対話的にモータを操作できるようにしています。

# Pythonコードの確認

まず、Python の構文チェックと import 確認を行いました。

```bash
python3 -m compileall el_a3_sdk examples
python3 -c "from el_a3_sdk import RobstrideCanDriver, CommType, MotorType, ParamIndex, RunMode; print('import OK')"
```

実行結果は以下です。

```text
Listing 'el_a3_sdk'...
Compiling 'el_a3_sdk/__init__.py'...
Compiling 'el_a3_sdk/can_driver.py'...
Compiling 'el_a3_sdk/data_types.py'...
Compiling 'el_a3_sdk/protocol.py'...
Compiling 'el_a3_sdk/utils.py'...
Listing 'examples'...
Compiling 'examples/interactive_motor_cli.py'...
import OK
```

ここで import が通れば、ひとまず Python パッケージとしては読み込めています。

# CLIを起動する

次に、対話CLIを起動します。

```bash
python3 examples/interactive_motor_cli.py
```

起動すると、以下のような表示が出ました。

```text
CAN connected
CAN_NAME         = can0
HOST_CAN_ID      = 0xFD
target MOTOR_ID  = 0x7F (127)
MOTOR_TYPE       = 1
Kt estimate      = 0.940000 Nm/A
NOTE: zero is software zero only. Motor internal zero is not modified.
NOTE: use 'scan id' to find connected motor IDs.
NOTE: use 'use id <motor_id>' to change runtime target motor ID.

EduLite 05 / RobStride 05 interactive CLI
Type 'help' or '?' to list commands.
Recommended first commands: get all -> disable -> mode motion_control -> enable
```

`CAN connected` と表示されているので、少なくとも SocketCAN の `can0` には接続できています。

ここで表示されている主な情報は以下です。

| 項目 | 内容 |
|---|---|
| `CAN_NAME` | 使用するCANインターフェース |
| `HOST_CAN_ID` | PC側のCAN ID |
| `target MOTOR_ID` | 操作対象のモータID |
| `MOTOR_TYPE` | モータ種別 |
| `Kt estimate` | トルク定数の推定値 |

起動時点では `target MOTOR_ID = 0x7F` になっています。  
これは探索用・初期設定用のIDとして扱い、実際に接続されているモータIDは次の `scan id` で確認します。

# モータIDをスキャンする

まずはモータを動かさず、接続されているIDを探します。

```text
edulite05> scan id
```

実行結果は以下です。

```text
Scanning motor IDs by read_parameter(VBUS)...
range   = 0x01-0x7F
timeout = 0.050 sec/id
NOTE: This command sends read-only parameter requests. It does not move motors.
found motor: motor_id=0x01 (1), vbus=24.352 V, run_mode=0 (motion_control)
---- scan result ----
- motor_id=0x01 (1)
```

`motor_id=0x01` のモータが見つかりました。

また、`vbus=24.352 V` と出ているので、モータ側でバス電圧も読めています。

ここで重要なのは、`scan id` が **read-only parameter request** であることです。  
つまり、モータを動かさずにIDや状態を確認できます。

初回接続では、こういう「動かない確認」から始めるのが安全です。

# 操作対象IDを変更する

スキャンで見つかったIDが `0x01` だったので、CLIの操作対象を変更します。

```text
edulite05> use id 1
```

これで、以降のコマンドは `motor_id=0x01` のモータを対象にします。

# 状態を読む

次に、モータの状態を読みます。

```text
edulite05> get all
```

ここでは、角度、速度、電流、電圧、温度、制御モードなどを確認します。

初回確認時は、いきなり `enable` して動かすのではなく、まず現在状態を読むのがよいです。

見るべきポイントは以下です。

- バス電圧が想定範囲か
- 温度が異常でないか
- エラー状態が出ていないか
- 制御モードが意図したものか
- 角度や速度が不自然な値になっていないか

# 制御モードを設定する

今回のCLIでは、制御モードを切り替えられるようにしています。

起動時のおすすめコマンドにも出ている通り、まずは以下の流れで確認します。

```text
get all
disable
mode motion_control
enable
```

`disable` で一度モータを無効化し、`mode motion_control` で制御モードを設定してから、`enable` します。

```text
edulite05> disable
edulite05> mode motion_control
edulite05> enable
```

このあたりは、モータを不用意に動かさないためにも、毎回手順を固定しておくと安心です。

# 実際に動かす

制御モード設定と enable ができたら、低い指令値から動作確認します。

たとえば位置制御や速度制御を行う場合でも、最初は必ず小さい値から試します。

例：

```text
edulite05> pos 0.1
```

または速度制御であれば、

```text
edulite05> vel 0.5
```

のように、まずは小さい指令値から確認します。

ここでの目的は、高速に動かすことではなく、

- 指令に対してモータが反応するか
- 符号方向が意図通りか
- 異音や振動がないか
- 電流や温度が不自然に上がらないか
- disable で安全に止められるか

を確認することです。

動作確認後は、必ず無効化します。

```text
edulite05> disable
```

# ハマりどころ

## 1. CANインターフェース名

CLI側では `can0` を使う前提にしています。

Linux側でCANデバイスが `can0` になっているか確認します。

```bash
ip link
```

`can0` が存在しない場合は、CANインターフェースのドライバや設定を確認します。

## 2. CAN bitrate

CAN通信では、PC側とモータ側の bitrate が一致している必要があります。

たとえば `can0` を設定する場合は、環境に応じて以下のように設定します。

```bash
sudo ip link set can0 down
sudo ip link set can0 type can bitrate 1000000
sudo ip link set can0 up
```

bitrate はモータ側の仕様に合わせる必要があります。

## 3. モータIDが分からない

モータIDが分からない場合は、今回のように `scan id` で探索します。

ただし、スキャンは対象範囲内のIDに対して順番にリクエストを送るため、接続台数やタイムアウト設定によっては少し時間がかかります。

## 4. いきなり enable しない

初回確認では、いきなり `enable` して指令を出すのではなく、以下の順番にしています。

```text
scan id
use id <motor_id>
get all
disable
mode motion_control
enable
```

特に実機モータでは、符号やゼロ点、制御モードの認識違いで意図しない動作をする可能性があります。

## 5. ゼロ点の扱い

起動ログにも以下の注意を出しています。

```text
NOTE: zero is software zero only. Motor internal zero is not modified.
```

今回のCLIで扱うゼロ点は、基本的にソフトウェア上のゼロです。  
モータ内部の機械的ゼロや不揮発設定を変更するものではありません。

ゼロ点を永続的に変更する場合は、モータ側の仕様を十分確認してから行う必要があります。

# 今後やりたいこと

今回で、EduLite 05 / RobStride 05 を SocketCAN 経由で認識し、CLIから状態取得や基本操作を行えるところまで確認できました。

今後は、このモータを使って以下を試していきたいです。

- 二輪倒立振子への組み込み
- 速度制御・トルク制御の確認
- 電流値からのトルク推定
- MuJoCo上のモデルとの比較
- 実機とシミュレーションの差分確認
- 回生電力やバス電圧変動の観察

特に、二輪倒立振子では低速域のトルク応答やバックドライブ性が効いてくるはずなので、QDDアクチュエータの特徴を実機で確認する題材として面白そうです。

# おわりに

今回は、EduLite 05 / RobStride 05 を SocketCAN 経由で接続し、Python の対話CLIからモータIDスキャンや状態取得を行うところまで整理しました。

モータ制御は、理論だけでなく、実際に通信できること、状態が読めること、安全に有効化・無効化できることが重要です。

今回のCLIはまだ実験用ですが、今後の二輪倒立振子や脚式ロボット実験の土台として使っていく予定です。

次回以降は、制御則や SymPy を使った運動方程式の実装、MuJoCoとの接続あたりも整理していきたいと思います。