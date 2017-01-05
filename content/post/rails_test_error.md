+++
title = "rails testしたらセグフォで怒られた"
menu = ""
images = [
]
slug = "rails_test_segmentation_fault"
tags = [
"rails"
]
banner = ""
draft = false
categories = [
"error"
]
date = "2017-01-05T17:48:53+09:00"

+++

## エラー
railsのテストを実行したらセグフォで怒られた．
```
$ rails test
/Users/[username]/.rbenv/versions/2.2.3/lib/ruby/gems/2.2.0/gems/activerecord-5.0.0.1/lib/active_record/connection_adapters/sqlite3_adapter.rb:27: [BUG] Segmentation fault at 0x00000000000110
ruby 2.2.3p173 (2015-08-18 revision 51636) [x86_64-darwin14]

-- Crash Report log information --------------------------------------------
   See Crash Report log file under the one of following:
     * ~/Library/Logs/CrashReporter
     * /Library/Logs/CrashReporter
     * ~/Library/Logs/DiagnosticReports
     * /Library/Logs/DiagnosticReports
   for more details.
︙
```
のようなエラーが出た．  

ちなみに`test:models`のように部分的にテストするとちゃんと動く．
いろいろ試してもダメだったが，結局のところよくわからんが，再起動したら治った．  

## 結論
再起動は偉大．

### [メモ]実行時の出力をファイルに書き出す
エラーログが多すぎて流れきってしまってエラーを追えないとき
```
$ rails test 2>log.txt
```
でファイルに出力できる．
`2`は標準エラー出力，`1`は標準出力．  
ちなみに両方出力する場合は，
```
$ [comannd] 1>log.txt 2>&1 # こうする
$ [comannd] 1>log.txt 2>log.txt # これは上書きしてしまうので間違い
```
