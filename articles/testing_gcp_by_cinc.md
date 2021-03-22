---
title: CINCを使ったGCPの設定内容のテスト
emoji: 📝
type: tech
topics: [gcp, Inspec, CINC]
published: false
---

# 概要
CINCを使ったGCPの設定値をテストする方法について。詳細設計に沿ってGCPが設定されているかの確認にCINCを使う方法について説明します。
CINCはChef Softwareが提供するソフトウェアのOSSです。よって、Inspecの商用利用において、CINCを使うことで、ライセンス購入せずに利用が可能となります。

# [CINCとは](https://cinc.sh/goals/)
[Chef Softwareは2019年4月2日より、全製品をソフトウェア配布物に商用ライセンス条項の適用を開始しました。](https://blog.chef.io/chef-software-announces-the-enterprise-automation-stack)そのため、現在、商用利用するには有償ライセンスの購入が必要となります。そこで、CINC(CINC is not Chefの略)はChef Softwareが提供しているソフトウェアを自由に使えるように活動しています。

CINCプロジェクトはChef Softwareのリポジトリのコードをビルドし、パッチ適用したCINCバイナリを配布しています。このバイナリのライセンスがApache2.0となっており商用利用可能となっています。(EULAやライセンス不要)
CINCはChef Softwareのソフトウェアのみを対象にしているため、その他のソフトウェアは対象にしていません。(test-kitchenなど)しかしながら、Chef Softwareのソフトウェアとの互換のあるサードパーティツールツールとの互換も保証するようリリースしています。

Chef Softwareが提供する各ソフトウェアごとのCINCバイナリは、以下の通りとなっています。

- Cinc Workstation: Chef Workstation
- Cinc Auditor: Chef Inspec
- Cinc Dashboard: Chef Automate
- Cinc Packager: Chef Habitat

この記事では、Inspecを使ったクラウドの設定をテストする例を示すので、`Cinc Auditor`の使い方を説明します。

# Inspec
Inspecはシステムの現在状態を確認し、詳細設計などのあるべき状態であるかを確認します。状態の確認はSDKを使い実装されたメソッドを使い稼動中のシステムの状態値を取得し確認します。
各クラウドの状態を確認するメソッドは、[このページ](https://docs.chef.io/inspec/resources)を参照してください。

## [CINC Auditorのインストール方法](https://cinc.sh/start/auditor/)
 
 スクリプトによる各OSでのインストール方法は下記の通りです。
 
 - UNIX/Linux/MacOS
 
 ```
$ curl -L https://omnitruck.cinc.sh/install.sh | sudo bash -s -- -P cinc-auditor -v 4
 ```
 
 - Windows
 
 ```
 . { iwr -useb https://omnitruck.cinc.sh/install.ps1 } | iex; install -project cinc-auditor -version 4
 ```
 
 bundlerによるインストール方法は、下記の通りです。
 
 ```
source "https://packagecloud.io/cinc-project/stable" do
  gem "cinc-auditor-bin"
end
 ```

各バージョンのバイナリは、下記ページのOpen Source Labより、実行環境のOSに対応するバイナリを取得してください。

- Cinc Open Source Lab Mirrors: http://downloads.cinc.sh/files/stable/cinc/

インストールが完了すると、下記のように `cinc-auditor` コマンドを実行することができます。

```
$ cinc-auditor version
4.24.28
```

また、CINCのDockerイメージはDockerHubの[cincproject/auditor](https://hub.docker.com/r/cincproject/auditor)で提供されています。

## Inspecの書き方
InspecはRSpecベースの言語であるため、RSpecに準拠した文法となります。Inspecのディレクトリ構成は以下の通りとなります。

```
./
  ├── controls
  │   └── main.rb
  ├── files
  │   └── lamb_actual.yaml
  ├── attribute.yaml  
  ├── inspec.yml
  └── inspec.lock
```

- controls: テスト内容のコードが記述されたコード
- files: テスト内容のコードが呼び出す設定値などを定義したファイル
- attribute.yaml: テスト実行時に渡すファイル
- inspec.yml: 実行するinspecの内容を定義するファイル
- inspec.lock: 実行環境のinspecバージョン定義するファイル

上記の中で実行に必須となるのは、 `controls` と `inspec.yml` のみです。
このディレクトリのrootでinspecコマンドを実行すると、定義したテストを実行します。

### inspec.ymlの設定内容
inspec.ymlは、実行するテストの情報(profile)を定義するファイルとなっています。定義する内容は、以下の通りです。

| 項目            | 内容                                      | 必須 |
|-----------------|-------------------------------------------|------|
| name            | profileで一意になる名前                   | ○    |
| title           | profileのnameの読み易い値                 | ×    |
| maintainer      | profileを管理するユーザー名               | ×    |
| copyright       | copyright                                 | ×    |
| copyright_email | profileの管理者のメールアドレス           | ×    |
| license         | profileのライセンス                       | ×    |
| summary         | profileのサマリ                           | ×    |
| description     | profileの具体的な内容                     | ×    |
| version         | profileのバージョン                       | ×    |
| inspec_version  | inspec のバージョン                       | ×    |
| supports        | profileがサポートするplatform             | ×    |
| depends         | profileが依存するライブラリなどを定義する | ×    |
| inputs          | controlsのテストが参照するinputを定義する | ×    |

sshを実行する踏台サーバーのテストのinspec.ymlの例は以下の様になります。

```
name: ssh
title: Basic SSH
maintainer: Chef Software, Inc.
copyright: Chef Software, Inc.
copyright_email: support@chef.io
license: Proprietary, All rights reserved
summary: Verify that SSH Server and SSH Client are configured securely
version: 1.0.0
supports:
  - platform-family: linux
depends:
  - name: profile
    path: ../path/to/profile
inspec_version: "~> 2.1"
```

#### クラウドのテスト方法
AWS,GCP,Azureなどのパブリッククラウドをテストする方法は、inspec.ymlの`depends`に対象のクラウドのライブラリと`supports`に対象のプラットフォームを定義します。
例えばGCPをテストする場合、inspec.ymlは以下のようになります。

```
name: ssh
depends:
  - name: inspec-gcp
    url: https://github.com/inspec/inspec-gcp/archive/<tag version>
supports:
  - platform: gcp
```

`tag version` は、GitHubの[tagsページ](https://github.com/inspec/inspec-gcp/tags)から、利用する対象のバージョンを指定します。例えば、1.8.8バージョンを利用する場合、tag versionに `v1.8.8`を指定します。

### Inspecのサンプル (HelloWorld)
ここでは、ローカルにあるyamlファイル(hello_world.yaml)の内容が、`name: HelloWorld`という文字列が含まれているかのテストサンプルを示します。

ディレクトリ構造は、以下の通りとします。

```
helloworld
 ├── controls
 │   └── main.rb
 ├── files
 │   └── hello_world.yaml
 └── inspec.yml
```

main.rb,hello_world.yamlとinspec.ymlの内容は以下の通りです。

- main.rb

```
actual = yaml(content: inspec.profile.file("hello_world.yaml"))

describe actual do
  its(:name) { should eq "HelloWorld" }
end
```
`inspec.profile.file()`メソッドにより、`files`ディレクトリのファイルをファイルパスなしに参照することができます。


- hello_world.yaml

```
name: HelloWorld
```

- inspec.yml

```
name: "test"
title: "testing hello world yaml file"
```

上記のファイルを作成し、ディレクトリのルートにて下記コマンドを実行しテストを実施します。

```
$ cinc-auditor exec .
```

実行結果は以下のようになります。

```
Profile: testing hello world yaml file (test)
Version: (not specified)
Target:  local://

  YAML content
     ✔  name is expected to eq "HelloWorld"

Test Summary: 1 successful, 0 failures, 0 skipped
```

# GCPの設定内容のテスト
次に、下記のような構成のシステムについてテストをします。

![system_arch](https://storage.googleapis.com/zenn-user-upload/km1j41dvy0de2ctgp27asvwawjij)

このシステムは、GCP上にVPCネットワークを作成し、そのVPCネットワーク内にGCEを作成し、その上でNginxが稼動するWebシステムです。
GCPの設定項目は、以下の通りでその内容をテストします。

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

これらのシステムの構築するTerraformコードは、この[リポジトリ](https://github.com/AtsushiKitano/cinc-test-sample)のsrcディレクトリを参照してください。
システムの構築は以下のコマンドで作成されます。

```
$ terragrunt run-all apply
```

CINCのテストは、さきのリポジトリのtestディレクトリに格納されています。テストは以下のように実行します。

```
$ cinc-auditor exec ./test -t gcp://
```

実行には、テスト対象のプラットフォームを指定する必要があり、`-t`で指定します。今回はGCPをテストするため、`gcp://`を渡します。
実行の結果、以下のような結果が出力されます。

```
Profile: nginx web system sample system test (http-system)
Version: (not specified)
Target:  gcp://764086051850-6qr4p6gpi6hn506pt8ejuq83di341hur.apps.googleusercontent.com

  ✔  network: vpc,subnetwork,firewallの設定
     ✔  Network sample is expected to exist
     ✔  Network sample name is expected to cmp == "sample"
     ✔  Subnetwork sample is expected to exist
     ✔  Subnetwork sample name is expected to cmp == "sample"
     ✔  Subnetwork sample region is expected to match "asia-northeast1"
     ✔  Subnetwork sample ip_cidr_range is expected to cmp == "192.168.10.0/24"
     ✔  Firewall ingress-sample is expected to exist
     ✔  Firewall ingress-sample name is expected to cmp == "ingress-sample"
     ✔  Firewall ingress-sample direction is expected to cmp == "INGRESS"
     ✔  Firewall ingress-sample priority is expected to cmp == 1000
     ✔  Firewall ingress-sample source_ranges is expected to be in "0.0.0.0/0"
     ✔  Firewall ingress-sample FirewallAllowed ip_protocol is expected to cmp == "tcp"
     ✔  Firewall ingress-sample FirewallAllowed ports is expected to be in "80"
  ✔  gce: gce instance, diskの設定
     ✔  Instance sample is expected to exist
     ✔  Instance sample name is expected to cmp == "sample"
     ✔  Instance sample zone is expected to match "asia-northeast1-b"
     ✔  Instance sample machine_type is expected to match "f1-micro"
     ✔  Instance sample InstanceNetworkInterfaces subnetwork is expected to match "sample"
     ✔  Instance sample InstanceDisks source is expected to match "sample"
     ✔  Disk sample is expected to exist
     ✔  Disk sample name is expected to cmp == "sample"
     ✔  Disk sample zone is expected to match "asia-northeast1-b"
     ✔  Disk sample source_image is expected to match "ubuntu-2004"
     ✔  Disk sample size_gb is expected to cmp == 20


Profile: Google Cloud Platform Resource Pack (inspec-gcp)
Version: 1.8.8
Target:  gcp://764086051850-6qr4p6gpi6hn506pt8ejuq83di341hur.apps.googleusercontent.com

     No tests executed.

Profile Summary: 2 successful controls, 0 control failures, 0 controls skipped
Test Summary: 24 successful, 0 failures, 0 skipped
```

## ライブラリが準備されていない設定項目のテスト
先の説明ではGCPの設定をテストするためにinspecのgcpメソッドを利用しました。しかし、GCPのshared vpcの設定項目などは準備されていなかったりします。そのときの解決策のひとつを紹介します。

構築したシステム設定値をSDKを用い取得しファイルに出力し、その値が想定したものか確認することでテストを実施することができます。
例えば先の例で、VPCネットワークのテストをinspecのメソッドを使わずにおこなうには`gcloud`コマンドで設定値をyaml形式で出力した後に生成したyamlファイルの項目をcincのテストで確認します。
この内容のテストは先のリポジトリのno_lib_testのディレクトリを参照してください。

# まとめ
InspecのOSSであるCINCについて、GCP上に構築したシステムをテストしながら説明をしました。
Inspecは設定項目すべてをテストするメソッドを揃えていませんが、SDKを使いながら項目のテストをすることができます。
また、OSSであるため、案件でライセンス購入をせずともInSpecの活用ができます。
以上より、CINCを使うことでソフトウェアのようにインフラにもテストを導入することができるので、活用を検討してみてください。
