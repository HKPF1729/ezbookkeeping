# AGENTS.md

ezBookkeeping —— 自托管个人理财应用。Go 后端（gin + xorm）将 Vue 3 前端作为静态文件一起提供服务。
单一仓库，无 monorepo 工具。

## 分支与改造原则

- 本仓库是**独立仓库**（已脱离原 fork 网络，与上游 mayswind/ezbookkeeping 无关联），
  定制化改造只在本仓库内进行。
- 唯一能影响他人项目的途径是对上游发起 PR/issue——**禁止这样做**。
- 代码中 `go.mod` 的 module path、`package.json` 的 repository.url 等仍写着
  `mayswind/ezbookkeeping`，那只是字符串引用，不产生任何连接，无需修改。
- CI 工作流（`.github/workflows/`、`.gitea/workflows/`）和 i18n 进度脚本属于公共区域，
  非必要不改动；自己的定制优先落在业务代码、`conf/`、`docker/custom-*.sh` 等位置。

## 命令

- 完整构建：`./build.sh package -o ezbookkeeping.tar.gz`（Windows：`build.bat` / `build.ps1`）。
  Docker：`./build.sh docker`。构建前默认先跑 lint → test → build，可用 `--no-lint` / `--no-test` 跳过。
- 仅后端：`./build.sh backend`（= `go vet ./...` + `go test ./... -v` + `go build`）。
  需要 CGO（sqlite3 驱动），因此需要 gcc/clang；Go 1.27.x。
- 仅前端：先 `npm install`（不是 `npm ci`），再 `npm run lint`、`npm run test`、`npm run build`。
- 本地开发：先跑后端（`./ezbookkeeping server run`，配置在 `conf/ezbookkeeping.ini`，端口 8080），
  再 `npm run serve` → Vite 跑在 **:8081（strictPort）**，把 `/api`、`/mcp`、
  `/server_settings.js`、`/oauth2`、`/avatar` 等代理到 `127.0.0.1:8080`。
  前端开发必须先起后端。
- 单条测试：`go test ./pkg/services -run TestName -v`；
  `npx vitest run src/lib/__tests__/math.test.ts`。

## 常见坑

- **`go test ./...` 会真实请求第三方央行汇率 API**（除非设置 `BUILD_PIPELINE=1`）。
  本地请用 `BUILD_PIPELINE=1 go test ./...` 以避免联网。
- `npm run lint` 实为 `vue-tsc --noEmit && eslint . --fix` —— 会**改写文件**，跑完检查 diff。
- 前端测试跑在 vitest `environment: 'node'`（无 DOM），测试文件在 `src/**/__tests__/*.test.ts`。
- `build.sh` 测试前会 `go clean -cache` → 首次构建总是很慢，属正常。
- 构建会从 `package.json` + git（`git rev-parse`、`git-rev-sync`）注入版本号/commit：
  必须在 git checkout 里构建，否则失败或注入错误值。

## 架构

- 入口：`ezbookkeeping.go` → `cmd/` 子命令（urfave/cli）：`server`、`database`、`user-data`、
  `cron-jobs`、`security`、`utilities`。**所有 HTTP 路由都注册在 `cmd/webserver.go`。**
- `pkg/` 分层：`api/` HTTP 处理器 → `services/` 业务逻辑 → `models/` xorm 实体；
  `settings/` 配置，`datastore/` 数据库会话。
- **没有迁移文件**：表结构在服务启动时从 Go 结构体同步
  （`auto_update_database = true`），或手动执行 `./ezbookkeeping database update`。
- 前端：Vite root 是 `src/`（不是仓库根目录），输出 `dist/`。一套代码两个完整 UI：
  `mobile.html` + `src/router/mobile.ts` + `src/views/mobile`（Framework7）与
  `desktop.html` + `src/router/desktop.ts` + `src/views/desktop`（Vuetify）；
  `index.html` 只按 user agent 跳转。共享代码：`src/lib`、`src/models`、`src/stores`、
  `src/views/base`、`src/components/{base,common}`。
- 运行时配置通过后端生成的 `server_settings.js` 下发到浏览器
  （`window.EZBOOKKEEPING_SERVER_SETTINGS`，前端在 `src/lib/server_settings.ts` 读取），
  不是构建期 define。
- 配置：`conf/ezbookkeeping.ini`；任意配置项可用环境变量 `EBK_<SECTION>_<ITEM>` 覆盖
  （基于文件内容的：`EBKCFP_<SECTION>_<ITEM>`），工作目录用 `EBK_WORK_DIR`，
  自定义配置文件用 `--conf-path`。
- i18n 分**两处**维护：`src/locales/*.json`（+ 在 `src/locales/index.ts` 注册）和
  `pkg/locales/*.go`（+ `all_locales.go`）。新增语言两边都要改；
  翻译流程见 https://ezbookkeeping.mayswind.net/translating

## 定制化钩子

- `docker/custom-backend-pre-setup.sh` / `docker/custom-frontend-pre-setup.sh`：
  Docker 构建时若存在则会执行——这是官方预留的本地构建修改入口。
- `third-party-dependencies.json` + `contributors.json` 为手工维护，经 vite `define`
  注入到 About 页面；新增重要依赖时记得更新。

## 代码风格

- 4 空格缩进，Go 用 tab（`.editorconfig`）。TS 开启 `strict`，含 `noUncheckedIndexedAccess`。
- CI 在每个 PR 上跑 `./build.sh`（`.github/workflows/`）；`.gitea/workflows/` 是
  自托管 Gitea 的镜像。
