# GitHubActions学習ログ

## GitHubActionsとは
- GitHubのワークフロー自動化の仕組み
- PullRequestやPushイベントなどを契機に自動でテスト、デプロイをする仕組み

## 料金
- public:無料
- private:無料枠超えると有料

## 仕組み
- 一連の処理をまとめたワークフローを単位として、その中にトリガーやjobといった部品が組み込まれている
- トリガー：PullRequestやPushなど、ワークフローを起動するきっかけとなるイベント
- Job：実際にテストやデプロイを行うための複数のステップを設定し管理する実行単位
- Step:Jobを構成する具体的なコマンドやActionなどの部品

## 制作ログ

- リポジトリのルート直下に、GitHub用の設定ファイルのディレクトリである`.github`、その下にワークフロー用ディレクトリ`workflows`を作成する`~/English_vocab_quiz/.github/workflows`
- ↑の下のYAMLファイルに従ってGitHubがワークフローを動かしてくれるようになる
- `workflows`でVSCodeを開き、好きな名前でファイルを作る`workflows/web_pytest.yaml`
‐ YAMLファイルに書き込む 
・ワークフロー名指定

```yaml
name: Run-Web-Pytest #ワークフローの名前
```

・トリガー指定

```yaml
on: #トリガーを指定するよ
  push: #指定されたトリガー（push）
    branches: ["main"] #（フィルター）main branchのイベントに限定

  pull_request: #（pull request）
    branches: ["main"] #（フィルター）main branchのイベントに限定
```

・ジョブ指定

```yaml
jobs: #jobと指定
  test-backend: #ジョブ識別
    runs-on: ubuntu-latest #ジョブを動かす環境（ランナー）GitHub hosted runner←今回使用/Self hosted runner
```

・ステップ指定（jobsの１つ下のインシデント）

・ステップ１　リポジトリのチェック　理由：ランナーの環境にはテスト対象のソースコードがまだ入っていない状態だから

```yaml
    steps:
      - name: Checkout-repository #リポジトリのチェック
        uses: actions/checkout@v7 #チェックアウト用アクション

      - name: Setup-Python #Python環境のセットアップ（バージョン指定したい時）
        uses: actions/setup-python@v7 #actions/setup-python
        with:
          python-version: 3.12 #バージョン指定

      - name: Install-Dependencies #requirements.txtのインストール
        run: pip install -r web/vps-web/backend/requirements.txt #実行コード

      - name: Run-Pytest #Pytest実行
        run: pytest web/vps-web/backend/tests
```

## 参考資料
- [GitHub Actionsを使ってCI/CDを実装！（YouTube）](https://www.youtube.com/watch?v=RxcUrg3OO3o&t=222s)
- [GitHub公式ドキュメント：ワークフローの構文](https://docs.github.com/ja/actions/reference/workflows-and-actions/workflow-syntax)
