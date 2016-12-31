+++
tags = [
"RaspberryPi",
"音声認識"
]
categories = [
"RaspberryPi"
]
draft = false
slug = "hey_raspi_part1"
date = "2016-12-31T18:52:59+09:00"
title = "ラズパイでSiriのような音声認識BOTを作ってみた part1"
menu = ""
banner = ""
images = [
]

+++

ラズパイにSiriみたいなことをさせようと思ったが，マイクとスピーカの購入がまだなのでとりあえず手元にあるMacで似たような処理を再現する．

## 仕様
今回作るもののイメージはこんな感じ  
```
私「ヘイ，ラズパイ！」
Raspi「ポンポンっ」（コマンド受け付けますよ音）
私「今日の天気は？」
Raspi「今日の天気は晴れです」
```
これを再現したい  

## 実装
音声認識 => [Julius](http://julius.osdn.jp/)  
天気情報取得 => Ruby

JuliusはOSSの音声認識エンジンで，認識精度自体はDocomo等の音声認識APIには劣るが，ローカルで動き，単語の登録等ができるのが強み．
常時認識しておきたいのでWebAPIを用いるよりこちらのほうがよいと判断した．

## 手順
### Juliusのインストール
ディクテーションキット  
- Mac  
http://sourceforge.jp/projects/julius/downloads/60416/dictation-kit-v4.3.1-osx.tgz
- Linux  
https://ja.osdn.net/projects/julius/downloads/60416/dictation-kit-v4.3.1-linux.tgz
をダウンロードしてくる

```
$ tar xvzf dictation-kit-v4.3.1-osx.tgz
$ cd dictation-kit-v4.3.1-osx
$ ./run-gmm.sh
```
これでJuliusが起動するはず
(最新版は4.4  https://ja.osdn.net/projects/julius/downloads/66544/dictation-kit-v4.4.zip だが，試してみた感じ精度が悪そうだったのでとりあえず4.3を使う)

適当にしゃべってみると
```
<<< please speak >>>Warning: strip: sample 0-66 has zero value, stripped
pass1_best:  こんにちは
sentence1:  こんにちは 。
<<< please speak >>>
```
うまくうごいてそう
このdictation-kitの中には必要ないものも多いので，必要なものだけを取り出してくる．
今回は`model`,`am-gmm.jconf`,`bin/julius`,`yomi2voca.pl`を取り出してきて適当に新しいフォルダを作る．
```
$ mkdir julius
$ cp dictation-kit-v4.3.1-osx/model dictation-kit-v4.3.1-osx/am-gmm.jconf dictation-kit-v4.3.1-osx/bin/julius dictation-kit-v4.3.1-osx/bin/yomi2voca.pl julius
```

### 単語の登録
動くことが確認できたので次に，自分が認識して欲しい単語の辞書を作る(julius/word.yomi)
```
#julius/word.yomi
ヘイラズパイ へいらずぱい
今日の天気 きょうのてんき
明日の天気 あしたのてんき
天気 てんき
今何時 いまなんじ
```
そしてこれらをJuliusが使える辞書形式に変換する．
Juliusは文字コードがEUC-JPなので変換して，辞書形式に変換する．
```
$ iconv -f utf-8 -t eucjp julius/word.yomi | ./julius/yomi2voca.pl > ./julius/word.dic
```

### 登録した辞書を用いる設定ファイルの作成
単語の登録が完了したので，反映させるため設定ファイル(julius/word.jconf)を作成する．
```
#julius/word.jconf
-w word.dic
-v model/lang_m/bccwj.60k.htkdic
-h model/phone_m/hmmdefs_ptm_gid.binhmm
-hlist model/phone_m/logicalTri
-n 5
-output 1
-input mic
-zmeanframe
-rejectshort 800
-charconv EUC-JP UTF-8
```
※この際，設定ファイルに記載する相対パスは，設定ファイルを起点とするパスになる．

### 起動の確認
```
./julius/julius -C julius/word.jconf -C julius/am-gmm.jconf
```
上記のコマンドで起動できる．
```
pass1_best: ヘイラズパイ
pass1_best_wordseq: ヘイラズパイ
pass1_best_phonemeseq: silB h e i r a z u p a i silE
pass1_best_score: -2283.777832
sentence1: ヘイラズパイ
wseq1: ヘイラズパイ
phseq1: silB h e i r a z u p a i silE
cmscore1: 1.000
score1: -2283.777832
```
こんな感じに今まで登録されていな方単語が認識されるようになった．
※注意すべきなのが，登録した単語ありきの認識をするため，なにかしらの言葉を発すると登録した単語のどれかに割り当ててしまうということ．

### Rubyとのつなぎ込み
[julius gem](https://github.com/hadzimme/julius)を使うと簡単にできる．  
その際，Juliusはモジュールモードで起動しておくこと．(julius-server.sh)  
```
# julius-server.sh
./julius/julius -C julius/word.jconf -C julius/am-gmm.jconf -module
```
Rubyスクリプト(main.rb)は以下のようにした．
```
# main.rb
SRC_DIR = File.expand_path(File.dirname(__FILE__))

require 'rubygems'
require 'julius'

class HeyRaspi
  def run(julius)
    begin
      julius.each_message do |message, prompt|
        case message.name
        when :RECOGOUT
          prompt.pause

          shypo = message.first
          whypo = shypo.first
          confidence = whypo.cm.to_f

          puts "#{message.sentence} #{confidence}"

          prompt.resume
        end
      end
    rescue REXML::ParseException
      puts "retry…"
      retry
    end
  end
end

puts "接続中…"
julius = Julius.new

puts "準備OK！"
hey_raspi = HeyRaspi.new
hey_raspi.run(julius)
```
ターミナルを二つ立ち上げ，ひとつで`julius-server.sh`を実行．  
もう一つは`main.rb`を実行．

そして，準備OKと表示されたら，話しかける！
```
接続中…
準備OK！
ヘイラズパイ 1.0
今日の天気 0.997
```
いい感じに認識できた〜  
あとはここにコマンドの中身を追加すれば完了！

### 天気の取得
は次回〜

とりあえず，今回は音声認識BOTの外枠だけ作成した．  
これの中身を変えるとそれぞれの好きなBOTが作れるのではないでしょうか〜
