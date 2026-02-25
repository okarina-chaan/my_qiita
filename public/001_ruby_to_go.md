---
title: Ruby初学者がGoで天気CLIアプリを作って感じた違い
tags:
  - Ruby
  - Go
private: true
updated_at: '2026-02-25T23:42:35+09:00'
id: 0600d30059b9544f4646
organization_url_name: null
slide: false
ignorePublish: false
---
# 0. 導入
こんにちは！  
卒制がちょっと落ち着いてきたプログラミングスクールRUNTEQ生です。
スクールではRuby中心に学習してきましたが、  
「Rubyらしさってなんだろう」と思ったことがきっかけで、他の言語に興味が出てきました。
「よりシンプルで、もう少しだけ低レイヤーによってて、ドキュメントが充実している言語はないかなあ」と探していたら
Goという言語に出会いました。
その文法のシンプルさとTour of Goの親切さに惹かれて  
ちょこっと触ってみることにしました。  
今は「CLIで動く小さなお天気アプリ」を作っています。  
その中で感じた違いをまとめます。  

# 1. Rubyの世界では意識しなかった型定義
Rubyの感覚ではJSONはHashとして格納しましたが、  
Goではstruct(構造体)として格納します。

### Rubyの場合
```:ruby
def self.fetch(today)

# ここまでOpen Meteoのエンドポイント（urlとする）の作成
	response = Net::HTTP.get(uri)
	data = JSON.parse(response.body)
	temperature = data["hourly"]["temperature_2m"][0]
end

```
RubyではこのようにネストしたHashへのアクセスによって時間と気温を取得できます。  
キーはいずれも文字列なので、エディタの補完ができないかもしれません。  
つまり、タイプミスをしてしまっても、実行してエラーが出るまで気づくことができません。  

```:ruby
def self.fetch(today)
	response = Net::HTTP.get(uri)
	data = JSON.parse(response.body)
	temperature = data["hourly"]["temprature_2m"][0]  // ←タイポ
end
```
↑これをコンソールで呼び出すとnilが返ってくる



### Goの場合
```:go
// ここまでOpen Meteoのエンドポイントの作成

type WeatherResponse struct {
	Latitude  float64    `json:"latitude"`
	Longitude float64    `json:"longitude"`
	Hourly    HourlyData `json:"hourly"`
}
```
このように、JSONで取得する値についてそれぞれ型定義し、  
構造体として落とし込みます。  
`Latitude float64 `json:"latitude"`をみてみましょう。  
`json:"latitude"`はJSONのキー名と構造体のフィールドを対応させるためのタグです。
外部から取ってきたデータ形式とGoの型をつなぐ役割になっています。  
Goでは、キーの打ち間違いなどはコンパイル時に防げるので、  
Rubyの良くも悪くも「よしなにする文化」とは違って、「手綱を掴んだ安心感」があります。  


## 
