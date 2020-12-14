---
layout: classic-docs
title: "ワークフローを使用したジョブのスケジュール"
short-title: "ワークフローを使用したジョブのスケジュール"
description: "ワークフローを使用したジョブのスケジュール"
order: 30
version:
  - Cloud
  - Server v2.x
---

Workflows help you increase the speed of your software development through faster feedback, shorter reruns, and more efficient use of resources. This document describes the Workflows feature and provides example configurations in the following sections:

- 目次
{:toc}

## 概要

A **workflow** is a set of rules for defining a collection of jobs and their run order. Workflows support complex job orchestration using a simple set of configuration keys to help you resolve failures sooner.

With workflows, you can:

- リアルタイムのステータス フィードバックによって、ジョブの実行とトラブルシューティングを分離する
- 定期的に実行したいジョブをワークフローとしてスケジュールする
- Fan-out to run multiple jobs concurrently for efficient version testing.
- ファンインして複数のプラットフォームにすばやくデプロイする

For example, if only one job in a workflow fails, you will know it is failing in real-time. Instead of wasting time waiting for the entire build to fail and rerunning the entire job set, you can rerun *just the failed job*.

### ステータス
{:.no_toc}

Workflows may appear with one of the following states:

| State       | Description                                                                                                                                  |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| RUNNING     | Workflow is in progress                                                                                                                      |
| NOT RUN     | Workflow was never started                                                                                                                   |
| CANCELLED   | Workflow was cancelled before it finished                                                                                                    |
| FAILING     | A job in the workflow has failed                                                                                                             |
| FAILED      | One or more jobs in the workflow failed                                                                                                      |
| SUCCESS     | All jobs in the workflow completed successfully                                                                                              |
| ON HOLD     | A job in the workflow is waiting for approval                                                                                                |
| NEEDS SETUP | A workflow stanza is not included or is incorrect in the [config.yml]({{ site.baseurl }}/2.0/configuration-reference/) file for this project |

### 制限事項
{:.no_toc}

- Projects that have pipelines enabled may use the CircleCI API to trigger workflows. 
- Config without workflows requires a job called `build`.

Refer to the [Workflows]({{ site.baseurl }}/2.0/faq/#workflows) section of the FAQ for additional information and limitations.

## Workflows configuration examples

*For a full specification of the* `workflows` *key, see the [Workflows]({{ site.baseurl }}/2.0/configuration-reference/#workflows) section of the Configuring CircleCI document.*

**Note:** Projects configured with Workflows often include multiple jobs that share syntax for Docker images, environment variables, or `run` steps. Refer the [YAML Anchors/Aliases](http://yaml.org/spec/1.2/spec.html#id2765878) documentation for information about how to alias and reuse syntax to keep your `.circleci/config.yml` file small. See the [Reuse YAML in the CircleCI Config](https://circleci.com/blog/circleci-hacks-reuse-yaml-in-your-circleci-config-with-yaml/) blog post for a summary.

To run a set of concurrent jobs, add a new `workflows:` section to the end of your existing `.circleci/config.yml` file with the version and a unique name for the workflow. The following sample `.circleci/config.yml` file shows the default workflow orchestration with two concurrent jobs. It is defined by using the `workflows:` key named `build_and_test` and by nesting the `jobs:` key with a list of job names. The jobs have no dependencies defined, therefore they will run concurrently.

```yaml
jobs:
  build:
    docker:
      - image: circleci/<language>:<version TAG>
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD  # context / project UI env-var reference
    steps:
      - checkout
      - run: <command>
  test:
    docker:
      - image: circleci/<language>:<version TAG>
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD  # context / project UI env-var reference
    steps:
      - checkout
      - run: <command>
workflows:
  version: 2
  build_and_test:
    jobs:
      - build
      - test
```

See the [Sample Parallel Workflow config](https://github.com/CircleCI-Public/circleci-demo-workflows/blob/parallel-jobs/.circleci/config.yml) for a full example.

## Tips for advanced configuration

Using workflows enables users to create much more advanced configurations over running a single set of jobs. With more customizability and control comes more room for error, however. When using workflows try to do the following:

- Move the quickest jobs up to the start of your workflows. For example, lint or syntax checking should happen before longer-running, more computationally expensive jobs.
- Using a "setup" job at the *start* of a workflow can be helpful to do some preflight checks and populate a workspace for all the following jobs.

Consider reading the [optimization]({{ site.baseurl }}/2.0/optimizations) and [advanced config]({{ site.baseurl }}/2.0/adv-config) documentation for more tips related to improving your configuration.

### Sequential job execution example
{:.no_toc}

The following example shows a workflow with four sequential jobs. The jobs run according to configured requirements, each job waiting to start until the required job finishes successfully as illustrated in the diagram.

![Sequential Job Execution Workflow]({{ site.baseurl }}/assets/img/docs/sequential_workflow.png)

The following `config.yml` snippet is an example of a workflow configured for sequential job execution:

```yaml
workflows:
  version: 2
  build-test-and-deploy:
    jobs:
      - build
      - test1:
          requires:
            - build
      - test2:
          requires:
            - test1
      - deploy:
          requires:
            - test2
```

The dependencies are defined by setting the `requires:` key as shown. The `deploy:` job will not run until the `build` and `test1` and `test2` jobs complete successfully. A job must wait until all upstream jobs in the dependency graph have run. So, the `deploy` job waits for the `test2` job, the `test2` job waits for the `test1` job and the `test1` job waits for the `build` job.

See the [Sample Sequential Workflow config](https://github.com/CircleCI-Public/circleci-demo-workflows/blob/sequential-branch-filter/.circleci/config.yml) for a full example.

### Fan-out/fan-in workflow example
{:.no_toc}

The illustrated example workflow runs a common build job, then fans-out to run a set of acceptance test jobs concurrently, and finally fans-in to run a common deploy job.

![Fan-out and Fan-in Workflow]({{ site.baseurl }}/assets/img/docs/fan-out-in.png)

The following `config.yml` snippet is an example of a workflow configured for fan-out/fan-in job execution:

```yaml
workflows:
  version: 2
  build_accept_deploy:
    jobs:
      - build
      - acceptance_test_1:
          requires:
            - build
      - acceptance_test_2:
          requires:
            - build
      - acceptance_test_3:
          requires:
            - build
      - acceptance_test_4:
          requires:
            - build
      - deploy:
          requires:
            - acceptance_test_1
            - acceptance_test_2
            - acceptance_test_3
            - acceptance_test_4
```

In this example, as soon as the `build` job finishes successfully, all four acceptance test jobs start. The `deploy` job must wait for all four acceptance test jobs to complete successfully before it starts.

See the [Sample Fan-in/Fan-out Workflow config](https://github.com/CircleCI-Public/circleci-demo-workflows/tree/fan-in-fan-out) for a full example.

## Holding a workflow for a manual approval

Workflows can be configured to wait for manual approval of a job before continuing to the next job. Anyone who has push access to the repository can click the Approval button to continue the workflow. To do this, add a job to the `jobs` list with the key `type: approval`. Let's look at a commented config example.

```yaml
# ...
# << your config for the build, test1, test2, and deploy jobs >>
# ...

workflows:
  version: 2
  build-test-and-approval-deploy:
    jobs:

      - build  # 設定ファイルに含まれるカスタム ジョブ。コードをビルドします。
      - test1: # カスタム ジョブ。テスト スイート 1 を実行します。
          requires: # `build` ジョブが完了するまで、test1 は実行されません。
            - build
      - test2: # another custom job; runs test suite 2,
          requires: # test2 is dependent on the success of job `test1`
            - test1
      - hold: # <<< A job that will require manual approval in the CircleCI web application.
          type: approval # <<< このキー・値のペアにより、ワークフローのステータスが "On Hold" に設定されます。
          requires: # test2 が成功した場合にのみ "hold" ジョブを実行します。
           - test2
      # `hold` ジョブが承認されると、`hold` ジョブを必要とする後続のジョブが実行されます。 
      # この例では、ユーザーが手動でデプロイ ジョブをトリガーしています。
      - deploy:
          requires:
            - hold
```

The outcome of the above example is that the `deploy:` job will not run until you click the `hold` job in the Workflows page of the CircleCI app and then click Approve. In this example the purpose of the `hold` job is to wait for approval to begin deployment. A job can be approved for up to 15 days after being issued.

Some things to keep in mind when using manual approval in a workflow:

- `approval` is a special job type that is **only** available to jobs under the `workflow` key
- The `hold` job must be a unique name not used by any other job.
- that is, your custom configured jobs, such as `build` or `test1` in the example above wouldn't be given a `type: approval` key.
- The name of the job to hold is arbitrary - it could be `wait` or `pause`, for example, as long as the job has a `type: approval` key in it.
- All jobs that are to run after a manually approved job *must* `require:` the name of that job. Refer to the `deploy:` job in the above example.
- Jobs run in the order defined until the workflow processes a job with the `type: approval` key followed by a job on which it depends.

The following screenshot demonstrates a workflow on hold. 

{:.tab.switcher.Cloud}
![Approved Jobs in On Hold Workflow]({{ site.baseurl }}/assets/img/docs/approval_job_cloud.png)

{:.tab.switcher.Server}
![Switch Organization Menu]({{ site.baseurl }}/assets/img/docs/approval_job.png)

By clicking on the pending job's name (`build`, in the screenshot above), an approval dialog box appears requesting that you approve or cancel the holding job.

After approving, the rest of the workflow runs as directed.

## Scheduling a workflow

It can be inefficient and expensive to run a workflow for every commit for every branch. Instead, you can schedule a workflow to run at a certain time for specific branches. This will disable commits from triggering jobs on those branches.

Consider running workflows that are resource-intensive or that generate reports on a schedule rather than on every commit by adding a `triggers` key to the configuration. The `triggers` key is **only** added under your `workflows` key. This feature enables you to schedule a workflow run by using `cron` syntax to represent Coordinated Universal Time (UTC) for specified branches.

**Note:** In CircleCI v2.1, when no workflow is provided in config, an implicit one is used. However, if you declare a workflow to run a scheduled build, the implicit workflow is no longer run. You must add the job workflow to your config in order for CircleCI to also build on every commit.

**Note:** Please note that when you schedule a workflow, the workflow will be counted as an individual user seat.

### Nightly example
{:.no_toc}

デフォルトでは、`git push` ごとにワークフローがトリガーされます。 スケジュールに沿ってワークフローをトリガーするには、ワークフローに `triggers` キーを追加し、`schedule` を指定します。

以下の例では、`nightly` ワークフローが毎日午前 0 時 (UTC) に実行されるように構成しています。 `cron` キーは、POSIX `crontab` 構文を使用して指定されます。`cron` 構文の基本については、[crontab の man ページ](https://www.unix.com/man-page/POSIX/1posix/crontab/)を参照してください。 このワークフローは、`master` ブランチと `beta` ブランチで実行されます。

**メモ:** スケジュールが設定されたワークフローは、最大 15 分遅れることがあります。 これは、午前 0 時 (UTC) などの混雑時の信頼性を維持するために実施されます。 スケジュールが設定されたワークフローが分単位の正確性で開始されることを想定しないようにご注意ください。

```yaml
workflows:
  version: 2
  commit:
    jobs:
      - test
      - deploy
  nightly:
    triggers:
      - schedule:
          cron: "0 0 * * *"
          filters:
            branches:
              only:
                - master
                - beta
    jobs:
      - coverage
```

上の例の `commit` ワークフローに `triggers` キーはなく、`git push` ごとに実行されます。 `nightly` ワークフローには `triggers` キーがあるため、`schedule` の指定に沿って実行されます。

### Specifying a valid schedule
{:.no_toc}

有効な `schedule` には、`cron` キーと `filters` キーが必要です。

`cron` キーの値は、[有効な crontab エントリ](https://crontab.guru/)である必要があります。

**メモ:** cron のステップ構文 (たとえば、`*/1`、`*/20`) は**サポートされません**。 エレメントのカンマ区切りリスト内の範囲エレメントも**サポートされません**。

`filters` キーの値は、特定ブランチ上の実行ルールを定義するマップです。

詳細については、「CircleCI を設定する」の「[branches ]({{ site.baseurl }}/2.0/configuration-reference/#branches-1)」を参照してください。

このサンプルの全文は、[ワークフローのスケジュールを設定する構成例](https://github.com/CircleCI-Public/circleci-demo-workflows/blob/try-schedule-workflow/.circleci/config.yml)でご覧いただけます。

## Using contexts and filtering in your workflows

以下のセクションでは、コンテキストとフィルターを使用してジョブの実行を管理する例を示します。

### Using job contexts to share environment variables
{:.no_toc}

以下の例は、コンテキストを使用して環境変数を共有して 4 つの順次ジョブから成るワークフローを示しています。 アプリケーションでこの設定を行う詳細な手順については、[コンテキストに関するドキュメント]({{ site.baseurl }}/2.0/contexts)を参照してください。

以下の `config.yml` スニペットは、`org-global` コンテキスト内に定義されたリソースを使用するように順次ジョブ ワークフローを構成した例です。

```yaml
workflows:
  version: 2
  build-test-and-deploy:
    jobs:
      - build
      - test1:
          requires:
            - build
          context: org-global  
      - test2:
          requires:
            - test1
          context: org-global  
      - deploy:
          requires:
            - test2
```

これらの環境変数は、ここに示すように、`context` キーにデフォルト名 `org-global` を設定することによって定義されます。 この構成例の `test1` ジョブと `test2` ジョブは、組織の所属ユーザーによって開始された場合、同じ共有環境変数を使用します。 デフォルトでは、組織に対して設定されたコンテキストには、その組織内のすべてのプロジェクトがアクセスできます。

### Branch-level job execution
{:.no_toc}

以下の例は、3 つのブランチ (Dev、Stage、Pre-Prod) 上にあるジョブを使用して構成されたワークフローを示しています。 ワークフローは `jobs` 設定の下にネストされた `branches` キーを無視するため、ジョブレベルのブランチを使用して後でワークフローを追加する場合は、ジョブレベルのブランチを削除し、代わりにそれを `config.yml` のワークフロー セクションで宣言する必要があります。

![ブランチレベルでジョブを実行する]({{ site.baseurl }}/assets/img/docs/branch_level.png)

以下の `config.yml` スニペットは、ブランチレベルのジョブ実行を構成するワークフローの例を示しています。

```yaml
workflows:
  version: 2
  dev_stage_pre-prod:
    jobs:
      - test_dev:
          filters:  # 正規表現フィルターを使用すると、ブランチ全体が一致する必要があります
            branches:
              only:  # 以下の正規表現フィルターに一致するブランチのみが実行されます
                - dev
                - /user-.*/
      - test_stage:
          filters:
            branches:
              only: stage
      - test_pre-prod:
          filters:
            branches:
              only: /pre-prod(?:-.+)?$/
```

正規表現の詳細については、この後の「[正規表現を使用してタグとブランチをフィルタリングする](#正規表現を使用してタグとブランチをフィルタリングする)」を参照してください。

ワークフロー構成例の全文は、ブランチを含む順次ワークフロー サンプル プロジェクトの[設定ファイル](https://github.com/CircleCI-Public/circleci-demo-workflows/blob/sequential-branch-filter/.circleci/config.yml)でご覧いただけます。

### Executing workflows for a git tag
{:.no_toc}

明示的にタグ フィルターを設定しない限り、CircleCI はタグに関するワークフローを実行しません。 さらに、ジョブが (直接的または間接的に) 他のジョブを必要とする場合は、[正規表現を使用](#正規表現を使用してタグとブランチをフィルタリングする)して、それらのジョブに対応するタグ フィルターを指定する必要があります。 軽量のタグと注釈付きのタグがサポートされています。

以下の例では、2 つのキーが定義されています。

- `untagged-build` runs the `build` job for all branches.
- `tagged-build` runs `build` for all branches **and** all tags starting with `v`.

```yaml
workflows:
  version: 2
  untagged-build:
    jobs:
      - build
  tagged-build:
    jobs:
      - build:
          filters:
            tags:
              only: /^v.*/
```

以下の例では、`build-n-deploy` ワークフローで 2 つのジョブが定義されています。

- The `build` job runs for all branches and all tags.
- The `deploy` job runs for no branches and only for tags starting with 'v'.

```yaml
workflows:
  version: 2
  build-n-deploy:
    jobs:
      - build:
          filters:  # `deploy` にタグ フィルターがあり、それが `build` を必要とするため必須
            tags:
              only: /.*/
      - deploy:
          requires:
            - build
          filters:
            tags:
              only: /^v.*/
            branches:
              ignore: /.*/
```

以下の例では、`build-test-deploy` ワークフローで 3 つのジョブが定義されています。

- The `build` job runs for all branches and only tags starting with 'config-test'.
- The `test` job runs for all branches and only tags starting with 'config-test'.
- The `deploy` job runs for no branches and only tags starting with 'config-test'.

```yaml
workflows:
  version: 2
  build-test-deploy:
    jobs:
      - build:
          filters:  # `test` にタグ フィルターがあり、それが `build` を必要とするため必須
            tags:
              only: /^config-test.*/
      - test:
          requires:
            - build
          filters:  # `deploy` にタグ フィルターがあり、それが `test` を必要とするため必須
            tags:
              only: /^config-test.*/
      - deploy:
          requires:
            - test
          filters:
            tags:
              only: /^config-test.*/
            branches:
              ignore: /.*/
```

In the example below, two jobs are defined (`test` and `deploy`) and three workflows utilize those jobs:

- The `build` workflow runs for all branches except `main` and is not run on tags.
- The `staging` workflow will only run on the `main` branch and is not run on tags.
- The `production` workflow runs for no branches and only for tags starting with `v.`.

```yaml
workflows:
  build: # This workflow will run on all branches except 'main' and will not run on tags
    jobs:
      - test:
          filters:
            branches:
              ignore: main
  staging: # This workflow will only run on 'main' and will not run on tags
    jobs:
      - test:
          filters: &filters-staging # this yaml anchor is setting these values to "filters-staging"
            branches:
              only: main
            tags:
              ignore: /.*/
      - deploy:
          requires:
            - build
          filters:
            <<: *filters-staging # this is calling the previously set yaml anchor
  production: # This workflow will only run on tags (specifically starting with 'v.') and will not run on branches
    jobs:
      - test:
          filters: &filters-production # this yaml anchor is setting these values to "filters-production"
            branches:
              ignore: /.*/
            tags:
              only: /^v.*/
      - deploy:
          requires:
            - build
          filters:
            <<: *filters-production # this is calling the previously set yaml anchor
```

**Note:** Webhook payloads from GitHub [are capped at 5MB](https://developer.github.com/webhooks/#payloads) and [for some events](https://developer.github.com/v3/activity/events/types/#createevent) a maximum of 3 tags. If you push several tags at once, CircleCI may not receive all of them.

### Using regular expressions to filter tags and branches
{:.no_toc}

CircleCI branch and tag filters support the Java variant of regex pattern matching. When writing filters, CircleCI matches exact regular expressions.

For example, `only: /^config-test/` only matches the `config-test` tag. To match all tags starting with `config-test`, use `only: /^config-test.*/` instead.

Using tags for semantic versioning is a common use case. To match patch versions 3-7 of a 2.1 release, you could write `/^version-2\.1\.[3-7]/`.

For full details on pattern-matching rules, see the [java.util.regex documentation](https://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html).

## Using workspaces to share data among jobs

Each workflow has an associated workspace which can be used to transfer files to downstream jobs as the workflow progresses. The workspace is an additive-only store of data. Jobs can persist data to the workspace. This configuration archives the data and creates a new layer in an off-container store. Downstream jobs can attach the workspace to their container filesystem. Attaching the workspace downloads and unpacks each layer based on the ordering of the upstream jobs in the workflow graph.

![workspaces data flow]({{ site.baseurl }}/assets/img/docs/workspaces.png)

Use workspaces to pass along data that is unique to this run and which is needed for downstream jobs. Workflows that include jobs running on multiple branches may require data to be shared using workspaces. Workspaces are also useful for projects in which compiled data are used by test containers.

For example, Scala projects typically require lots of CPU for compilation in the build job. In contrast, the Scala test jobs are not CPU-intensive and may be parallelised across containers well. Using a larger container for the build job and saving the compiled data into the workspace enables the test containers to use the compiled Scala from the build job.

A second example is a project with a `build` job that builds a jar and saves it to a workspace. The `build` job fans-out into the `integration-test`, `unit-test`, and `code-coverage` to run those tests concurrently using the jar.

To persist data from a job and make it available to other jobs, configure the job to use the `persist_to_workspace` key. Files and directories named in the `paths:` property of `persist_to_workspace` will be uploaded to the workflow's temporary workspace relative to the directory specified with the `root` key. The files and directories are then uploaded and made available for subsequent jobs (and re-runs of the workflow) to use.

Configure a job to get saved data by configuring the `attach_workspace` key. The following `config.yml` file defines two jobs where the `downstream` job uses the artifact of the `flow` job. The workflow configuration is sequential, so that `downstream` requires `flow` to finish before it can start.

```yaml
# Note that the following stanza uses CircleCI 2.1 to make use of a Reusable Executor
# This allows defining a docker image to reuse across jobs.
# visit https://circleci.com/docs/2.0/reusing-config/#authoring-reusable-executors to learn more.

version: 2.1

executors:
  my-executor:
    docker:

      - image: buildpack-deps:jessie
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD  # context / project UI env-var reference
    working_directory: /tmp

jobs:
  flow:
    executor: my-executor
    steps:

      - run: mkdir -p workspace
      - run: echo "Hello, world!" > workspace/echo-output

      # Persist the specified paths (workspace/echo-output) into the workspace for use in downstream job. 

      - persist_to_workspace:
          # Must be an absolute path, or relative path from working_directory. This is a directory on the container which is 
          # taken to be the root directory of the workspace.
          root: workspace
          # Must be relative path from root
          paths:
            - echo-output

  downstream:
    executor: my-executor
    steps:

      - attach_workspace:
          # Must be absolute path or relative path from working_directory
          at: /tmp/workspace

      - run: |
          if [[ `cat /tmp/workspace/echo-output` == "Hello, world!" ]]; then
            echo "It worked!";
          else
            echo "Nope!"; exit 1
          fi

workflows:
  version: 2

  btd:
    jobs:

      - flow
      - downstream:
          requires:
            - flow
```

For a live example of using workspaces to pass data between build and deploy jobs, see the [`config.yml`](https://github.com/circleci/circleci-docs/blob/master/.circleci/config.yml) that is configured to build the CircleCI documentation.

For additional conceptual information on using workspaces, caching, and artifacts, refer to the [Persisting Data in Workflows: When to Use Caching, Artifacts, and Workspaces](https://circleci.com/blog/persisting-data-in-workflows-when-to-use-caching-artifacts-and-workspaces/) blog post.

## Rerunning a workflow's failed jobs

When you use workflows, you increase your ability to rapidly respond to failures. To rerun only a workflow's **failed** jobs, click the **Workflows** icon in the app and select a workflow to see the status of each job, then click the **Rerun** button and select **Rerun from failed**.

![CircleCI Workflows Page]({{ site.baseurl }}/assets/img/docs/rerun-from-failed.png)

## トラブルシューティング

This section describes common problems and solutions for Workflows.

### Workflow and subsequent jobs do not trigger

If you do not see your workflows triggering, a common cause is a configuration error preventing the workflow from starting. As a result, the workflow does not start any jobs. Navigate to your project's pipelines and click on your workflow name to discern what might be failing.

### Rerunning workflows fails
{:.no_toc}

It has been observed that in some cases, a failure happens before the workflow runs (during pipeline processing). In this case, re-running the workflow will fail even though it was succeeding before the outage. To work around this, push a change to the project's repository. This will re-run pipeline processing first, and then run the workflow.

### Workflows waiting for status in GitHub
{:.no_toc}

If you have implemented Workflows on a branch in your GitHub repository, but the status check never completes, there may be status settings in GitHub that you need to deselect. For example, if you choose to protect your branches, you may need to deselect the `ci/circleci` status key as this check refers to the default CircleCI 1.0 check, as follows:

![Uncheck GitHub Status Keys]({{ site.baseurl }}/assets/img/docs/github_branches_status.png)

Having the `ci/circleci` checkbox enabled will prevent the status from showing as completed in GitHub when using a workflow because CircleCI posts statuses to GitHub with a key that includes the job by name.

Go to Settings > Branches in GitHub and click the Edit button on the protected branch to deselect the settings, for example https://github.com/your-org/project/settings/branches.

## See also
{:.no_toc}

- For procedural instructions on how to add Workflows your configuration as you are migrating from a 1.0 `circle.yml` file to a 2.0 `.circleci/config.yml` file, see the [Steps to Configure Workflows]({{ site.baseurl }}/2.0/migrating-from-1-2/) section of the Migrating from 1.0 to 2.0 document.

- For frequently asked questions and answers about Workflows, see the [Workflows]({{ site.baseurl }}/2.0/faq) section of the FAQ.

- For demonstration apps configured with Workflows, see the [CircleCI Demo Workflows](https://github.com/CircleCI-Public/circleci-demo-workflows) on GitHub.

## Video: configure multiple jobs with workflows
{:.no_toc}

<iframe width="560" height="315" src="https://www.youtube.com/embed/3V84yEz6HwA" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen mark="crwd-mark"></iframe> 

### Video: how to schedule your builds to test and deploy automatically
{:.no_toc}

<iframe width="560" height="315" src="https://www.youtube.com/embed/FCiMD6Gq34M" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen mark="crwd-mark"></iframe>