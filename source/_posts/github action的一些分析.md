---
title: github action workflow的一些分析
date: 2025-01-31 13:47:40
categories: 
- act
---
主要是分析一些比较熟悉的开源项目的github action。
# [etcd](https://github.com/etcd-io/etcd)
## codeql-analysis.yml
```yml
name: "CodeQL"
on:
  push:
    branches: [main, release-3.4, release-3.5, release-3.6]
  pull_request:
    # The branches below must be a subset of the branches above
    branches: [main]
  schedule: # 定时任务
    - cron: '20 14 * * 5'
permissions: read-all
jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write
    strategy: # 具体参考https://docs.github.com/zh/actions/writing-workflows/workflow-syntax-for-github-actions#jobsjob_idstrategy
      fail-fast: false
      matrix:
        # CodeQL supports [ 'cpp', 'csharp', 'go', 'java', 'javascript', 'python' ]
        # Learn more:
        # https://docs.github.com/en/free-pro-team@latest/github/finding-security-vulnerabilities-and-errors-in-your-code/configuring-code-scanning#changing-the-languages-that-are-analyzed
        language: ['go']
    steps:
      - name: Checkout repository
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      # Initializes the CodeQL tools for scanning.
      - name: Initialize CodeQL
        uses: github/codeql-action/init@b6a472f63d85b9c78a3ac5e89422239fc15e9b3c # v3.28.1
        with:
          # If you wish to specify custom queries, you can do so here or in a config file.
          # By default, queries listed here will override any specified in a config file.
          # Prefix the list here with "+" to use these queries and those in the config file.
          # queries: ./path/to/local/query, your-org/your-repo/queries@main
          languages: ${{ matrix.language }}
      # Autobuild attempts to build any compiled languages  (C/C++, C#, or Java).
      # If this step fails, then you should remove it and run the build manually (see below)
      - name: Autobuild
        uses: github/codeql-action/autobuild@b6a472f63d85b9c78a3ac5e89422239fc15e9b3c # v3.28.1
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@b6a472f63d85b9c78a3ac5e89422239fc15e9b3c # v3.28.1
```
这里其实主要是对CodeQl的使用。
**CodeQL** 是一个由 GitHub 开发的 **语义代码分析引擎**，用于自动化地检测代码中的安全漏洞、编码错误和其他问题。它通过将代码转换为可查询的数据库（即 **CodeQL 数据库**），并允许开发者编写自定义查询来分析代码。

## contrib.yaml
```yml
---
name: Test contrib/mixin
on: [push, pull_request]
permissions: read-all
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - id: goversion
        run: echo "goversion=$(cat .go-version)" >> "$GITHUB_OUTPUT"
      - uses: actions/setup-go@3041bf56c941b39c61721a86cd11f3bb1338122a # v5.2.0
        with:
          go-version: ${{ steps.goversion.outputs.goversion }}
      - run: |
          set -euo pipefail

          make -C contrib/mixin tools test
```
这个就相对比较简单，就是在push和pr的时候make test。

## coverage.yaml
```yml
---
name: Coverage
on: [push, pull_request]
permissions: read-all
jobs:
  coverage:
    # this is to prevent the job to run at forked projects
    if: github.repository == 'etcd-io/etcd'
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        target:
          - linux-amd64-coverage
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - id: goversion
        run: echo "goversion=$(cat .go-version)" >> "$GITHUB_OUTPUT"
      - uses: actions/setup-go@3041bf56c941b39c61721a86cd11f3bb1338122a # v5.2.0
        with:
          go-version: ${{ steps.goversion.outputs.goversion }}
      - env:
          TARGET: ${{ matrix.target }}
        run: |
          mkdir "${TARGET}"
          case "${TARGET}" in
            linux-amd64-coverage)
              GOARCH=amd64 ./scripts/codecov_upload.sh
              ;;
            *)
              echo "Failed to find target"
              exit 1
              ;;
          esac
```
这里主要就是运行一个脚本执行测试`./scripts/codecov_upload.sh`。


## fuzzing.yaml
```yml
---
name: Fuzzing v3rpc
on: [push, pull_request]
permissions: read-all
jobs:
  fuzzing:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
    env:
      TARGET_PATH: ./server/etcdserver/api/v3rpc
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - id: goversion
        run: echo "goversion=$(cat .go-version)" >> "$GITHUB_OUTPUT"
      - uses: actions/setup-go@3041bf56c941b39c61721a86cd11f3bb1338122a # v5.2.0
        with:
          go-version: ${{ steps.goversion.outputs.goversion }}
      - run: |
          set -euo pipefail

          GOARCH=amd64 CPU=4 make fuzz
      - uses: actions/upload-artifact@65c4c4a1ddee5b72f698fdd19549f0f0fb45cf08 # v4.6.0
        if: failure()
        with:
          path: "${{env.TARGET_PATH}}/testdata/fuzz/**/*"
```
这里主要跑的是模糊测试，最后用了一个actions上传测试结果。

## gh-workflow-approve.yaml
```yml
---
name: Approve GitHub Workflows
permissions: read-all

on:
  pull_request_target:
    types:
      - labeled
      - synchronize
    branches:
      - main
      - release-3.5
      - release-3.4

jobs:
  approve:
    name: Approve ok-to-test
    if: contains(github.event.pull_request.labels.*.name, 'ok-to-test')
    runs-on: ubuntu-latest
    permissions:
      actions: write
    steps:
      - name: Update PR
        uses: actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea # v7.0.1
        continue-on-error: true
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          debug: ${{ secrets.ACTIONS_RUNNER_DEBUG == 'true' }}
          script: |
            const result = await github.rest.actions.listWorkflowRunsForRepo({
              owner: context.repo.owner,
              repo: context.repo.repo,
              event: "pull_request",
              status: "action_required",
              head_sha: context.payload.pull_request.head.sha,
              per_page: 100
            });

            for (var run of result.data.workflow_runs) {
              await github.rest.actions.approveWorkflowRun({
                owner: context.repo.owner,
                repo: context.repo.repo,
                run_id: run.id
              });
            }
```
这个代码主要是在pr被添加标签(labeled)和提交(synchronize)时候触发。其主要功能是通过 `actions/github-script`来执行脚本检查是否有需要批准的操作，并自动批准这些操作。
`pull_request` 事件*
- `opened`：当 PR 被创建时触发。
- `reopened`：当 PR 被重新打开时触发。
- `closed`：当 PR 被关闭时触发。
- `labeled`：当 PR 被添加标签时触发。
- `unlabeled`：当 PR 的标签被移除时触发。
- `synchronize`：当 PR 的代码被更新（如推送新提交）时触发。
- `review_requested`：当 PR 被请求审查时触发。
## grpcproxy.yaml
```yml
---
name: grpcProxy-tests
on: [push, pull_request]
permissions: read-all
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        target:
          - linux-amd64-grpcproxy-integration
          - linux-amd64-grpcproxy-e2e
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - id: goversion
        run: echo "goversion=$(cat .go-version)" >> "$GITHUB_OUTPUT"
      - uses: actions/setup-go@3041bf56c941b39c61721a86cd11f3bb1338122a # v5.2.0
        with:
          go-version: ${{ steps.goversion.outputs.goversion }}
      - env:
          TARGET: ${{ matrix.target }}
        run: |
          set -euo pipefail

          echo "${TARGET}"
          case "${TARGET}" in
            linux-amd64-grpcproxy-integration)
              GOOS=linux GOARCH=amd64 CPU=4 make test-grpcproxy-integration
              ;;
            linux-amd64-grpcproxy-e2e)
              GOOS=linux GOARCH=amd64 CPU=4 make test-grpcproxy-e2e
              ;;
            *)
              echo "Failed to find target"
              exit 1
              ;;
          esac
```
主要是跑grpc proxy的集成测试和e2e测试。和前面的coverage的yml相似。通过设置target来运行两个任务。


## measure-testgrid-flakiness.yaml
```yaml
---
name: Measure TestGrid Flakiness

on:
  schedule:
    - cron: "0 0 * * *" # run every day at midnight

permissions: read-all

jobs:
  measure-testgrid-flakiness:
    name: Measure TestGrid Flakiness
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - id: goversion
        run: echo "goversion=$(cat .go-version)" >> "$GITHUB_OUTPUT"
      - uses: actions/setup-go@3041bf56c941b39c61721a86cd11f3bb1338122a # v5.2.0
        with:
          go-version: ${{ steps.goversion.outputs.goversion }}
      - env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          set -euo pipefail

          ./scripts/measure-testgrid-flakiness.sh
```
- 此处主要是执行一个相关测试。
- GITHUB_TOKEN是一个环境变量注入，提供给脚本使用。由github提供内置的默认令牌。当然你可以查看[官方文档](https://docs.github.com/zh/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions?tool=webui)来看看如何创建secret。


## release.yaml
```yaml
---
name: Release
on: [push, pull_request]
permissions: read-all
jobs:
  main:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - id: goversion
        run: echo "goversion=$(cat .go-version)" >> "$GITHUB_OUTPUT"
      - uses: actions/setup-go@3041bf56c941b39c61721a86cd11f3bb1338122a # v5.2.0
        with:
          go-version: ${{ steps.goversion.outputs.goversion }}
      - name: release
        run: |
          set -euo pipefail

          git config --global user.email "github-action@etcd.io"
          git config --global user.name "Github Action"
          gpg --batch --gen-key <<EOF
          %no-protection
          Key-Type: 1
          Key-Length: 2048
          Subkey-Type: 1
          Subkey-Length: 2048
          Name-Real: Github Action
          Name-Email: github-action@etcd.io
          Expire-Date: 0
          EOF
          DRY_RUN=true ./scripts/release.sh --no-upload --no-docker-push --no-gh-release --in-place 3.6.99
      - name: test-image
        run: |
          VERSION=3.6.99 ./scripts/test_images.sh
      - name: save-image
        run: |
          docker image save -o /tmp/etcd-img.tar gcr.io/etcd-development/etcd
      - name: upload-image
        uses: actions/upload-artifact@65c4c4a1ddee5b72f698fdd19549f0f0fb45cf08 # v4.6.0
        with:
          name: etcd-img
          path: /tmp/etcd-img.tar
          retention-days: 1
  trivy-scan:
    needs: main
    strategy:
      fail-fast: false
      matrix:
        platforms: [amd64, arm64, ppc64le, s390x]
    permissions:
      security-events: write # for github/codeql-action/upload-sarif to upload SARIF results
    runs-on: ubuntu-latest
    steps:
      - name: get-image
        uses: actions/download-artifact@fa0a91b85d4f404e444e00e005971372dc801d16 # v4.1.8
        with:
          name: etcd-img
          path: /tmp
      - name: load-image
        run: |
          docker load < /tmp/etcd-img.tar
      - name: trivy-scan
        uses: aquasecurity/trivy-action@18f2510ee396bbf400402947b394f2dd8c87dbb0 # v0.29.0
        with:
          image-ref: 'gcr.io/etcd-development/etcd:v3.6.99-${{ matrix.platforms }}'
          severity: 'CRITICAL,HIGH'
          format: 'sarif'
          output: 'trivy-results-${{ matrix.platforms }}.sarif'
        env:
          # Use AWS' ECR mirror for the trivy-db image, as GitHub's Container
          # Registry is returning a TOOMANYREQUESTS error.
          # Ref: https://github.com/aquasecurity/trivy-action/issues/389
          TRIVY_DB_REPOSITORY: 'public.ecr.aws/aquasecurity/trivy-db:2'
      - name: upload scan results
        uses: github/codeql-action/upload-sarif@b6a472f63d85b9c78a3ac5e89422239fc15e9b3c # v3.28.1
        with:
          sarif_file: 'trivy-results-${{ matrix.platforms }}.sarif'
```
- jobs主要执行了两个任务main和trivy-scan。
- main任务执行了模拟发布流程，然后再test image, save image, upload image。
- trivy-scan任务是扫描docker镜像的漏洞后，将结果上传至github的安全中心。
- 这里docker image是一个多平台镜像，加载后会将所有平台都加载到本地。这也就是为什么main只打包了一个镜像，而trivy-scan却可以测试四个镜像。

## scorecards.yml
```yaml
---
name: Scorecards supply-chain security
on:
  # Only the default branch is supported.
  branch_protection_rule:
  schedule:
    - cron: '45 1 * * 0'
  push:
    branches: ["main"]

# Declare default permissions as read only.
permissions: read-all

jobs:
  analysis:
    name: Scorecards analysis
    runs-on: ubuntu-latest
    permissions:
      # Needed to upload the results to code-scanning dashboard.
      security-events: write
      # Used to receive a badge.
      id-token: write

    steps:
      - name: "Checkout code"
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          persist-credentials: false

      - name: "Run analysis"
        uses: ossf/scorecard-action@62b2cac7ed8198b15735ed49ab1e5cf35480ba46 # v2.4.0
        with:
          results_file: results.sarif
          results_format: sarif

          # Publish the results for public repositories to enable scorecard badges. For more details, see
          # https://github.com/ossf/scorecard-action#publishing-results.
          # For private repositories, `publish_results` will automatically be set to `false`, regardless
          # of the value entered here.
          publish_results: true

      # Upload the results as artifacts (optional). Commenting out will disable uploads of run results in SARIF
      # format to the repository Actions tab.
      - name: "Upload artifact"
        uses: actions/upload-artifact@65c4c4a1ddee5b72f698fdd19549f0f0fb45cf08 # v4.6.0
        with:
          name: SARIF file
          path: results.sarif
          retention-days: 5

      # Upload the results to GitHub's code scanning dashboard.
      - name: "Upload to code-scanning"
        uses: github/codeql-action/upload-sarif@b6a472f63d85b9c78a3ac5e89422239fc15e9b3c # v3.28.1
        with:
          sarif_file: results.sarif
```
- `branch_protection_rule`在工作流程存储库中的分支保护规则发生更改时运行工作流程。
- 其后面主要就是对OpenSSF Scorecards的使用。首先是执行分析，后再将数据上传至`github的securtiy和制品库`。
- **工具介绍：OpenSSF Scorecards**
    - **OpenSSF Scorecards**是一个开源工具，用于评估开源项目的供应链安全性。
    - 它会对仓库的多个方面进行检查，例如：
        - 是否有安全策略（如`SECURITY.md`）。
        - 是否启用分支保护。
        - 是否有代码审查流程。
        - 是否有自动化测试。
        - 是否有依赖项更新机制。
        - 检查结果会生成一个分数，并指出需要改进的地方。


# [gitea](https://github.com/go-gitea/gitea)
## cron-licenses.yml
```yaml
name: cron-licenses

on:
  schedule:
    - cron: "7 0 * * 1" # every Monday at 00:07 UTC
  workflow_dispatch: # 支持手动触发

jobs:
  cron-licenses:
	    runs-on: ubuntu-latest
    if: github.repository == 'go-gitea/gitea'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5 # 比etcd的setup-go高级
        with:
          go-version-file: go.mod
          check-latest: true
      - run: make generate-license generate-gitignore
        timeout-minutes: 40
      - name: push translations to repo
        uses: appleboy/git-push-action@v0.0.3
        with:
          author_email: "teabot@gitea.io"
          author_name: GiteaBot
          branch: main
          commit: true
          commit_message: "[skip ci] Updated licenses and gitignores"
          remote: "git@github.com:go-gitea/gitea.git"
          ssh_key: ${{ secrets.DEPLOY_KEY }}
```
- 这段首先是一个定时任务。
- 其次是他的setup-go相较于etcd会高级一点，可以直接通过go.mod来获取go版本。
- 看起来make命令主要是执行生成`license`和`.gitignore`
- 最后再push到repo。
  第二个文件cron-translations.yml的逻辑与这个几乎一致，所以可以忽略。
##  files-changed.yml
```yaml
name: files-changed

on:
  workflow_call:
    outputs:
      backend:
        value: ${{ jobs.detect.outputs.backend }}
      frontend:
        value: ${{ jobs.detect.outputs.frontend }}
      docs:
        value: ${{ jobs.detect.outputs.docs }}
      actions:
        value: ${{ jobs.detect.outputs.actions }}
      templates:
        value: ${{ jobs.detect.outputs.templates }}
      docker:
        value: ${{ jobs.detect.outputs.docker }}
      swagger:
        value: ${{ jobs.detect.outputs.swagger }}
      yaml:
        value: ${{ jobs.detect.outputs.yaml }}

jobs:
  detect:
    runs-on: ubuntu-latest
    timeout-minutes: 3
    outputs:
      backend: ${{ steps.changes.outputs.backend }}
      frontend: ${{ steps.changes.outputs.frontend }}
      docs: ${{ steps.changes.outputs.docs }}
      actions: ${{ steps.changes.outputs.actions }}
      templates: ${{ steps.changes.outputs.templates }}
      docker: ${{ steps.changes.outputs.docker }}
      swagger: ${{ steps.changes.outputs.swagger }}
      yaml: ${{ steps.changes.outputs.yaml }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: changes
        with:
          filters: |
            backend:
              - "**/*.go"
              - "templates/**/*.tmpl"
              - "assets/emoji.json"
              - "go.mod"
              - "go.sum"
              - "Makefile"
              - ".golangci.yml"
              - ".editorconfig"
              - "options/locale/locale_en-US.ini"

            frontend:
              - "**/*.js"
              - "web_src/**"
              - "assets/emoji.json"
              - "package.json"
              - "package-lock.json"
              - "Makefile"
              - ".eslintrc.yaml"
              - "stylelint.config.js"
              - ".npmrc"

            docs:
              - "**/*.md"
              - ".markdownlint.yaml"
              - "package.json"
              - "package-lock.json"

            actions:
              - ".github/workflows/*"
              - "Makefile"

            templates:
              - "tools/lint-templates-*.js"
              - "templates/**/*.tmpl"
              - "pyproject.toml"
              - "poetry.lock"

            docker:
              - "Dockerfile"
              - "Dockerfile.rootless"
              - "docker/**"
              - "Makefile"

            swagger:
              - "templates/swagger/v1_json.tmpl"
              - "Makefile"
              - "package.json"
              - "package-lock.json"
              - ".spectral.yaml"

            yaml:
              - "**/*.yml"
              - "**/*.yaml"
              - ".yamllint.yaml"
              - "pyproject.toml"
              - "poetry.lock"
```
- 该job是一个workflow_call，也就是允许其他actions复用。
- 通过 `dorny/paths-filter`来检测文件是否发生变化并输出bool值
- 最后工作流将这些bool值作为输出来检测相关文件是否发生变化。
  该工作流是其他操作的辅助，可以通过该工作流来决定其他操作是否执行。

## pull-compliance.yml
```yaml
name: compliance

on:
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

jobs:
  files-changed:
    uses: ./.github/workflows/files-changed.yml

  lint-backend:
    if: needs.files-changed.outputs.backend == 'true' || needs.files-changed.outputs.actions == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - run: make deps-backend deps-tools
      - run: make lint-backend
        env:
          TAGS: bindata sqlite sqlite_unlock_notify

  lint-templates:
    if: needs.files-changed.outputs.templates == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: package-lock.json
      - run: pip install poetry
      - run: make deps-py
      - run: make deps-frontend
      - run: make lint-templates

  lint-yaml:
    if: needs.files-changed.outputs.yaml == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install poetry
      - run: make deps-py
      - run: make lint-yaml

  lint-swagger:
    if: needs.files-changed.outputs.swagger == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: package-lock.json
      - run: make deps-frontend
      - run: make lint-swagger

  lint-spell:
    if: needs.files-changed.outputs.backend == 'true' || needs.files-changed.outputs.frontend == 'true' || needs.files-changed.outputs.actions == 'true' || needs.files-changed.outputs.docs == 'true' || needs.files-changed.outputs.templates == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - run: make lint-spell

  lint-go-windows:
    if: needs.files-changed.outputs.backend == 'true' || needs.files-changed.outputs.actions == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - run: make deps-backend deps-tools
      - run: make lint-go-windows lint-go-vet
        env:
          TAGS: bindata sqlite sqlite_unlock_notify
          GOOS: windows
          GOARCH: amd64

  lint-go-gogit:
    if: needs.files-changed.outputs.backend == 'true' || needs.files-changed.outputs.actions == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - run: make deps-backend deps-tools
      - run: make lint-go
        env:
          TAGS: bindata gogit sqlite sqlite_unlock_notify

  checks-backend:
    if: needs.files-changed.outputs.backend == 'true' || needs.files-changed.outputs.actions == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - run: make deps-backend deps-tools
      - run: make --always-make checks-backend # ensure the "go-licenses" make target runs

  frontend:
    if: needs.files-changed.outputs.frontend == 'true' || needs.files-changed.outputs.actions == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: package-lock.json
      - run: make deps-frontend
      - run: make lint-frontend
      - run: make checks-frontend
      - run: make test-frontend
      - run: make frontend

  backend:
    if: needs.files-changed.outputs.backend == 'true' || needs.files-changed.outputs.actions == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      # no frontend build here as backend should be able to build
      # even without any frontend files
      - run: make deps-backend
      - run: go build -o gitea_no_gcc # test if build succeeds without the sqlite tag
      - name: build-backend-arm64
        run: make backend # test cross compile
        env:
          GOOS: linux
          GOARCH: arm64
          TAGS: bindata gogit
      - name: build-backend-windows
        run: go build -o gitea_windows
        env:
          GOOS: windows
          GOARCH: amd64
          TAGS: bindata gogit
      - name: build-backend-386
        run: go build -o gitea_linux_386 # test if compatible with 32 bit
        env:
          GOOS: linux
          GOARCH: 386

  docs:
    if: needs.files-changed.outputs.docs == 'true' || needs.files-changed.outputs.actions == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: package-lock.json
      - run: make deps-frontend
      - run: make lint-md

  actions:
    if: needs.files-changed.outputs.actions == 'true' || needs.files-changed.outputs.actions == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - run: make lint-actions
```
- 主要就是对pr的一些检查。
- 首先是利用了files-changed.yml，通过check相关模块是否发生变化来执行相关操作。
- 接着就是各种lint操作。
- 然后就是backend和frontend的各种build和test操作。
## pull-db-tests.yml
```yaml
name: db-tests

on:
  pull_request:

concurrency: #似乎与pull-compliance是同一个group。
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

jobs:
  files-changed:
    uses: ./.github/workflows/files-changed.yml

  test-pgsql:
    if: needs.files-changed.outputs.backend == 'true' || needs.files-changed.outputs.actions == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    services:
      pgsql:
        image: postgres:12
        env:
          POSTGRES_DB: test
          POSTGRES_PASSWORD: postgres
        ports:
          - "5432:5432"
      ldap:
        image: gitea/test-openldap:latest
        ports:
          - "389:389"
          - "636:636"
      minio:
        # as github actions doesn't support "entrypoint", we need to use a non-official image
        # that has a custom entrypoint set to "minio server /data"
        image: bitnami/minio:2023.8.31
        env:
          MINIO_ROOT_USER: 123456
          MINIO_ROOT_PASSWORD: 12345678
        ports:
          - "9000:9000"
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - name: Add hosts to /etc/hosts # 如果是容器网络，则可以直接通过容器名映射到对应ip地址，否则就需要将pgsql ldap minio解析为回环地址。
        run: '[ -e "/.dockerenv" ] || [ -e "/run/.containerenv" ] || echo "127.0.0.1 pgsql ldap minio" | sudo tee -a /etc/hosts'
      - run: make deps-backend
      - run: make backend
        env:
          TAGS: bindata
      - name: run migration tests
        run: make test-pgsql-migration
      - name: run tests
        run: make test-pgsql
        timeout-minutes: 50
        env:
          TAGS: bindata gogit
          RACE_ENABLED: true
          TEST_TAGS: gogit
          TEST_LDAP: 1
          USE_REPO_TEST_DIR: 1

  test-sqlite:
    if: needs.files-changed.outputs.backend == 'true' || needs.files-changed.outputs.actions == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - run: make deps-backend
      - run: make backend
        env:
          TAGS: bindata gogit sqlite sqlite_unlock_notify
      - name: run migration tests
        run: make test-sqlite-migration
      - name: run tests
        run: make test-sqlite
        timeout-minutes: 50
        env:
          TAGS: bindata gogit sqlite sqlite_unlock_notify
          RACE_ENABLED: true
          TEST_TAGS: gogit sqlite sqlite_unlock_notify
          USE_REPO_TEST_DIR: 1

  test-unit:
    if: needs.files-changed.outputs.backend == 'true' || needs.files-changed.outputs.actions == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    services:
      elasticsearch:
        image: elasticsearch:7.5.0
        env:
          discovery.type: single-node
        ports:
          - "9200:9200"
      meilisearch:
        image: getmeili/meilisearch:v1.2.0
        env:
          MEILI_ENV: development # disable auth
        ports:
          - "7700:7700"
      redis:
        image: redis
        options: >- # wait until redis has started
          --health-cmd "redis-cli ping"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 10
        ports:
          - 6379:6379
      minio:
        image: bitnami/minio:2021.3.17
        env:
          MINIO_ACCESS_KEY: 123456
          MINIO_SECRET_KEY: 12345678
        ports:
          - "9000:9000"
      devstoreaccount1.azurite.local: # https://github.com/Azure/Azurite/issues/1583
        image: mcr.microsoft.com/azure-storage/azurite:latest
        ports:
          - 10000:10000
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - name: Add hosts to /etc/hosts
        run: '[ -e "/.dockerenv" ] || [ -e "/run/.containerenv" ] || echo "127.0.0.1 minio devstoreaccount1.azurite.local mysql elasticsearch meilisearch smtpimap" | sudo tee -a /etc/hosts'
      - run: make deps-backend
      - run: make backend
        env:
          TAGS: bindata
      - name: unit-tests
        run: make unit-test-coverage test-check
        env:
          TAGS: bindata
          RACE_ENABLED: true
          GITHUB_READ_TOKEN: ${{ secrets.GITHUB_READ_TOKEN }}
      - name: unit-tests-gogit
        run: make unit-test-coverage test-check
        env:
          TAGS: bindata gogit
          RACE_ENABLED: true
          GITHUB_READ_TOKEN: ${{ secrets.GITHUB_READ_TOKEN }}

  test-mysql:
    if: needs.files-changed.outputs.backend == 'true' || needs.files-changed.outputs.actions == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    services:
      mysql:
        # the bitnami mysql image has more options than the official one, it's easier to customize
        image: bitnami/mysql:8.0
        env:
          ALLOW_EMPTY_PASSWORD: true
          MYSQL_DATABASE: testgitea
        ports:
          - "3306:3306"
        options: >-
          --mount type=tmpfs,destination=/bitnami/mysql/data
      elasticsearch:
        image: elasticsearch:7.5.0
        env:
          discovery.type: single-node
        ports:
          - "9200:9200"
      smtpimap:
        image: tabascoterrier/docker-imap-devel:latest
        ports:
          - "25:25"
          - "143:143"
          - "587:587"
          - "993:993"
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - name: Add hosts to /etc/hosts
        run: '[ -e "/.dockerenv" ] || [ -e "/run/.containerenv" ] || echo "127.0.0.1 mysql elasticsearch smtpimap" | sudo tee -a /etc/hosts'
      - run: make deps-backend
      - run: make backend
        env:
          TAGS: bindata
      - name: run migration tests
        run: make test-mysql-migration
      - name: run tests
        # run: make integration-test-coverage (at the moment, no coverage is really handled)
        run: make test-mysql
        env:
          TAGS: bindata
          RACE_ENABLED: true
          USE_REPO_TEST_DIR: 1
          TEST_INDEXER_CODE_ES_URL: "http://elastic:changeme@elasticsearch:9200"

  test-mssql:
    if: needs.files-changed.outputs.backend == 'true' || needs.files-changed.outputs.actions == 'true'
    needs: files-changed
    # specifying the version of ubuntu in use as mssql fails on newer kernels
    # pending resolution from vendor
    runs-on: ubuntu-20.04
    services:
      mssql:
        image: mcr.microsoft.com/mssql/server:2017-latest
        env:
          ACCEPT_EULA: Y
          MSSQL_PID: Standard
          SA_PASSWORD: MwantsaSecurePassword1
        ports:
          - "1433:1433"
      devstoreaccount1.azurite.local: # https://github.com/Azure/Azurite/issues/1583
        image: mcr.microsoft.com/azure-storage/azurite:latest
        ports:
          - 10000:10000
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - name: Add hosts to /etc/hosts
        run: '[ -e "/.dockerenv" ] || [ -e "/run/.containerenv" ] || echo "127.0.0.1 mssql devstoreaccount1.azurite.local" | sudo tee -a /etc/hosts'
      - run: make deps-backend
      - run: make backend
        env:
          TAGS: bindata
      - run: make test-mssql-migration
      - name: run tests
        run: make test-mssql
        timeout-minutes: 50
        env:
          TAGS: bindata
          USE_REPO_TEST_DIR: 1
```
- 该workflow主要是对各种db进行测试。
- 每个job基本都是`services`做好前置准备，再在`steps`来执行各种操作。主要是理解serveices如何使用。
- 在官方文档中提及了![](images/20250127162218.png)
  所以仅需要注意如果是容器网络，则可以直接通过容器名映射到对应ip地址，否则就需要容器开放端口。gitea的处理方案将容器名解析为127.0.0.1。也是一个不错的解决方案。


## pull-docker-dryrun.yml
```yml
name: docker-dryrun

on:
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

jobs:
  files-changed:
    uses: ./.github/workflows/files-changed.yml

  regular:
    if: needs.files-changed.outputs.docker == 'true' || needs.files-changed.outputs.actions == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    steps:
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v5
        with:
          push: false
          tags: gitea/gitea:linux-amd64

  rootless:
    if: needs.files-changed.outputs.docker == 'true' || needs.files-changed.outputs.actions == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    steps:
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v5
        with:
          push: false
          file: Dockerfile.rootless # 使用Dockerfile.rootless
          tags: gitea/gitea:linux-amd64
```
- 这里主要是在pr的时候打包两个镜像用来测试。
- docker/setup-buildx-action是用来多平台构建的，以适配多个平台。

## pull-e2e-tests.yml
```yaml
name: e2e-tests

on:
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

jobs:
  files-changed:
    uses: ./.github/workflows/files-changed.yml

  test-e2e:
    if: needs.files-changed.outputs.backend == 'true' || needs.files-changed.outputs.frontend == 'true' || needs.files-changed.outputs.actions == 'true'
    needs: files-changed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: package-lock.json
      - run: make deps-frontend frontend deps-backend
      - run: npx playwright install --with-deps
      - run: make test-e2e-sqlite
        timeout-minutes: 40
        env:
          USE_REPO_TEST_DIR: 1
```
- 这个文件主要是执行e2e测试，以下是一些gpt生成的简介。
  **E2E 测试（End-to-End Testing，端到端测试）** 是一种软件测试方法，用于验证整个应用程序的流程是否从开始到结束都能正常工作。它模拟真实用户场景，确保系统的各个组件（如前端、后端、数据库等）能够协同工作。

E2E 测试的核心特点
1. **覆盖完整流程**：测试从用户操作到系统响应的整个流程。
2. **模拟真实用户行为**：通过自动化工具模拟用户操作（如点击、输入等）。
3. **跨组件测试**：涵盖前端、后端、数据库、网络等多个组件。
4. **黑盒测试**：不关注内部实现，只验证输入和输出是否符合预期。
   E2E 测试的典型场景
1. **用户注册和登录**：
    - 测试用户从注册到登录的完整流程。
2. **购物流程**：
    - 测试从商品浏览、添加到购物车、下单到支付的流程。
3. **表单提交**
    - 测试表单填写、提交及后端处理的正确性。
4. **多步骤操作**：
    - 测试需要多个步骤才能完成的功能（如配置向导）.


## pull-labeler.yml
```yaml
name: labeler

on:
  pull_request_target:
    types: [opened, synchronize, reopened]

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

jobs:
  labeler:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/labeler@v5
        with:
          sync-labels: true
```
- 这里主要是在pr被创建，同步(推送)，重新打开后，自动会pr添加 标签。
- 相关的配置文件`.github/labeler.yml`.这个配置文件主要看 `any-glob-to-any-file`，如果修改了这个key所匹配的文件，就会添加标签`modifies/xxx`。
```yaml
modifies/docs:
  - changed-files:
      - any-glob-to-any-file:
          - "**/*.md"
          - "docs/**"

modifies/templates:
  - changed-files:
      - all-globs-to-any-file:
          - "templates/**"
          - "!templates/swagger/v1_json.tmpl"

modifies/api:
  - changed-files:
      - any-glob-to-any-file:
          - "routers/api/**"
          - "templates/swagger/v1_json.tmpl"

modifies/cli:
  - changed-files:
      - any-glob-to-any-file:
          - "cmd/**"

modifies/translation:
  - changed-files:
      - any-glob-to-any-file:
          - "options/locale/*.ini"

modifies/migrations:
  - changed-files:
      - any-glob-to-any-file:
          - "models/migrations/**"

modifies/internal:
  - changed-files:
      - any-glob-to-any-file:
          - ".air.toml"
          - "Makefile"
          - "Dockerfile"
          - "Dockerfile.rootless"
          - ".dockerignore"
          - "docker/**"
          - ".editorconfig"
          - ".eslintrc.yaml"
          - ".golangci.yml"
          - ".gitpod.yml"
          - ".markdownlint.yaml"
          - ".spectral.yaml"
          - "stylelint.config.js"
          - ".yamllint.yaml"
          - ".github/**"
          - ".gitea/"
          - ".devcontainer/**"
          - "build.go"
          - "build/**"
          - "contrib/**"

modifies/dependencies:
  - changed-files:
      - any-glob-to-any-file:
          - "package.json"
          - "package-lock.json"
          - "pyproject.toml"
          - "poetry.lock"
          - "go.mod"
          - "go.sum"

modifies/go:
  - changed-files:
      - any-glob-to-any-file:
          - "**/*.go"

modifies/frontend:
  - changed-files:
      - any-glob-to-any-file:
          - "**/*.js"
          - "**/*.ts"
          - "**/*.vue"

docs-update-needed:
  - changed-files:
      - any-glob-to-any-file:
          - "custom/conf/app.example.ini"
```

## release-nightly.yml
```yaml
name: release-tag-version

on:
  push:
    tags:
      - "v1.*"
      - "!v1*-rc*"
      - "!v1*-dev"

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false

jobs:
  binary:
    runs-on: namespace-profile-gitea-release-binary
    steps:
      - uses: actions/checkout@v4
      # fetch all commits instead of only the last as some branches are long lived and could have many between versions
      # fetch all tags to ensure that "git describe" reports expected Gitea version, eg. v1.21.0-dev-1-g1234567
      - run: git fetch --unshallow --quiet --tags --force
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: package-lock.json
      - run: make deps-frontend deps-backend
      # xgo build
      - run: make release
        env:
          TAGS: bindata sqlite sqlite_unlock_notify
      - name: import gpg key
        id: import_gpg
        uses: crazy-max/ghaction-import-gpg@v6
        with:
          gpg_private_key: ${{ secrets.GPGSIGN_KEY }}
          passphrase: ${{ secrets.GPGSIGN_PASSPHRASE }}
      - name: sign binaries
        run: |
          for f in dist/release/*; do
            echo '${{ secrets.GPGSIGN_PASSPHRASE }}' | gpg --pinentry-mode loopback --passphrase-fd 0 --batch --yes --detach-sign -u ${{ steps.import_gpg.outputs.fingerprint }} --output "$f.asc" "$f"
          done
      # clean branch name to get the folder name in S3
      - name: Get cleaned branch name
        id: clean_name
        run: |
          REF_NAME=$(echo "${{ github.ref }}" | sed -e 's/refs\/heads\///' -e 's/refs\/tags\/v//' -e 's/release\/v//')
          echo "Cleaned name is ${REF_NAME}"
          echo "branch=${REF_NAME}" >> "$GITHUB_OUTPUT"
      - name: configure aws
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ${{ secrets.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      - name: upload binaries to s3
        run: |
          aws s3 sync dist/release s3://${{ secrets.AWS_S3_BUCKET }}/gitea/${{ steps.clean_name.outputs.branch }} --no-progress
      - name: Install GH CLI
        uses: dev-hanz-ops/install-gh-cli-action@v0.1.0
        with:
          gh-cli-version: 2.39.1
      - name: create github release
        run: |
          gh release create ${{ github.ref_name }} --title ${{ github.ref_name }} --notes-from-tag dist/release/*
        env:
          GITHUB_TOKEN: ${{ secrets.RELEASE_TOKEN }}
  docker-rootful:
    runs-on: namespace-profile-gitea-release-docker
    steps:
      - uses: actions/checkout@v4
      # fetch all commits instead of only the last as some branches are long lived and could have many between versions
      # fetch all tags to ensure that "git describe" reports expected Gitea version, eg. v1.21.0-dev-1-g1234567
      - run: git fetch --unshallow --quiet --tags --force
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: gitea/gitea
          # this will generate tags in the following format:
          # latest
          # 1
          # 1.2
          # 1.2.3
          tags: |
            type=semver,pattern={{major}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{version}}
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: build rootful docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
  docker-rootless:
    runs-on: namespace-profile-gitea-release-docker
    steps:
      - uses: actions/checkout@v4
      # fetch all commits instead of only the last as some branches are long lived and could have many between versions
      # fetch all tags to ensure that "git describe" reports expected Gitea version, eg. v1.21.0-dev-1-g1234567
      - run: git fetch --unshallow --quiet --tags --force
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: gitea/gitea
          # each tag below will have the suffix of -rootless
          flavor: |
            suffix=-rootless,onlatest=true
          # this will generate tags in the following format (with -rootless suffix added):
          # latest
          # 1
          # 1.2
          # 1.2.3
          tags: |
            type=semver,pattern={{major}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{version}}
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: build rootless docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          file: Dockerfile.rootless
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```
这个文件很大，所以我们一个一个job来分析。
1.  binary job
    - 在check出代码的时候获取的是完整的提交历史而非仅最新的提交。
    - 接着是使用gpg对构建的二进制文件进行签名。
    - 最后就是将二进制文件上传至aws的s3桶，类似与minio。还会上传到github的Release。
2. docker-rootful
    - 这里主要是打包多平台的docker镜像
    - 通过docker/metadata-action来生成多个标签。比如1.2.3会生产`1`， `1.2`， `1.2.3`。
3. docker-rootless
    - 这里与上面的区别主要在于打标签，会增加后缀-rootless。同时保证生成`latest-rootless`标签。

## release-tag-rc.yml
```yaml
name: release-tag-rc

on:
  push:
    tags:
      - "v1*-rc*"

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false

jobs:
  binary:
    runs-on: namespace-profile-gitea-release-binary
    steps:
      - uses: actions/checkout@v4
      # fetch all commits instead of only the last as some branches are long lived and could have many between versions
      # fetch all tags to ensure that "git describe" reports expected Gitea version, eg. v1.21.0-dev-1-g1234567
      - run: git fetch --unshallow --quiet --tags --force
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: package-lock.json
      - run: make deps-frontend deps-backend
      # xgo build
      - run: make release
        env:
          TAGS: bindata sqlite sqlite_unlock_notify
      - name: import gpg key
        id: import_gpg
        uses: crazy-max/ghaction-import-gpg@v6
        with:
          gpg_private_key: ${{ secrets.GPGSIGN_KEY }}
          passphrase: ${{ secrets.GPGSIGN_PASSPHRASE }}
      - name: sign binaries
        run: |
          for f in dist/release/*; do
            echo '${{ secrets.GPGSIGN_PASSPHRASE }}' | gpg --pinentry-mode loopback --passphrase-fd 0 --batch --yes --detach-sign -u ${{ steps.import_gpg.outputs.fingerprint }} --output "$f.asc" "$f"
          done
      # clean branch name to get the folder name in S3
      - name: Get cleaned branch name
        id: clean_name
        run: |
          REF_NAME=$(echo "${{ github.ref }}" | sed -e 's/refs\/heads\///' -e 's/refs\/tags\/v//' -e 's/release\/v//')
          echo "Cleaned name is ${REF_NAME}"
          echo "branch=${REF_NAME}" >> "$GITHUB_OUTPUT"
      - name: configure aws
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ${{ secrets.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      - name: upload binaries to s3
        run: |
          aws s3 sync dist/release s3://${{ secrets.AWS_S3_BUCKET }}/gitea/${{ steps.clean_name.outputs.branch }} --no-progress
      - name: Install GH CLI
        uses: dev-hanz-ops/install-gh-cli-action@v0.1.0
        with:
          gh-cli-version: 2.39.1
      - name: create github release
        run: |
          gh release create ${{ github.ref_name }} --title ${{ github.ref_name }} --draft --notes-from-tag dist/release/*
        env:
          GITHUB_TOKEN: ${{ secrets.RELEASE_TOKEN }}
  docker-rootful:
    runs-on: namespace-profile-gitea-release-docker
    steps:
      - uses: actions/checkout@v4
      # fetch all commits instead of only the last as some branches are long lived and could have many between versions
      # fetch all tags to ensure that "git describe" reports expected Gitea version, eg. v1.21.0-dev-1-g1234567
      - run: git fetch --unshallow --quiet --tags --force
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: gitea/gitea
          flavor: |
            latest=false
          # 1.2.3-rc0
          tags: |
            type=semver,pattern={{version}}
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: build rootful docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
  docker-rootless:
    runs-on: namespace-profile-gitea-release-docker
    steps:
      - uses: actions/checkout@v4
      # fetch all commits instead of only the last as some branches are long lived and could have many between versions
      # fetch all tags to ensure that "git describe" reports expected Gitea version, eg. v1.21.0-dev-1-g1234567
      - run: git fetch --unshallow --quiet --tags --force
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: gitea/gitea
          # each tag below will have the suffix of -rootless
          flavor: |
            latest=false
            suffix=-rootless
          # 1.2.3-rc0
          tags: |
            type=semver,pattern={{version}}
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: build rootless docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          file: Dockerfile.rootless
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```
这里也是分成三个job，大同小异，这里就不作过多解释了。需要说一下的是rc是候选版本，候选版本在正式发布之前，发布一个或多个候选版本，供测试人员或用户进行最终验证。

## release-tag-version.yml
```yaml
name: release-tag-version

on:
  push:
    tags:
      - "v1.*"
      - "!v1*-rc*"
      - "!v1*-dev"

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false

jobs:
  binary:
    runs-on: namespace-profile-gitea-release-binary
    steps:
      - uses: actions/checkout@v4
      # fetch all commits instead of only the last as some branches are long lived and could have many between versions
      # fetch all tags to ensure that "git describe" reports expected Gitea version, eg. v1.21.0-dev-1-g1234567
      - run: git fetch --unshallow --quiet --tags --force
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          check-latest: true
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: package-lock.json
      - run: make deps-frontend deps-backend
      # xgo build
      - run: make release
        env:
          TAGS: bindata sqlite sqlite_unlock_notify
      - name: import gpg key
        id: import_gpg
        uses: crazy-max/ghaction-import-gpg@v6
        with:
          gpg_private_key: ${{ secrets.GPGSIGN_KEY }}
          passphrase: ${{ secrets.GPGSIGN_PASSPHRASE }}
      - name: sign binaries
        run: |
          for f in dist/release/*; do
            echo '${{ secrets.GPGSIGN_PASSPHRASE }}' | gpg --pinentry-mode loopback --passphrase-fd 0 --batch --yes --detach-sign -u ${{ steps.import_gpg.outputs.fingerprint }} --output "$f.asc" "$f"
          done
      # clean branch name to get the folder name in S3
      - name: Get cleaned branch name
        id: clean_name
        run: |
          REF_NAME=$(echo "${{ github.ref }}" | sed -e 's/refs\/heads\///' -e 's/refs\/tags\/v//' -e 's/release\/v//')
          echo "Cleaned name is ${REF_NAME}"
          echo "branch=${REF_NAME}" >> "$GITHUB_OUTPUT"
      - name: configure aws
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ${{ secrets.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      - name: upload binaries to s3
        run: |
          aws s3 sync dist/release s3://${{ secrets.AWS_S3_BUCKET }}/gitea/${{ steps.clean_name.outputs.branch }} --no-progress
      - name: Install GH CLI
        uses: dev-hanz-ops/install-gh-cli-action@v0.1.0
        with:
          gh-cli-version: 2.39.1
      - name: create github release
        run: |
          gh release create ${{ github.ref_name }} --title ${{ github.ref_name }} --notes-from-tag dist/release/*
        env:
          GITHUB_TOKEN: ${{ secrets.RELEASE_TOKEN }}
  docker-rootful:
    runs-on: namespace-profile-gitea-release-docker
    steps:
      - uses: actions/checkout@v4
      # fetch all commits instead of only the last as some branches are long lived and could have many between versions
      # fetch all tags to ensure that "git describe" reports expected Gitea version, eg. v1.21.0-dev-1-g1234567
      - run: git fetch --unshallow --quiet --tags --force
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: gitea/gitea
          # this will generate tags in the following format:
          # latest
          # 1
          # 1.2
          # 1.2.3
          tags: |
            type=semver,pattern={{major}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{version}}
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: build rootful docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
  docker-rootless:
    runs-on: namespace-profile-gitea-release-docker
    steps:
      - uses: actions/checkout@v4
      # fetch all commits instead of only the last as some branches are long lived and could have many between versions
      # fetch all tags to ensure that "git describe" reports expected Gitea version, eg. v1.21.0-dev-1-g1234567
      - run: git fetch --unshallow --quiet --tags --force
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: gitea/gitea
          # each tag below will have the suffix of -rootless
          flavor: |
            suffix=-rootless,onlatest=true
          # this will generate tags in the following format (with -rootless suffix added):
          # latest
          # 1
          # 1.2
          # 1.2.3
          tags: |
            type=semver,pattern={{major}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{version}}
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: build rootless docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          file: Dockerfile.rootless
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```
和以上两个release大同小异，不做过多分析了。




