# Sketio Architecture (GitHub Action)

Reads the architecture of the checked-out repository (Cloudflare Workers, bindings, routes, CloudFormation, CDK,
Terraform, TypeScript / Hono) and writes it as an Architecture Artifact (JSON). With `mode: sync` or `mode: diff` it also
sends the Artifact to [Sketio Arch](https://arch.sketio.app).

日本語は[下](#日本語)にあります。

```yaml
name: architecture
on:
  push: { branches: [main] }
  pull_request:
jobs:
  arch:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2   # mode: diff reads the pull request's head commit (a parent of the merge commit)
      - uses: sketio-app/action@0abcd451c3f8b6e70854da40827bca353e6525a3 # v1.9.0
        with:
          mode: ${{ github.event_name == 'pull_request' && 'diff' || 'sync' }}
          api-url: https://arch.sketio.app
```

Pin the Action to a commit, as above. A tag can be moved, so pinning to the commit means only that code runs. Dependabot (`package-ecosystem: github-actions`) can raise the commit and the version in the comment for you. For a quick try, `sketio-app/action@v1` (the moving major tag) also works.

`analyze` needs no permissions and sends nothing:

```yaml
      - uses: actions/checkout@v4
      - uses: sketio-app/action@0abcd451c3f8b6e70854da40827bca353e6525a3 # v1.9.0
```

## Modes

| mode | runs on | what it does |
| --- | --- | --- |
| `analyze` | anything | Reads the repository, writes the Artifact to `$RUNNER_TEMP/sketio-architecture.json` and the job summary. Sends nothing. |
| `sync` | `push`, `workflow_dispatch`, `schedule` | Analyzes, then sends the Artifact to Arch. Arch stores a Snapshot. Use it on the default branch so pull requests have something to compare with. |
| `diff` | `pull_request` | Analyzes, sends the Artifact, and writes the Architecture Diff Arch returns (added / removed / modified Components and Relations) to the job summary. |

`sync` and `diff` need:

- `api-url`: the base URL of Arch, `https://arch.sketio.app`. https only (http is accepted for `localhost`). The
  **origin** of this URL is the OIDC audience, so a token minted for one Arch cannot be used on another.
- `permissions: id-token: write` on the job. The Action signs in with GitHub's OIDC token; there is no `token`
  input and no secret to store.
- The Sketio GitHub App ("Sketio Arch") installed on the repository.

## Inputs and outputs

| input | default | |
| --- | --- | --- |
| `mode` | `analyze` | `analyze`, `sync`, `diff` |
| `config` | `.sketio.yml` | path inside the repository |
| `api-url` | | Arch base URL. Required by `sync` and `diff` |
| `diagram-view` | `infrastructure` | the view the `mermaid` output draws: `infrastructure`, `application`, `deployment`, `overview` |

Outputs: `artifact-path`, `status`, `mermaid`; for `sync` / `diff` also `outcome` (`created`, `duplicate`, `recorded`),
`snapshot-id`, `inference-snapshot-id` and (diff) `diff-id`. The step exits 1 when the analysis failed or Arch
rejected the request; `partial` exits 0 with a warning.

### The diagram as Mermaid

The `mermaid` output is the diagram of one view as Mermaid text (several lines, without the code fence),
made on the runner from the Artifact in every mode, so `analyze` and fork pull requests get it too. `diagram-view`
picks the view (`infrastructure` by default, or `application`, `deployment`, `overview`). It is the same text
as Sketio Arch's "Copy Mermaid" and the MCP tool `get_diagram`, without the labels and groups set in Arch, and
it is drawn as the Arch screen draws it: each box its name and the official service name, the stacks, regions
and VPCs as nested subgraphs, a round mark for the users, flows and bindings as arrows (a binding says its names
and permissions), dependencies as plain lines, inferred relations dotted (`-.->`) and inferred components marked
`(inferred)`. In English.
Names from the repository are written so that they cannot end a label or close a code block. It is empty when
the analysis has no graph.

The Action only gives the text. Writing it to a file, committing it or opening a pull request is up to the later
steps of your workflow, and what they write is yours to check. An example that keeps a part of README.md up to
date (the commit is left out):

```yaml
      - uses: actions/checkout@v4
      - id: arch
        uses: sketio-app/action@0abcd451c3f8b6e70854da40827bca353e6525a3 # v1.9.0
        with:
          diagram-view: infrastructure   # or application, deployment, overview
      # Your own step. This one replaces what is between two markers in README.md:
      #   <!-- architecture:start -->
      #   <!-- architecture:end -->
      - name: Put the diagram in README.md
        env:
          MERMAID: ${{ steps.arch.outputs.mermaid }}   # through env: never put the expression inside the script
        run: |
          node --input-type=module -e '
            import { readFileSync, writeFileSync } from "node:fs"
            const start = "<!-- architecture:start -->"
            const end = "<!-- architecture:end -->"
            const text = readFileSync("README.md", "utf8")
            const from = text.indexOf(start)
            const to = text.indexOf(end)
            if (from < 0 || to < from) throw new Error("the markers are not in README.md")
            const block = "\n```mermaid\n" + process.env.MERMAID + "\n```\n"
            writeFileSync("README.md", text.slice(0, from + start.length) + block + text.slice(to))
          '
```

`mermaid` and `diagram-view` are available from v1.9.0 (the pin above).

## Notes

- **Fork pull requests** are analyzed and never sent: the Action does not even ask for an OIDC token.
  `pull_request_target` is not accepted by `sync` or `diff`.
- **The Action never writes to the pull request.** For `diff`, Arch keeps one comment and a Check Run named
  `Sketio Architecture` up to date using the GitHub App's own installation token.
- **The pull request comment** is written in English with a Mermaid diagram of the change (the changed elements and
  both ends of the changed relations) unless `.sketio.yml` says otherwise. `mode: diff` sends these settings to Arch;
  the Check Run summary follows them too:

  ```yaml
  comment:
    language: ja   # ja or en (default: en)
    diagram: true  # add the diagram (default: true); false leaves it out
  ```

  With more than 30 changed elements the diagram is left out and the comment links to Sketio Arch instead.
- **`fetch-depth`**: Arch orders Snapshots by commit date. `sync` uses `GITHUB_SHA` (the default checkout is
  enough). `diff` uses the pull request's head commit, so set `fetch-depth: 2`.
- **AI completion is off by default.** It only runs when `.sketio.yml` says `ai: { enabled: true }`; then the
  masked text of some files the static parsers could not read is sent to Arch.
- **Runners**: Node 24 (`runs.using: node24`), provided by GitHub. `dist/` is self-contained (no
  `node_modules`); the native addon of the TypeScript parser ships next to it for `linux-x64-gnu`,
  `linux-arm64-gnu`, `darwin-arm64` and `darwin-x64`. Windows runners and musl (Alpine) containers are not
  supported.
- **Retries**: a 5xx answer is retried (3 attempts); 4xx answers and redirects are not.

## Versions

Use the moving major tag `@v1`, or pin an exact release (`@v1.0.0`) or a commit SHA.

This repository is a built copy: `dist/` is generated from the Sketio monorepo and replaced on every release. Please
do not send pull requests here.

---

## 日本語

チェックアウトしたリポジトリの構成（Cloudflare Workers、binding、ルート、CloudFormation、CDK、Terraform、TypeScript / Hono）を読み、
Architecture Artifact（JSON）として書き出す Action です。`mode: sync` / `mode: diff` では Artifact を
[Sketio Arch](https://arch.sketio.app) に送ります。

使い方は上の例のとおりです（`uses: sketio-app/action@0abcd451c3f8b6e70854da40827bca353e6525a3 # v1.9.0`）。タグは動かせるので、コミットハッシュで固定すると、同じコードしか動きません。Dependabot（`package-ecosystem: github-actions`）でハッシュとコメントの版を自動で上げられます。手軽に試すだけなら `sketio-app/action@v1`（動くメジャータグ）でも動きます。

| mode | 動く場面 | すること |
| --- | --- | --- |
| `analyze` | どこでも | 読んで Artifact を `$RUNNER_TEMP/sketio-architecture.json` とジョブの要約に書く。何も送らない |
| `sync` | `push` / `workflow_dispatch` / `schedule` | 読んだ結果を Arch に送り、Snapshot として保存させる。PR の比較元になるよう、既定ブランチで使う |
| `diff` | `pull_request` | 読んだ結果を送り、Arch が返す差分（追加・削除・変更）をジョブの要約に書く |

`sync` と `diff` に必要なもの:

- `api-url`: Arch の URL（`https://arch.sketio.app`）。https のみ（`localhost` は http も可）。この URL の origin が OIDC の
  audience になるので、ほかの Arch に使い回せません
- ジョブの `permissions: id-token: write`。GitHub の OIDC トークンでサインインするので、`token` 入力も保存する秘密もありません
- リポジトリに Sketio の GitHub App（Sketio Arch）を入れておくこと

補足:

- fork からの PR は読むだけで、何も送りません（OIDC トークンも要求しません）
- Action は PR に書き込みません。`diff` では Arch が GitHub App の権限でコメントと Check Run `Sketio Architecture` を更新します
- PR のコメントは、既定では英語で、変わった要素と変わった関係の両端を描いた Mermaid の図が付きます。`.sketio.yml` で変えられます
  （`mode: diff` が Arch に伝え、Check Run の要約も同じ設定になります）

  ```yaml
  comment:
    language: ja   # ja か en（既定は en）
    diagram: true  # 図を付ける（既定は true）。false で図なし
  ```

  変わった要素が 30 を超えるときは図を省き、Sketio Arch で開くリンクだけにします
- `mode: diff` では PR の head コミットの日付を使うので、`actions/checkout` に `fetch-depth: 2` を付けてください
- AI による補完は既定で無効です。`.sketio.yml` に `ai: { enabled: true }` と書いたときだけ、静的な解析で読めなかったファイルの
  （秘密をマスクした）テキストを Arch に送ります
- 実行環境は Node 24（GitHub が用意）。`dist/` だけで動き、`node_modules` は要りません。対応は `linux-x64-gnu`、`linux-arm64-gnu`、
  `darwin-arm64`、`darwin-x64`。Windows ランナーと Alpine（musl）のコンテナは非対応です
- `@v1`（移動タグ）のほか、`@v1.0.0` やコミット SHA での固定もできます

### Mermaid の構成図

出力 `mermaid` は、1 つの表示の構成図を Mermaid のテキスト（複数行、コードブロックの柵なし）にしたものです。
ランナーの上で Artifact から作るので、`analyze` でも fork の PR でも出ます。表示は `diagram-view` で選びます
（既定は `infrastructure`。ほかに `application`、`deployment`、`overview`）。Sketio Arch の「Mermaid をコピー」や
MCP の `get_diagram` と同じテキストで、Arch で付けたラベルとグループは入りません。描き方は Arch の画面と同じです。
箱は名前と正式なサービス名で、スタック・リージョン・VPC は入れ子の枠になり、利用者は丸い印で出ます。流れと binding は
矢印（binding には名前と権限）、付属の関係は矢印のない線、推測は点線（`-.->`）で、推測の要素には `(inferred)` が付きます。
文字は英語です。リポジトリから来た名前はラベルやコードブロックを
閉じられない形で書きます。解析にグラフが無いときは空です。

Action はテキストを出すだけです。ファイルに書く、コミットする、PR を作るのは、ワークフローの後のステップで
行ってください（書いた中身の確認も利用者の責任です）。README.md の決めた区間を書き換える例（コミットは省いています）:

```yaml
      - uses: actions/checkout@v4
      - id: arch
        uses: sketio-app/action@0abcd451c3f8b6e70854da40827bca353e6525a3 # v1.9.0
        with:
          diagram-view: infrastructure   # or application, deployment, overview
      # Your own step. This one replaces what is between two markers in README.md:
      #   <!-- architecture:start -->
      #   <!-- architecture:end -->
      - name: Put the diagram in README.md
        env:
          MERMAID: ${{ steps.arch.outputs.mermaid }}   # through env: never put the expression inside the script
        run: |
          node --input-type=module -e '
            import { readFileSync, writeFileSync } from "node:fs"
            const start = "<!-- architecture:start -->"
            const end = "<!-- architecture:end -->"
            const text = readFileSync("README.md", "utf8")
            const from = text.indexOf(start)
            const to = text.indexOf(end)
            if (from < 0 || to < from) throw new Error("the markers are not in README.md")
            const block = "\n```mermaid\n" + process.env.MERMAID + "\n```\n"
            writeFileSync("README.md", text.slice(0, from + start.length) + block + text.slice(to))
          '
```

`mermaid` と `diagram-view` は v1.9.0（上の固定）から使えます。

このリポジトリは Sketio のモノレポから作った配布用のコピーです。リリースのたびに `dist/` ごと置き換わるので、ここへの
プルリクエストは受け付けていません。
