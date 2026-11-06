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
