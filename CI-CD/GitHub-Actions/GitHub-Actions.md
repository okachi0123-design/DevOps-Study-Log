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

## CI制作ログ

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
- `name: Run-Pytest` でテストを実行するディレクトリを指定しなかったことで、GitHub Actionsが、テスト実行時のmain.pyのインポートを検索することが出来ず、エラーが起きた
```
      - name: Run-Pytest #Pytest実行
        run: pytest web/vps-web/backend/tests
        working-directory: web/vps-web/backend
```
- `working-directory: web/vps-web/backend`を追加して、認識させるように指定
```
run: pytest web/vps-web/backend/tests
working-directory: web/vps-web/backend
```
にしていたことで、pytestがworking-directoryからさらにweb/vps-web/backend/testsまで潜って実行をしようとしてエラーが起きた
```
- name: Run pytest
  working-directory: web/vps-web/backend
  run: python -m pytest tests/ -v
```
に変更して、working-directoryで実行されるように変更した
- テスト後　再度エラー　`sqlalchemy.exc.ArgumentError:Expected string or URL object, got None`
- main.py内でDB接続などを行う関数を実行するようにしているが、githubの方には.envを導入していないので、main.pyのimport時にそれらを読み取る関数が動かずにエラーが起きた
- runs-on: ubuntu-latestの下に↓を追加
```
env:
  DATABASE_URL: "db_url"
  FRONTEND_URL: "frontend_url"
```
- 再度エラー　`Could not parse SQLAlchemy URL from given URL string`
- URLの形式が正しくないので、SQLAlchemyが解釈できない
- GitHub ActionsでPostgreSQLを起動するサービスコンテナを利用する
- DB接続URLの構成`DATABASE_URL=postgresql://user:password@host:port/db名`
- サービスコンテナの設定
```
services: 
  postgres: #サービス名
    image: postgres:16 #Postgresイメージ
    env: #環境設定
      POSTGRES_USER: testuser 
      POSTGRES_PASSWORD: testdbpass
      POSTGRES_DB: testdb
    ports:
      - 5432:5432 #ランナーとDBコンテナのポートの接続
    options: >- Dockerコンテナ起動時のオプション
      --health-cmd "pg_isready -U testuser -d testdb" #ヘルスチェック
      --health-interval 10s #インターバル
      --health-timeout 5s #タイムアウト
      --health-retries 5 #５回失敗で以上判定
```
- envに渡す値を↑のDB接続に合うようにする
  `DATABASE_URL: "postgresql://testuser:testdbpass@localhost:5432/testdb"`
- ほかのenvの値も追加する
```
      AUTH_CLIENT_ID: "test_client_id"
      AUTH_CLIENT_SECRET: "test_client_secret"
      SECRET_SESSION_KEY: "test_session_key"
```
- push CI成功

---
## CD制作ログ
- 参考：[GitHub ActionsからUbuntuサーバーへデプロイする方法（LinuxLab）](https://linuxlab.jp/2026/07/10/ubuntu-github-actions-deploy/)
- ssh設定
- ssh鍵生成 ` ssh-keygen -t ed25519 -C "コメント" -f .ssh/鍵ファイル名 -N ""`-Nはパスワードなしオプション
- 公開鍵登録　`authorized_keys`に追記
- 秘密鍵、ユーザー名、ホストorIPアドレスをgithubactionsのsecretに登録
```
・リポジトリのSettings→Secrets and variables→Repository secretsのNew repository secret→シークレット名（VPS_SSH_KEY）,秘密鍵内容を登録
・同様にホスト(VPS_HOST)にIPアドレスを、ユーザー(VPS_USER)にユーザー名を登録
```
- `.github/workflows/deploy.yml`にCD用ワークフローYAMLを作る
```
on:
  push:
    branches: ["main"]

on:
  pull_request:
    branches: ["main"]

jobs:
  deploy:
  runs_on: ubuntu-latest
  steps:
    - name: Checkout-repository
      uses: actions/checkout@v7

    - name: Setup SSH agent #秘密鍵のssh-agentへの登録
      uses: webfactory/ssh-agent@v0.9.0
      with:
        ssh-private-key: ${{ secrets.VPS_SSH_KEY }}

    - name: Add Host to known_hosts #IPアドレスからVPSのホスト公開鍵を取得してknown_hostsに登録
      run:
        ssh-keyscan -H ${{ secrets.VPS_HOST }} >> ~/.ssh/known_hosts

    - name: Deploy via rsync #rsyncによるファイルの差分転送でデプロイ
      run: | #|で複数行のコマンドと指定 
        rsync -avzn \　　一回目nオプションでドライランでテスト　--deleteはファイルを消すのでドライラン後
          --exclude='.env' \　除外
          --exclude='venv' \
          --exclude='__pycache__/' \
          --exclude='webserver.sh' \
          --exclude='venv.sh' \
          ~/English_vocab_quiz/web/vps-web/backend \
          ${{ secrets.VPS_USER }}@${{ secrets.VPS_HOST }}:/opt/webquiz
   
    - name: Restart services
      run: |
        ssh ${{ secrets.VPS_USER }}@${{ secrets.VPS_HOST }} \
          "sudo -n systemctl restart webquiz.service && echo 'Restarted'" 　SSH接続をしてsystemdサービスの再起動 -nで非対話にしてパスワード要求時は自動で失敗するように
```
- SSHでのサービス再起動時にsudoパスワードが求められるので、`visudo`で設定を変更
```
sudo visudo -f /etc/sudoers.d/webquiz-deployでCD用のsudo設定ファイルを編集
ユーザー名 ホスト=(実行ユーザー) NOPASSWD: /usr/bin/systemctl restart webquiz.service　でsystemdサービス再起動コマンドだけにNOPASSWDを許可
```
‐ フロントエンドのデプロイも追加
‐ github上でビルド実行
```
    - name: Build frontend
      run: |
        cd web/vps_web/frontend &&/
        npm ci $$/
        npm run build
```
- run: |では、改行ごとにコマンドを記述できるので&&不要
```
    - name: Build frontend
      run: |
        cd web/vps_web/frontend
        npm ci
        npm run build
```
- cdは別のstepsには引き継がれない
- `dist/`の転送
```
    - name: Deploy frontend via rsync #フロントエンド差分転送
      run: |
        rsync -avzn \
          web/vps_web/frontend/dist/ \
          ${{ secrets.VPS_USER }}@${{ secrets.VPS_HOST }}:/var/www/webquiz/
```
- `Add Host to known_hosts`をホストから公開鍵を取得する方法じゃなくて、シークレットにあらかじめホスト公開鍵を登録する方式に変更　参考：[OpenSSH公式マニュアル — SSH_KNOWN_HOSTS FILE FORMAT](https://man.openbsd.org/sshd.8#SSH_KNOWN_HOSTS_FILE_FORMAT)
- シークレット名:`VPS_HOST_KEY`　内容：（マーカー） IPアドレス 鍵方式 鍵文　を半角スペース区切りで書く
- 鍵方式、鍵文は` cat /etc/ssh/ssh_host_ed25519_key.pub`をVPS上で実行
- `.ssh`を作成する工程も追加
```
      - name: Add Host to known_hosts #VPSのホスト公開鍵をknown_hostsに登録
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.VPS_HOST_KEY }}" >> ~/.ssh/known_hosts
```
## 参考資料
- [GitHub Actionsを使ってCI/CDを実装！（YouTube）](https://www.youtube.com/watch?v=RxcUrg3OO3o&t=222s)
- [GitHub公式ドキュメント：ワークフローの構文](https://docs.github.com/ja/actions/reference/workflows-and-actions/workflow-syntax)
- [GitHub Actions で FastAPI + PostgreSQL テストを自動実行する（Qiita）](https://qiita.com/takuabon/items/e51de1ff4e5d9a29bb6b)
- [PostgreSQLサービスコンテナの作成（GitHub Docs）](https://docs.github.com/ja/actions/tutorials/use-containerized-services/create-postgresql-service-containers)
- [GitHub ActionsからUbuntuサーバーへデプロイする方法（LinuxLab）](https://linuxlab.jp/2026/07/10/ubuntu-github-actions-deploy/)
- [OpenSSH公式マニュアル — SSH_KNOWN_HOSTS FILE FORMAT](https://man.openbsd.org/sshd.8#SSH_KNOWN_HOSTS_FILE_FORMAT)
