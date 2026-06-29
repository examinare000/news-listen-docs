# ADR-005: Star/Dismiss を契機にジョブを自動起動する

**ステータス:** 採用済み
**決定日:** 2026-06-13
**対象読者:** バックエンド開発者・インフラ担当

## 背景

これまで recommendation（スコア再計算）と podcast-generator（Podcast 生成）は、Cloud Scheduler による定時実行か `docker compose run` / `gcloud run jobs execute` による手動実行でしか起動できなかった。ユーザーが記事をスター／dismiss しても、その操作が反映されるのは次の定時実行までずれ込む。スター直後に Podcast を聴きたい、dismiss を即座にレコメンドへ反映したい、という要求に応えられない。

これらのジョブは重い。recommendation は最大 200 件を Gemini でスコアリングし、podcast-generator は Gemini スクリプト生成 + TTS で 1 件あたり数分かかる。api（Cloud Run サービス）はリクエストタイムアウト 300s かつレスポンス送出後の CPU 割り当てが保証されないため、リクエスト内でジョブ本体を完結させられない。

## 検討した選択肢

| 選択肢 | メリット | デメリット |
|---|---|---|
| FastAPI in-process BackgroundTasks でジョブ本体を実行 | 実装が単純・追加インフラ不要 | TTS の数分処理を api プロセスで走らせると Cloud Run の CPU 打ち切り（min-instances=0）で不安定。api イメージに Gemini 鍵・メモリ増が必要 |
| **Cloud Run Jobs Admin API（`jobs:run`）を api から起動** | 本番は Cloud Scheduler と同一経路。ジョブは専用コンテナ（1Gi / 長 timeout）で非同期実行され api と分離 | SA に `run.jobs.run` 権限が必要。ローカル docker compose には Cloud Run Jobs が無い |

## 決定

api の `POST /articles/{id}/star` / `/dismiss` ハンドラから、FastAPI `BackgroundTasks`（レスポンス送出後に実行）でジョブを fire-and-forget 起動する。

- **star**: 正のシグナル → `recommendation` + `podcast-generator` を起動
- **dismiss**: 負のシグナル → `recommendation` のみ起動

起動経路は `shared/job_trigger.py` で抽象化し、環境変数 `JOB_TRIGGER_BACKEND` で切り替える。

- `cloud_run`（本番）: SA 認証で `https://{region}-run.googleapis.com/.../jobs/{name}:run` を POST。
- `local_process`（ローカル docker compose）: api コンテナ内で `python -m jobs.*.main` をサブプロセス起動。Cloud Run を介さず動かすため、api イメージに `jobs/` と Gemini 鍵・`DIFFICULTY` を同梱する。
- 未設定/不正: `NoOpJobTrigger`（自動起動を無効化）。

**多重実行ガード（debounce）**: 連打や短時間の連続操作でジョブが積み上がり Gemini コストが膨らむのを防ぐため、Firestore の `jobLocks/{user_id}_{job_name}` ドキュメントで TTL 付きロックを原子的に取得する（recommendation 120s / podcast-generator 600s）。ジョブ側は starred/dismissed の全件を都度走査するため、ウィンドウ内の連続操作を 1 回の実行へまとめても取りこぼさない。完了時の明示解放はせず TTL 失効で自然解放する。

## 理由

ジョブを api と別コンテナに分離する Cloud Run Jobs 経路が、長時間・高メモリ処理を本番で安定実行できる唯一の案であり、既存の Cloud Scheduler 起動経路とも一致するため。ローカルはサブプロセス起動でユーザー要求（ローカルでも自動起動）を満たす。

## 結果（影響）

- ジョブ起動失敗は握りつぶしてログのみ記録し、star/dismiss の HTTP 応答（200）には影響させない（ベストエフォート）。
- SA に `roles/run.developer`（`run.jobs.run` を含む）を付与する必要がある（`infra/setup.sh`）。これは Cloud Scheduler 経路の前提権限も兼ねる。
- api サービスに `JOB_TRIGGER_BACKEND=cloud_run` / `GCP_REGION` / `GCP_PROJECT_ID` を設定する（`infra/deploy.sh`）。
- `docker compose run` / `gcloud run jobs execute` による手動・定時起動は従来どおり併用可能。
- debounce ウィンドウ内に発生したスターは、進行中ジョブが拾えなければ次回起動まで反映が遅れる。Podcast 生成は冪等（既存はスキップ）のため実害は小さい。

## 追記（2026-06-29）

backend#56（commit `22cbdbf`）で Cloud Run Jobs API が **v1 `jobs:run` から v2 `RunJob`** へ移行し、実行ごとの `USER_ID` 伝播方式が変更された。

- **v1（本 ADR 決定時点）**: `POST https://{region}-run.googleapis.com/.../jobs/{name}:run` に `env` ペイロード
- **v2（backend#56 現状）**: `POST https://{region}-run.googleapis.com/.../jobs/{name}:runJob` で `overrides.containerOverrides[].env` を指定

本文の「API 呼び出し」記述は v1 を前提としているが、**backend の実装は既に v2 で更新済み**（`shared/job_trigger.py` が環境に応じ `RunJob` リクエストを組み立てる）。ローカル `docker compose` では親環境の `USER_ID` を継承し、上書きで per-user 値を伝播させる点は不変。
