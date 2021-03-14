---
title: TerratestによるTerraformコードの単体テスト
emoji: 🔒
type: tech
topics: [Terraform, gcp, CICD]
published: false
---

# 概要
Terratestを使ったインフラのテスト方法について

# [Terratestとは](https://terratest.gruntwork.io/docs/getting-started/introduction/#watch-how-to-test-infrastructure-code)
TerratestはインフラをテストするGoライブラリです。
提供している機能としては、以下の機能があります。

- Terraformコードのテスト
- Packerテンプレートのテスト
- Dockerイメージのテスト
- Helm Chartのテスト
- SSH経由のホストへのコマンド実行
- AWS,Azure,GCP,k8sのAPI
- HTTPリクエストの作成
- shellコマンドの実行


Goのライブラリであり、HTTPリクエストを使いテストの記述もできるため、インフラの設定値のみだけではなく、自動化できる機能テスト、結合テストも一括して書くことが可能となります。

# 使い方
## 環境の要件
TerratestはGoライブラリであり、実行環境は1.13以上のバージョンである必要があります。

Terratestのライブラリは、以下のように取得します。
以下では、terratestのterraform,gcp,http-helper ライブラリの取得方法を示します。

- terratestライブラリ
```
go get github.com/gruntwork-io/terratest/modules/terraform
```

- gcpライブラリ
```
go get github.com/gruntwork-io/terratest/modules/gcp
```

- http-helperライブラリ
```
github.com/gruntwork-io/terratest/modules/http-helper
```

また、これらライブラリに依存するするライブラリを取得する必要があります。
実行環境で不足する場合は、取得してください。

## テストの書き方
Terratestの利用はテストで利用するモジュールをインポートして利用します。
テストの方法はtfstateのoutputブロックの値をテストします。  

参照の方法は、testコードからTerraformコードへの相対パスを`terrform.Options` の `TerraformDir` に渡し、`terraform.Options` オブジェクトを生成します。
このオブジェクトを`terraform`の`Output`メソッドに渡し、terraform outputコマンドを呼び出しoutputブロックの値を参照します。  

例えば、下記ディレクトリの構成としたとき、 `TerraformDir` に `../test` を渡して `terraform.Options`オブジェクトを生成します。
```
.
├── README.md
├── src # terraformコードのディレクトリ
│   ├── main.tf
│   ├── output.tf
│   ├── provider.tf
│   └── terraform.tf
└── test  # terratestコードのディレクトリ
    └── network_test.go
```
Terratestはterraformやその他のplatformのSDKをラップしています。
そのため、先に説明したように outputコマンドやその他のTerraformコマンドを呼ぶことができます。つまり、テストコード内部で terraform apply, terraform destroyによるリソースの変更が可能となります。

## HelloWorldのテスト
ここでは、Hello Worldの文字列を出力するTerraformをテストする方法について説明します。
この説明のディレクトリ構成は、以下のとおりです。
```
helloworld
├── helloworld_test.go  Hello WorldをテストするTerratestコード
└── main.tf             Hello Worldを出力するTerraformコード
```
Hello Worldを出力するTerraformコードは、以下の通りです。
```
output "helloworld" {
  value = "Hello World"
}
```
上記outputをテストするTerraformコードは、以下の通りです。
TerratestとTerraformのコードは同一ディレクトリにあるので、`TerraformDir` に `.` を渡します。
```
package helloworld

import(
    "testing"

    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestHelloWorld(t *testing.T) {
    t.Parallel()

    terraformOption := &terraform.Options{
        TerraformDir: ".",
    }

    actual := terraform.Output(t, terraformOption, "helloworld")
    expected := "Hello World"

    assert.Equal(t, expected, actual)
}
```

テストの実行方法は、下記の通りです。

1. Terraformコマンドで、`terraform apply`の実行
1. go コマンドでTerratestのコードの実行

上記の内容のコマンドは以下の通りです。

```
terraform apply
go test -v
```

## Webシステムのテスト
次に、下記のような構成のシステムについてテストをします。

![system_arch](https://storage.googleapis.com/zenn-user-upload/g8hbgck0z52xngmoevo1n1fysgl6)

このシステムは、GCP上にVPCネットワークを作成し、GCEを作成し、当該システムにNginxをインストールしたWebシステムです。
テストでは、GCPの設定とWebシステムへの通信を確認するテストを実施します。
GCPの設定項目およびWebシステムへの通信要件は、以下の通りで以下の内容をテストします。

- VPC ネットワークの設定

| vpc name | subnetwork name | subnetwork region | subnetwork cidr |
| ---      | ---             | ---               | ---              |
| sample   | sample          | asia-northeast1   | 192.168.10.0/24  |

- ファイアーフォールの設定

| name           | direction | target                    | source address | priority | rule type | protocol | port      |
| ---            | ---       | ---                       | ---            | ---      | ---       | ---      | ---       |
| ingress-sample | INGRESS   | VPC network上のすべてのVM | 0.0.0.0/0      | 1000     | allow     | tcp      | 80 |

- GCEの設定

| name   | machine type | zone              | subnetwork | tags | boot disk size | mashine image                   |
| ---    | ---          | ---               | ---        | ---  | ---            | ---                             |
| sample | f1-micro     | asia-northeast1-b | sample     | -    | 20             | ubuntu-os-cloud/ubuntu-2004-lts |

- ネットワーク疎通の設定

| Web IP address  | access port | HTTP Request | Response Status Code |
| ---             | ---        | ---          | ---                  |
| GCEのIPアドレス | 80         | GET          | 200                  |

これらのシステムを構築するTerraformコードは、 [当該コード](https://github.com/AtsushiKitano/terratest_sample/blob/master/src/main.tf) となります。
ここで利用しているTerraformモジュールは、 [こちらのモジュール](https://github.com/AtsushiKitano/assets/tree/master/terraform/gcp/modules) を参照しています。

Terraformの構築した内容をテストするテストは、[当該ディレクトリ内のコード](https://github.com/AtsushiKitano/terratest_sample/blob/master/test)です。

# さいごに
Terratestを使ったTerraformの設定内容のテスト方法をNginxのシステムをテストする例を使い説明しました。TerratestはGoのライブラリでありかつ、HTTPリクエストを実行するライブラリを提供しているため、ユニットテスト以上の機能テストも実施可能です。
Terraformの設定内容のテストは、outputの値を参照して実施するため、Terraformコードにテストすべき項目を記載する必要があります。また、Terratestのライブラリの制約により、outputを整形しないとうまくテストできません。
ユニットテスト以外にも使える点としては、その他のインフラのテストツールより優れていますが、Terraformのstatefileを参照している点や、出力の制約があるために導入にはその点を考慮する必要があります。
