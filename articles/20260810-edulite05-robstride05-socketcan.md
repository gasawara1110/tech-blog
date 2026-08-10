---
title: "RobStride EduLite 05 をSocketCANで動かしてみた"
emoji: "🦾"
type: "tech"
topics: ["robotics", "motor", "can", "socketcan", "python"]
published: false
---

# 1. はじめに

今回は、小型QDD系アクチュエータである **RobStride EduLite 05** を、Linux の **SocketCAN** 経由で動かしてみた記録です。

前回の記事では、歩行ロボットで使われる QDD アクチュエータについて整理しました。

- [歩行ロボットとQDDアクチュエータ入門](https://zenn.dev/kaz1110/articles/20260720-qdd-actuator-for-legged-robots)

QDD アクチュエータは、低減速比によるバックドライブ性や力制御との相性の良さから、歩行ロボットやヒューマノイドでよく使われています。

今回はその流れで、実際に手元の EduLite 05 / RobStride 05 を CAN 通信で認識し、状態取得や簡単な制御ができるところまで確認しました。

この記事は、厳密なプロトコル解説というより、**「まず実機を安全に認識して、CLIから触れるようにする」** ところまでの動作メモです。

# 2. 注意

実機モータは、意図せず回転すると危険です。

初回確認時は、以下のような安全対策を強く推奨します。

- モータをしっかり固定する
- 手やケーブルを回転部に近づけない
- 無負荷、または軽負荷で確認する
- 電流・トルク・速度・位置の上限を小さくする
- まずは状態取得やIDスキャンなど、動かないコマンドから確認する
- 異常時にすぐ電源を切れるようにする

今回の確認でも、最初は `scan id` やパラメータ読み出しなど、モータを動かさない操作から始めています。

# 3. 今回やること

今回の記事で扱う内容は以下です。

- RobStride EduLite 05 を SocketCAN で接続する
- Python 製の対話CLIを起動する
- CAN接続を確認する
- モータIDをスキャンする
- モータ状態を読む
- 制御モードを設定して有効化する
- 各種制御モードで動かす


# 4. 使用機材・環境

今回使ったものは以下です。

- RobStride EduLite 05 
- Linux PC（Raspberry Pi 5, Ubuntu 24.04 LTS）
- SocketCAN 対応USB-CANコンバータ（[innomaker USB2CAN-C](https://www.amazon.co.jp/dp/B09K3LL93Q?ref=ppx_yo2ov_dt_b_fed_asin_title&th=1)）
- Python 3
- 自作 Python SDK / CLI
- 安定化電源（24Vで使用、[SPS-3010V](https://www.amazon.co.jp/dp/B0DDPVDWPZ?ref=ppx_yo2ov_dt_b_fed_asin_title&th=1)）
- モータマウントおよびモータアーム（3Dプリンタで自作）
- モータマウント固定用クランプ([Takagi HQB-100-2P](https://www.amazon.co.jp/dp/B00G8PR80G?ref=ppx_yo2ov_dt_b_fed_asin_title&th=1))

CAN インターフェースは Linux 側で `can0` として認識させています。

今回の実験では、モータを安全に固定するために、3Dプリンタで自作したモータマウントとモータアームを使用しました。  
机上でモータをそのまま回すと危険なので、初期の動作確認ではこのように固定治具を用意しておくと安心です。

![](/images/20260810-edulite05-robstride05-socketcan/motor_mount_arm.jpg)

図：動作確認に使用した自作モータマウントとモータアーム

# 5.リポジトリ構成

今回の確認用に、Python SDK と対話型CLIを用意しました。

Python SDKは [Robstride公式のSDK](https://github.com/RobStride/EDULITE_A3/tree/main/el_a3_sdk/el_a3_sdk) を使用しています。
SDKはPythonライブラリなのですが、もともとEdulite 05/Robstride 05を使ったロボットアームの制御用らしく、モータ単体を動かすのには不必要なファイルも含まれており、それらは除外しています。

対話型CLI(interactive_motor_cli.py)は自作です。

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

元のSDKがMITライセンスだったので、SDKを含めた上記の構成ファイル類をまとめて[Gitリポジトリ](https://github.com/gasawara1110/edulite05-python-socketcan)に上げていますのでご利用ください。


# 6.Pythonコードの確認

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

# 7.CLIを起動する

次に、対話型CLIを起動します。

```bash
python3 examples/interactive_motor_cli.py
```

起動すると、以下のような表示が出ます。

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

# 8.モータIDをスキャンする

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
found motor: motor_id=0x7F (127), vbus=24.352 V, run_mode=0 (motion_control)
---- scan result ----
- motor_id=0x7F (127)
```

`motor_id=0x7F` のモータが見つかりました。モータの出荷時IDの初期値は0x7Fのようです。

また、`vbus=24.352 V` と出ているので、モータ側でバス電圧も読めています。

ここで重要なのは、`scan id` が **read-only parameter request** であることです。  
つまり、モータを動かさずにIDや状態を確認できます。

初回接続では、こういう「動かない確認」から始めるのが安全です。

# 9.操作対象IDを変更する

スキャンで見つかったIDが `0x7F` だったので、CLIの操作対象を変更します。

```text
edulite05> use id 0x7F
```

これで、以降のコマンドは `motor_id=0x7F` のモータを対象にします。

# 10.状態を読む

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

# 11.制御モードを設定する

EduLite 05 / RobStride 05 では、用途に応じていくつかの制御モードを使い分けます。  
今回のCLIで扱っている主な制御モードは以下です。

| 制御モード | CLI上の指定例 | 指令・設定項目 | 主に制御される変数 | 概要 |
|---|---|---|---|---|
| motion_control | `mode motion_control` | `set pos`, `set vel`, `set torque`, `set kp`, `set kd` | 角度・速度・トルクの組み合わせ | 位置、速度、PDゲイン、トルクフィードフォワードを組み合わせて送る制御モード。いわゆるMIT modeに近い使い方ができる |
| velocity | `mode velocity` | `set vel <rad/s>` | 速度 | 目標速度を指定してモータを連続回転させる制御モード |
| current | `mode current` | `set current <A>` | 電流 | 目標電流を指定してモータを動かすモード。簡易的なトルク制御として使用できる |
| position_pp | `mode position_pp` | `set pos <rad>`, 必要に応じて `set vel <rad/s>` | 角度 | Point-to-Point 的な位置制御モード。指定した目標角度へ速度制限を使って移動する。移動中の速度は台形速度プロファイルに近くなる |
| position_csp | `mode position_csp` | `set pos <rad>` | 角度 | 上位制御器から周期的に目標角度を与える用途に向いた位置制御モード |

ここでいう「指令・設定項目」は、PC側からモータへ送る目標値やゲイン設定です。  
一方で「主に制御される変数」は、モータ内部の制御ループが追従させようとする物理量です。

`motion_control` では、目標角度、目標速度、Kp、Kd、トルクなどを組み合わせて指令します。  
今回のCLIでは、`set kp`、`set kd`、`set pos`、`set vel`、`set torque` などで指令値を設定します。  
角度だけでなく、速度やゲイン、トルクフィードフォワードも含めて扱えるため、関節制御や軌道追従に向いたモードです。

`velocity` では、PC側から目標速度を送ります。  
今回のCLIでは `set vel <rad/s>` で指令します。  
モータ内部では現在速度が目標速度に近づくように制御されるため、車輪駆動や速度応答の確認に使いやすいです。

`current` では、PC側から目標電流を送ります。  
今回のCLIでは `set current <A>` で指令します。  
モータ電流はトルクとおおよそ比例するため、簡易的なトルク制御として扱えます。  
ただし、実際の出力軸トルクには減速機効率、摩擦、温度、個体差などの影響が入るため、電流値そのものが正確な出力軸トルクになるわけではありません。

`position_pp` では、PC側から目標角度を送ります。  
今回のCLIでは `set pos <rad>` で指令します。  
Point-to-Point 的な位置制御モードで、目標角度を1回指定すると、その角度へ移動する動作になります。  
初回動作確認や、単発の角度移動を試す用途では扱いやすいです。

一方で `position_csp` も、PC側からは `set pos <rad>` で目標角度を送ります。  
ただし、用途としては上位側から周期的に目標角度を更新するような使い方に向いています。  
たとえば、軌道生成器や ROS 2 ノードから連続的に角度指令を送る場合は、このモードの方が使いやすいと考えています。

今回の「まず動かしてみる」用途では、最初に **`position_pp`** を使うのが分かりやすいです。  
理由は、目標角度を1回指定すれば、その角度へ移動する動作を確認しやすいからです。

# 12. 実際に動かす

制御モード設定と `enable` ができたら、低い指令値から動作確認します。

今回の目的は、高速・高トルクで動かすことではありません。  
まずは、CAN通信でモータを認識し、状態を読み、安全に小さく動かせることを確認します。

各モードで共通して、最初は必ず小さい値から試します。  
また、動作確認後は必ず `disable` します。これによりモータがの励磁がoffになるので安全です。

## 12.1 motion_controlモード

`motion_control` モードは、目標角度、目標速度、Kp、Kd、トルクなどを組み合わせて指令するモードです。  
関節制御や軌道追従に使いやすいモードです。

今回のCLIでは、まずモードを切り替えてから、Kp、Kd、目標角度を順番に設定します。

```text
edulite05> mode motion_control
edulite05> set zero
edulite05> set kp 1.0
edulite05> set kd 0.05
edulite05> enable
edulite05> set pos 0.2
edulite05> set pos 0.0
edulite05> disable
```

ここでは、以下のような流れで確認しています。

| コマンド | 内容 |
|---|---|
| `mode motion_control` | motion_control モードへ切り替え |
| `set zero` | 現在位置をソフトウェア上のゼロとして扱う |
| `set kp 1.0` | 位置ゲインを設定 |
| `set kd 0.05` | 速度ゲインを設定 |
| `enable` | モータを有効化 |
| `set pos 0.2` | 目標角度 0.2 rad を指令 |
| `set pos 0.0` | 目標角度 0.0 rad を指令 |
| `disable` | モータを無効化 |

`0.2 rad` は角度にすると約 11.4 deg です。  
初回動作確認では、このくらいの小さい角度から確認すると安全です。

`motion_control` モードでは、Kp や Kd を大きくしすぎると急に硬い動作になったり振動する可能性があります。  
最初は小さいゲインから確認するのが安全です。

## 12.2 velocityモード

`velocity` モードは、目標速度を指定してモータを回転させるモードです。  
車輪駆動や速度応答の確認に使いやすいモードです。

```text
edulite05> mode velocity
edulite05> enable
edulite05> set vel 0.1
edulite05> set vel 0.0
edulite05> disable
```

ここでは、目標速度として `0.1 rad/s` を指令しています。

| コマンド | 内容 |
|---|---|
| `mode velocity` | velocity モードへ切り替え |
| `enable` | モータを有効化 |
| `set vel 0.1` | 目標速度 0.1 rad/s を指令 |
| `set vel 0.0` | 目標速度を 0 に戻す |
| `disable` | モータを無効化 |

速度制御では、モータが回り続けるため、机上確認では特に注意が必要です。  
初回はモータをしっかり固定し、ケーブルが巻き込まれない状態で確認します。

## 12.3 currentモード

`current` モードは、目標電流を指定するモードです。  
モータ電流はトルクと関係するため、簡易的なトルク制御として扱えます。

```text
edulite05> mode current
edulite05> enable
edulite05> set current 0.02
edulite05> set current 0.0
edulite05> disable
```

ここでは、目標電流として `0.02 A` を指令しています。

| コマンド | 内容 |
|---|---|
| `mode current` | current モードへ切り替え |
| `enable` | モータを有効化 |
| `set current 0.02` | 目標電流 0.02 A を指令 |
| `set current 0.0` | 目標電流を 0 に戻す |
| `disable` | モータを無効化 |

電流制御モードは、力制御やトルク制御の確認に便利です。  
ただし、実際の出力軸トルクには減速機効率、摩擦、温度、個体差などの影響が入ります。  
そのため、電流値そのものが正確な出力軸トルクになるわけではありません。

## 12.4 position_ppモード

`position_pp` モードは、Point-to-Point 的な位置制御モードです。  
目標角度を1回指定すると、その角度へ移動する位置制御として使えます。

```text
edulite05> mode position_pp
edulite05> enable
edulite05> set pos 0.1
edulite05> set pos 0.0
edulite05> disable
```

ここでは、目標角度として `0.1 rad` を指令しています。

| コマンド | 内容 |
|---|---|
| `mode position_pp` | position_pp モードへ切り替え |
| `enable` | モータを有効化 |
| `set pos 0.1` | 目標角度 0.1 rad を指令 |
| `set pos 0.0` | 目標角度 0.0 rad を指令 |
| `disable` | モータを無効化 |

`position_pp` モードは、初回の「小さく動くか確認する」用途では扱いやすいです。  
目標角度を単発で与えればよいので、回転方向や動作量を確認しやすいです。

## 12.5 position_cspモード

`position_csp` モードは、Cyclic Synchronous Position 的な位置制御モードです。  
上位側から周期的に目標角度を更新する用途に向いています。

```text
edulite05> mode position_csp
edulite05> enable
edulite05> set pos 0.1
edulite05> set pos 0.0
edulite05> disable
```

ここでは、`position_pp` と同じように `set pos` で目標角度を指令しています。

| コマンド | 内容 |
|---|---|
| `mode position_csp` | position_csp モードへ切り替え |
| `enable` | モータを有効化 |
| `set pos 0.1` | 目標角度 0.1 rad を指令 |
| `set pos 0.0` | 目標角度 0.0 rad を指令 |
| `disable` | モータを無効化 |

`position_csp` は、単発の位置決めよりも、上位制御器から連続的に目標角度を送る用途に向いています。  
たとえば、軌道生成器や ROS 2 ノードから周期的に目標角度を更新する場合は、このモードの方が使いやすいと考えています。

ただし、今回の確認では `position_csp` は単発の位置指令で動作確認しています。  
連続的な周期軌道指令までは、この記事の範囲では扱っていません。

今回のような初回動作確認では、まず `position_pp` で単発の角度移動を確認し、その後で `position_csp` による周期指令を試す流れが安全だと思います。


# 13ハマりどころ
## 13.0 電源の確認

初歩的なことですがモータの電源のプラスマイナスを逆に刺さないでください。一発でモータが死ぬ可能性があります。

今回の実験を行ったとき、買ったばかりの安定化電源だったので、安定化電源の根元のバナナプラグを±を逆にしていたせいでモータを壊しかけました（苦笑

幸い0.5Aの電流制限をかけていたので何とかモータは生きてましたが、できればテスター等で電圧を確認してからモータに電力を供給しましょう。たまに怪しげ安定化電源だと指定した電圧が出ていない時もあるので。

設定電圧24Vで電流制限を0.5Aあたりから始めましょう。

## 13.1. CANインターフェース名

CLI側では `can0` を使う前提にしています。

Linux側でCANデバイスが `can0` になっているか確認します。

```bash
ip link
```

`can0` が存在しない場合は、CANインターフェースのドライバや設定を確認します。

## 13.2. CAN bitrate

CAN通信では、PC側とモータ側の bitrate が一致している必要があります。

たとえば `can0` を設定する場合は、環境に応じて以下のように設定します。

```bash
sudo ip link set can0 down
sudo ip link set can0 type can bitrate 1000000
sudo ip link set can0 up
```

bitrate はモータ側の仕様に合わせる必要があります。

## 13.3. モータIDが分からない

モータIDが分からない場合は、今回のように `scan id` で探索します。

ただし、スキャンは対象範囲内のIDに対して順番にリクエストを送るため、接続台数やタイムアウト設定によっては少し時間がかかります。

## 13.4. いきなり enable しない

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

## 13.5. ゼロ点の扱い

起動ログにも以下の注意を出しています。

```text
NOTE: zero is software zero only. Motor internal zero is not modified.
```

今回のCLIで扱うゼロ点は、基本的にソフトウェア上のゼロです。  
モータ内部の機械的ゼロや不揮発設定を変更するものではありません。

ゼロ点を永続的に変更する場合は、モータ側の仕様を十分確認してから行う必要があります。

# 14. 今後やりたいこと

今回で、EduLite 05 / RobStride 05 を SocketCAN 経由で認識し、CLIから状態取得や基本操作を行えるところまで確認できました。

今後は、このモータを使って以下を試していきたいです。

- 二輪倒立振子への組み込み
- 速度制御・トルク制御の確認
- 電流値からのトルク推定
- MuJoCo上のモデルとの比較
- 実機とシミュレーションの差分確認
- 回生電力やバス電圧変動の観察

特に、二輪倒立振子では低速域のトルク応答やバックドライブ性が効いてくるはずなので、QDDアクチュエータの特徴を実機で確認する題材として面白そうです。

# 15. おわりに

今回は、EduLite 05 / RobStride 05 を SocketCAN 経由で接続し、Python の対話CLIからモータIDスキャンや状態取得を行うところまで整理しました。

モータ制御は、理論だけでなく、実際に通信できること、状態が読めること、安全に有効化・無効化できることが重要です。

今回のCLIはまだ実験用ですが、今後の二輪倒立振子や脚式ロボット実験の土台として使っていく予定です。

次回以降は、制御則や SymPy を使った運動方程式の実装、MuJoCoとの接続あたりも整理していきたいと思います。