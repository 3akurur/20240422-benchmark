---
marp: true
# dark theme
class: invert
---
<!-- headingDivider: 1 -->

# Rubyでのベンチマークテストのやり方

# Author

3akurur

![height:200](profile.jpg)
github

![height:200](qr_github.png)

# License

CC BY-NC-SA 4.0

[![CC BY-NC-SA 4.0](https://licensebuttons.net/l/by-nc-sa/4.0/88x31.png)](http://creativecommons.org/licenses/by-nc-sa/4.0/)

Copyright (C) 2024 3akurur

# About this contents

GitHub Pages: [https://3akurur.github.io/20240422-benchmark/](https://3akurur.github.io/20240422-benchmark/)

Repository: [https://github.com/3akurur/20240422-benchmark - GitHub](https://github.com/3akurur/20240422-benchmark)

![height:200](qr_github_pages.png)

# 今回ベンチマークを取る2つの処理

```each_with_index``` ・ ```each_cons``` この二つのベンチマークを取っていく。

```rb
array = Array.new(10) { rand(1..100) }
array.each_with_index do |value, index|
  next_value = value[index + 1]
  break if next_value.nil?
  # 何かしらの処理
end
```

```rb
array = Array.new(10) { rand(1..100) }
array.each_cons(2){|v| # 何かしらの処理 }
```

# each_with_index と eacH_cons について

処理内容は、重複ありでn要素ずつの配列を返す処理。

```rb
irb(main):001> array = Array.new(10) { rand(1..100) }
=> [44, 100, 46, 19, 73, 45, 77, 63, 13, 59]
irb(main):002> array.each_cons(2){|v| }
=> [44, 100, 46, 19, 73, 45, 77, 63, 13, 59]
irb(main):003> array.each_cons(2){|v| p v}
[44, 100]
[100, 46]
[46, 19]
[19, 73]
[73, 45]
[45, 77]
[77, 63]
[63, 13]
[13, 59]
```

# 問題のあるやり方

パッと思いつくの以下の方法。
間違っている訳ではないが、計測したい処理以外にCPUが費やしている時間も計測されてしまう。
今回紹介するモジュールを使うと、プログラムが実行に費やした時間自体を計測してくれる。

```rb
array = Array.new(10) { rand(1..100) }

start_time = Time.now
array.each_cons(2){|v| # 何かしらの処理 }
puts Time.now - start_time
```

# benchmarkの使い方

benchmarkの使い方を紹介していく。まずは以下の形式通りに実装して動かしてみる。

```rb
require 'benchmark'

Benchmark.bm 10 do |r|
  r.report "処理1" do
    # (計測したい処理その1)
  end
  r.report "処理2" do
    # (計測したい処理その2)
  end
end
```

# 実際に試してみよう！
# 実際に試してみよう！ 1

Benchmark.bm の引数で試行回数を指定できる。
今回だと、同じ処理を100回試してみた平均値を出してくれる。

```rb
require 'benchmark'
array = Array.new(100_000_000) { rand(1..100) };nil

Benchmark.bm 100 do |r|
  r.report "each_with_index" do
  array.each_with_index do |value, index|
    next_value = value[index + 1]
    break if next_value.nil?
  end
  end
  r.report "each_cons" do
    array.each_cons(2){|v| }
  end
end
```

# 実際に試してみよう！ 2

実行結果は以下。

```rb
                                                                                                           user     system      total        real
each_with_index                                                                                        7.162238   0.022551   7.184789 (  7.238424)
each_cons                                                                                             11.637019   0.107252  11.744271 ( 11.956509)
```

each_consの方が1.5倍近く遅い？やってること変わんないのにそんなに違いが出るのだろうかという疑問が湧いた。

# 実際に試してみよう！ 3

じっくり確認してみると、 ```each_with_index``` の方は配列を作っていないのに関わらず、 ```each_cons``` は ```v``` に配列が入るので処理内容に違いがある事に気付く。

```rb
require 'benchmark'
array = Array.new(100_000_000) { rand(1..100) }

Benchmark.bm 100 do |r|
  r.report "each_with_index" do
  array.each_with_index do |value, index|
    next_value = value[index + 1]
    break if next_value.nil?
  end
  end
  r.report "each_cons" do
    array.each_cons(2){|v| }
  end
end
```

# 実際に試してみよう！ 4

```each_with_index``` の方でも配列を作る様にすれば、実行内容は同じになって正しくどちらの処理が速いか確認出来るはず。

```rb
require 'benchmark'
array = Array.new(100_000_000) { rand(1..100) }

Benchmark.bm 100 do |r|
  r.report "each_with_index" do
  array.each_with_index do |value, index|
    next_value = value[index + 1]
    break if next_value.nil?
    [value, next_value]
  end
  end
  r.report "each_cons" do
    array.each_cons(2){|v| } # 何かしらの処理
  end
end
```

# 実際に試してみよう！ 5

実行結果はこうなった。

```rb
                                                                                                           user     system      total        real
each_with_index                                                                                       13.494490   0.148602  13.643092 ( 13.885159)
each_cons                                                                                             12.260557   0.141211  12.401768 ( 12.747879)
```

前回のと比較しても数値に大きな変化はないのでこれで正しい結果が取れているだろう。

```rb
                                                                                                           user     system      total        real
each_with_index                                                                                        7.162238   0.022551   7.184789 (  7.238424)
each_cons                                                                                             11.637019   0.107252  11.744271 ( 11.956509)
```

# データの見方について

# データの見方について 1

データの見方についてざっくり紹介します。

- user
  - ユーザーCPU時間。ユーザープログラムの実行に費やした時間。
  -  ```ruby``` 処理系が働いた時間
- system
  - システムCPU時間。OSがファイルの読み書きなどの基本的な機能に費やした時間。

```rb
                                                                                                           user     system      total        real
each_with_index                                                                                       13.494490   0.148602  13.643092 ( 13.885159)
each_cons                                                                                             12.260557   0.141211  12.401768 ( 12.747879)
```

# データの見方について 2

- total
  - ```user``` と ```system```の和。
- real
  - 処理の開始から終了までの時間
  - Time.now使った計測方法と同じ筈。ここはみなくていい。
  - ただ、今回の実行結果だと```user```と```real```の結果があまり変わらなかったので、とりあえず実行結果が見やすいモジュールがあるという認識になった。

```rb
                                                                                                           user     system      total        real
each_with_index                                                                                       13.494490   0.148602  13.643092 ( 13.885159)
each_cons                                                                                             12.260557   0.141211  12.401768 ( 12.747879)
```