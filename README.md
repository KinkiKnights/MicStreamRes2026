# mic_stream — USBマイク音声のロスレス(FLAC)転送＋Python解析

USB マイクの音声を **可逆(FLAC)** で別デバイスへ転送し、受信側 Ubuntu の
**Python(GStreamer + numpy)** でリアルタイム解析するための最小構成。

```
 Raspberry Pi (送信)                              Ubuntu 解析PC (受信)
┌──────────────────────────┐   FLAC over TCP   ┌────────────────────────────┐
│ alsasrc(USBマイク)         │ ───────────────▶ │ tcpclientsrc → flacparse    │
│  → flacenc(可逆) → flacparse│   (全サンプル到達) │  → flacdec → appsink        │
│  → tcpserversink :5005     │                   │  → numpy(int16) → 解析       │
└──────────────────────────┘                   └────────────────────────────┘
```

## 設計のポイント

- **可逆(FLAC)**: マイクが出した `S16LE` サンプルを受信側でビット完全に復元。精密な音響解析でも信号が変質しない。
- **TCP**: ロスレス要件のため、UDP/RTP のようなパケットロスでサンプルが欠けるのを避ける。「多少の遅延は許容」という条件に合致。
- **負荷**: エンコード/デコードは GStreamer(C)が担当。Python は復元済み numpy 配列を受け取るだけ。
- **実測帯域**: 約 **437 kbps**(生PCM 48kHz/mono/16bit = 768 kbps の約 57%)。`COMP`(FLAC圧縮レベル)で調整可。
- マイクのネイティブ仕様は **mono / S16LE / 48000(or 44100)Hz** のためリサンプル無し＝完全可逆。

## 送信側(この Pi)

```bash
~/kk_ws/src/kk_rescue26_pi/mic_publisher/mic-publish.sh   # (kk_rescue26_pi リポジトリに移設)
# 既定: ALSA_DEV=hw:1,0  RATE=48000  PORT=5005  COMP=5
# 例: ALSA_DEV=hw:1,0 RATE=48000 PORT=5005 COMP=8 ./mic-publish.sh
```
この Pi が TCP サーバになり、受信側が接続しに来る(複数受信可)。録音デバイスは `arecord -l` で確認。

### master control への登録(任意)
camera / joy と同様に programs.json へ追加すれば Web UI から起動/停止できる:
```json
{"id": 3, "name": "mic", "type": "bash", "cmd": "ALSA_DEV=hw:1,0 RATE=48000 PORT=5005 /home/kk/kk_ws/src/kk_rescue26_pi/mic_publisher/mic-publish.sh"}
```

## 受信側(Ubuntu 解析PC)

必要パッケージ:
```bash
sudo apt install python3-gi python3-numpy \
  gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-tools
```
実行:
```bash
python3 mic_receiver.py --host <PiのIP> --port 5005 --rate 48000
# 端末には level(dBFS) と peak(Hz) が流れる
# ※ ログをパイプ/リダイレクトする場合は python3 -u で（stdout バッファ対策）
```

## 解析の差し替え

`mic_receiver.py` の `Analyzer.analyze(self, frame)` を書き換える。
`frame` は mono の `numpy.int16` 配列(既定 0.5 秒ごと、`hop_sec` で変更可)。
既定実装は RMS レベル(dBFS)と主要周波数(FFT)を表示するだけのサンプル。

## 動作確認済み(2026-06-22, kk05)

- USBマイク(`hw:1,0`, mono/S16LE/48000) → FLAC/TCP → flacdec → numpy までループバックで疎通。
- `gst-launch` で受信→WAV 復元、Python 受信で RMS/peak 解析の出力を確認。
- FLAC ストリーム実測 437 kbps。
