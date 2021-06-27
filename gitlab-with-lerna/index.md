# 使用 gitlab NPM 仓库存储 lerna monorepo

## 简介

### Lerna 介绍

Lerna 是一个比较常用的 monorepo 管理工具。使用 Lerna 能将单个大项目拆分为多个 package，每个 package 可以有自己的版本，能够独立发布。

#### Lerna 常用命令

- lerna changed

用来检测 package 是有有变动，即是否需要更新 package 版本。

- lerna version

用来给 package 更新版本编号，版本编号遵循 [SEMVER](https://semver.org/) 版本规范。lerna version 后面可以跟随版本类型(patch / minor / major / etc... )，命令运行结束后会自动修改各 package 中的 package.json 对应的版本号，并在 git 分支中加入 对应的 tag，随后会将代码 push 至远程仓库。

- lerna publish

该命令包含 lerna version 所做的事情。在版本更新后，会按照各 package 中的定义将代码发布至 NPM registry。

### gitlab NPM 仓库 介绍

它是一个私有的 NPM 仓库，能将一些不想公开分 的 NPM 代码包存储在这个仓库当中。存储在此仓库中的 package 必须具有特定的[scope](https://docs.npmjs.com/cli/v6/using-npm/scope)。在 gitlab 私有 NPM 仓库中，package 的 scope 必须和创建项目时的命名空间保持一致，对于个人用户而言一般就是与用户名保持一致。

#### gitlab Token 介绍

在 gitlab NPM 仓库中，需要使用 Token 对 NPM 包进行 上传、更新、下载等操作。一般分为以下三种 Token：

- [Personal Access Token](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html)

该 token 访问范围最广且与用户相关，可以访问 gitlab 上所有有权限访问的仓库及代码。在私有仓库这个使用场景中，一般将 Token 的 Scope 设置为 api 即可。

- [Deploy Token](https://docs.gitlab.com/ee/user/project/deploy_tokens/index.html)

该 token 访问范围仅限单个 gitlab repo 内与用户不相关，如果仅需获取私有 npm 包可以只给予 token read_package_registry 的权限，如果需要发布 npm 包则需要 read_package_registry 的权限。

- [CI_JOB_TOKEN](https://docs.gitlab.com/ee/api/README.html#gitlab-ci-job-token)

该 token 会在 gitlab CI 运行时生成并且作为预定义的环境变量注入，只能访问特定的 API，刚好包括 npm 仓库，因此可以用来做 npm 包发布或者安装等操作。

#### gitlab NPM Endpoint 介绍

由于使用 NPM 私有仓库，因此需要在 NPM 安装、发布 NPM 包时制定具体的 registry。这个 registry 就是对应的 gitlab NPM Endpoint。目前 gitlab 有两种 NPM Endpoint，分别为 Project-level NPM endpoint 和 Instance-level NPM endpoint。

- Project-level NPM Endpoint

设置方式：

```shell
# Set URL for your scoped packages.
# For example package with name `@foo/bar` will use this URL for download
npm config set @foo:registry https://gitlab.example.com/api/v4/projects/<your_project_id>/packages/npm/

# Add the token for the scoped packages URL. Replace <your_project_id>
# with the project where your package is located.
npm config set '//gitlab.example.com/api/v4/projects/<your_project_id>/packages/npm/:_authToken' "<your_token>"
```

每个项目都会有一个独立的 Endpoint，能够允许通过这个 endpoint 发布与安装对应包裹。

- Instance-level NPM Endpoint

设置方式：

```shell
# Set URL for your scoped packages.
# For example package with name `@foo/bar` will use this URL for download
npm config set @foo:registry https://gitlab.example.com/api/v4/packages/npm/

# Add the token for the scoped packages URL. This will allow you to download
# `@foo/` packages from private projects.
npm config set -- '//gitlab.example.com/api/v4/packages/npm/:_authToken' "<your_token>"
```

只能用于安装 NPM 包，不能用于发布 NPM。适用于同个 scope 的 NPM 包散乱在不同 project 中。

## 使用 lerna 初始化项目

```
npx lerna init
```

运行上面命令可以创建一个基本的 lerna mono project。项目创建完成之后会初始化`lerna.json`以及`package.json`。

## 添加 package1

```
npx lerna create @viowan/package1
```

这里使用`viowan` 作为 package1 的 scope。在 gitlab 中 scope 不能为自定义的字符串， 一般使用所项目的 root namespace 作为 scope 的值，根据 NPM 包的命名规范，scope 应该使用全小写的值。

## 将包裹发布到 gitlab NPM registry

### 规划 NPM 发布流程

一个好的共用包项目，应该有一个稳定、明确、易用的发布流程。比起本地发布 NPM 包，在 CI 中完成整个发布的流程明显更合适。因此可以考虑按照以下流程完成包的发布：

- 基于 master 分支创建修改提交 MR
- Review & Approve MR
- 使用`lerna version`的命令完成 tag 的创建以及 package.json 的更新
- 合并 MR 到 master 分支当中
- master 分支运行`npm publish from-package` 命令完成 NPM 包的发布

### 创建.gitlab-ci.yml

```yaml
image: node:latest

stages:
  - deploy

deploy_to_gitlab:
  stage: deploy
  only:
    - main
  script:
    - npm config set -- '//gitlab.com/api/v4/projects/<your_project_id>/packages/npm/:_authToken' "${CI_JOB_TOKEN}"
    - npx lerna bootstrap
    - npx lerna publish from-package --registry https://gitlab.com/api/v4/projects/<your_project_id>/packages/npm/ --yes
```

在这个文件中，我们定义了一个`deploy_to_gitlab`的任务，在`main`分支更新后会执行以下内容

- 自动设置 CI_JOB_TOKEN 为 authToken
- 安装所需依赖
- 使用`publish`命令将，新版本的包发布到 gitlab registry 当中。

### 创建新的包版本

```
npx lerna version patch --yes --include-merged-tags
```

运行以上命令会自动增加一个版本，并生成一个 tag 以及 package.json 修改的 commit。在 push 到 gitlab 上合并进入 main 分支后会使用前面定义好的 CI 脚本完成发布工作。
