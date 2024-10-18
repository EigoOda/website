---
title: kopsを使ったAWS上でのKubernetesのインストール
content_type: task
weight: 20
---

<!-- overview -->

このクイックスタートでは、AWS上にKubernetesクラスターを簡単にインストールする方法を紹介します。
[`kops`](https://github.com/kubernetes/kops)と呼ばれるツールを使用します。

kopsは、自動プロビジョニングシステムです。

* 完全自動インストール
* クラスターの特定にDNSを使用
* 自己回復: すべてがオートスケーリンググループで実行
* 複数のOSをサポート（Debian、Ubuntu 16.04をサポート、CentOS & RHEL、Amazon Linux、CoreOS）- [images.md](https://github.com/kubernetes/kops/blob/master/docs/operations/images.md)を参照
* 高可用性をサポート - [high_availability.md](https://github.com/kubernetes/kops/blob/master/docs/operations/high_availability.md)を参照
* terraform マニフェストを直接プロビジョンまたは、生成できる - [terraform.md](https://github.com/kubernetes/kops/blob/master/docs/terraform.md)を参照


## {{% heading "prerequisites" %}}

* [kubectl](/docs/tasks/tools/install-kubectl/)がインストールされている必要があります

* `kops`が64-bit (AMD64 and Intel 64)デバイスアーキテクチャに[インストール](https://github.com/kubernetes/kops#installing)されている必要があります

* [AWSアカウント](https://docs.aws.amazon.com/polly/latest/dg/setting-up.html)を持ち、[IAMキー](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#access-keys-and-secret-access-keys)を生成して、[設定](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html#cli-quick-configuration)する必要があります。IAMユーザーには[十分な権限](https://github.com/kubernetes/kops/blob/master/docs/getting_started/aws.md#setup-iam-user)が必要です。


<!-- steps -->

## クラスターの作成

### (1/5) kopsのインストール

#### インストール

kopsを[リリースページ](https://github.com/kubernetes/kops/releases)からダウンロードする（ソースをビルドするのも簡単です）

{{< tabs name="kops_installation" >}}
{{% tab name="macOS" %}}

コマンドを実行して最新のリリースをダウンロード

```shell
curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-darwin-amd64
```

特定のバージョンをダウンロードする場合、 コマンドの以下の部分を特定のkopsバージョンに置き換えてください。

```shell
$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)
```

v1.20.0をダウンロードする場合

```shell
curl -LO https://github.com/kubernetes/kops/releases/download/v1.20.0/kops-darwin-amd64
```

実行できるようにバイナリの権限を変更する。

```shell
chmod +x kops-darwin-amd64
```

バイナリをPATHが通っているディレクトリに移動する。

```shell
sudo mv kops-darwin-amd64 /usr/local/bin/kops
```

[Homebrew](https://brew.sh/)を使ってインストールすることも可能です。

```shell
brew update && brew install kops
```
{{% /tab %}}
{{% tab name="Linux" %}}

コマンドを実行して最新のリリースをダウンロード

```shell
curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
```

特定のバージョンをダウンロードする場合、 コマンドの以下の部分を特定のkopsバージョンに置き換えてください。

```shell
$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)
```

v1.20.0をダウンロードする場合

```shell
curl -LO https://github.com/kubernetes/kops/releases/download/v1.20.0/kops-linux-amd64
```

実行できるようにバイナリの権限を変更する。

```shell
chmod +x kops-linux-amd64
```

バイナリをPATHが通っているディレクトリに移動する。

```shell
sudo mv kops-linux-amd64 /usr/local/bin/kops
```

[Homebrew](https://docs.brew.sh/Homebrew-on-Linux)を使ってインストールすることも可能です。

```shell
brew update && brew install kops
```

{{% /tab %}}
{{< /tabs >}}


### (2/5) クラスター用のroute53ドメインの作成

kopsはクラスター内部と外部の検出にDNSを使用するため、クライアントからKubernetes APIサーバーに到達することができます。

kopsは、クラスターの名前についてはっきりとしたルールがあり、有効なDNS名でなければなりません。そうすることで複数のクラスターを混同することがなくなるため、同僚とクラスターを明確に共有することができ、IPアドレスを覚えることなくクラスターにアクセスすることができます。

複数のクラスターを分割するためにサブドメインを使用することができ、おそらく使用すべきです。この例では、`useast1.dev.example.com`を使用します。APIサーバーのエンドポイントは、`api.useast1.dev.example.com`となります。

Route53のホステッドゾーンは、サブドメインを提供します。あなたのホステッドゾーンは`useast1.dev.example.com`だけでなく、`dev.example.com`や`example.com`でも動作します。kopsは、これらのいずれでも動作するため、通常は組織上の理由で選択することになります（例えば、`dev.example.com`配下にレコードを作成することは許可されているが、`example.com`配下に作成することは許可されていない）。

あなたが`dev.example.com`をホステッドゾーンを使用していると仮定します。[通常のプロセス]((https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingNewSubdomain.html))または、`aws route53 create-hosted-zone --name dev.example.com --caller-reference 1`のようなコマンドでホステッドゾーンを作成できます。

NSレコードを親ドメインで設定し、ドメイン内のレコードが解決できるようする必要があります。ここでは、`example.com`に`dev`のNSレコードを作成します。もしルートドメイン名であれば、ドメインレジストラーにNSレコードを設定することになります（例えば`example.com` は、`example.com`を購入した場所で設定する必要があります）。

このステップは、混乱しやすいです（問題の一番の原因です！）。digツールを実行することで、クラスターが正しく設定されていることをダブルチェックすることができます。

`dig NS dev.example.com`

Route53がホステッドゾーンにアサインした4つのNSレコードが確認できます。

### (3/5) クラスターの状態を保存するS3バケットの作成

kops lets you manage your clusters even after installation.  To do this, it must keep track of the clusters
that you have created, along with their configuration, the keys they are using etc.  This information is stored
in an S3 bucket.  S3 permissions are used to control access to the bucket.

Multiple clusters can use the same S3 bucket, and you can share an S3 bucket between your colleagues that
administer the same clusters - this is much easier than passing around kubecfg files.  But anyone with access
to the S3 bucket will have administrative access to all your clusters, so you don't want to share it beyond
the operations team.

So typically you have one S3 bucket for each ops team (and often the name will correspond
to the name of the hosted zone above!)

In our example, we chose `dev.example.com` as our hosted zone, so let's pick `clusters.dev.example.com` as
the S3 bucket name.

* Export `AWS_PROFILE` (if you need to select a profile for the AWS CLI to work)

* Create the S3 bucket using `aws s3 mb s3://clusters.dev.example.com`

* You can `export KOPS_STATE_STORE=s3://clusters.dev.example.com` and then kops will use this location by default.
   We suggest putting this in your bash profile or similar.


### (4/5) クラスター設定の構築

Run `kops create cluster` to create your cluster configuration:

`kops create cluster --zones=us-east-1c useast1.dev.example.com`

kops will create the configuration for your cluster.  Note that it _only_ creates the configuration, it does
not actually create the cloud resources - you'll do that in the next step with a `kops update cluster`.  This
give you an opportunity to review the configuration or change it.

It prints commands you can use to explore further:

* List your clusters with: `kops get cluster`
* Edit this cluster with: `kops edit cluster useast1.dev.example.com`
* Edit your node instance group: `kops edit ig --name=useast1.dev.example.com nodes`
* Edit your master instance group: `kops edit ig --name=useast1.dev.example.com master-us-east-1c`

If this is your first time using kops, do spend a few minutes to try those out!  An instance group is a
set of instances, which will be registered as kubernetes nodes.  On AWS this is implemented via auto-scaling-groups.
You can have several instance groups, for example if you wanted nodes that are a mix of spot and on-demand instances, or
GPU and non-GPU instances.


### (5/5) AWSにクラスターを作成

Run "kops update cluster" to create your cluster in AWS:

`kops update cluster useast1.dev.example.com --yes`

That takes a few seconds to run, but then your cluster will likely take a few minutes to actually be ready.
`kops update cluster` will be the tool you'll use whenever you change the configuration of your cluster; it
applies the changes you have made to the configuration to your cluster - reconfiguring AWS or kubernetes as needed.

For example, after you `kops edit ig nodes`, then `kops update cluster --yes` to apply your configuration, and
sometimes you will also have to `kops rolling-update cluster` to roll out the configuration immediately.

Without `--yes`, `kops update cluster` will show you a preview of what it is going to do.  This is handy
for production clusters!

### 他のアドオンの参照

See the [list of add-ons](/ja/docs/concepts/cluster-administration/addons/) to explore other add-ons, including tools for logging, monitoring, network policy, visualization, and control of your Kubernetes cluster.

## クリーンアップ

* To delete your cluster: `kops delete cluster useast1.dev.example.com --yes`



## {{% heading "whatsnext" %}}


* Learn more about Kubernetes [concepts](/docs/concepts/) and [`kubectl`](/ja/docs/reference/kubectl/).
* Learn more about `kops` [advanced usage](https://kops.sigs.k8s.io/) for tutorials, best practices and advanced configuration options.
* Follow `kops` community discussions on Slack: [community discussions](https://github.com/kubernetes/kops#other-ways-to-communicate-with-the-contributors)
* Contribute to `kops` by addressing or raising an issue [GitHub Issues](https://github.com/kubernetes/kops/issues)
