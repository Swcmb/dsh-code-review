# 发布流程（npm + GitHub Actions）

> 打 tag 即自动发布：`git tag vX.Y.Z && git push origin vX.Y.Z` → GitHub Actions 跑测试 → 发布到 npm → 建 GitHub Release。

> 仓库归属：`github.com/jkin8010/dsh-code-review`（GitHub 个人仓库，不变）；npm scope：`@jkin8010`（npm 组织）。
> 两者一致：GitHub 仓库与 npm scope 都是 `jkin8010`。CI 用 **Trusted Publishing（OIDC）** 发布，不需要任何 token/secret。

## 一、一次性配置

### 1. npm 账号与 scope

- 包名：`@jkin8010/dsh-code-review`（scope = `@jkin8010`，`publishConfig.access=public`）
- 首次发布前确认 npm 账号拥有该 scope：`npm login` → `npm whoami` → `npm view @jkin8010/dsh-code-review`（首次应返回 404）
- 若首次发布仍报 `E404`/`E403`，先手动发布一次占位版本，或到 npm 站点创建 `@jkin8010` 组织：

```bash
npm login                                  # 浏览器授权
npm publish --access public                # 首次手动发布（也可直接靠 CI）
```

### 2. 登记 Trusted Publisher（OIDC，无需任何 token）

> **这是当前唯一可行的 CI 发布方式。** npm 的 Granular token 在 2026-07-31 起被限制：勾了 Bypass 2FA 的 token 也不能再直接 publish，会被要求交互式 OTP（`EOTP`）；2027-01 起彻底禁止。
> 结论：**别再用 token 发布**，改用 Trusted Publishing（OIDC）。

npm 站点 → 进入包页面 → **Settings** → **Trusted Publisher** → **GitHub Actions**，按下表填：

| 字段 | 填什么 |
|---|---|
| **Publisher** | `GitHub Actions` |
| **Organization or user** | `jkin8010` |
| **Repository** | `dsh-code-review` |
| **Workflow filename** | `release.yml`（**必须带 `.yml`，且大小写完全一致**） |
| **Environment name** | 留空（工作流里也没有设置 `environment`） |

要求：

- npm CLI **≥ 11.5.1**（工作流里用 `npm install -g npm@latest` 保证）；
- Node **≥ 22.14.0**；
- 工作流必须有 `permissions: id-token: write`（`release.yml` 已声明）；
- 只支持 GitHub 托管 runner，自托管 runner 不行。

### 3. GitHub 侧不需要 secret

OIDC 发布**不需要** `NPM_TOKEN`，也不需要任何 secret。仓库里已有的环境 `NPM_TOKEN` 与 secret 可以删掉，避免混淆。

> 工作流里仍然声明了 `id-token: write`，既用于 OIDC 换取发布凭据，也用于 npm provenance 签名。

## 二、日常发布

```bash
# 1. 改版本（package.json + client.js 的 PLUGIN_VERSION 要一起改）
npm version patch --no-git-tag-version   # 或 minor / major
# 2. 跑一遍全量测试
npm run test:full
# 3. 提交并打 tag
git add -A && git commit -m "chore: release vX.Y.Z"
git tag vX.Y.Z
git push origin main --tags
```

推送 tag 后，`.github/workflows/release.yml` 自动执行：

1. **verify**：校验 tag 去掉 `v` 后与 `package.json.version` 完全一致，不一致直接失败；
2. **test**：跑 `engine / delegate / tool / analyze` 四套离线自测 + `npm pack` 冒烟；
3. **publish**：`npm publish --access public --provenance`；
4. **release**：把 `npm pack` 出的 tarball 挂到 GitHub Release。

> 测试为什么只跑这四套：`host-selftest` 与 `git-ocr.test` 需要真实 `ocr` 二进制与真实 git 仓库，放在 `npm run test:full` 里本地跑；CI 里跑的是零外部依赖的那部分。

## 三、CI 工作流清单

| 文件 | 触发 | 作用 |
|---|---|---|
| `.github/workflows/ci.yml` | push 到 main / PR | 四套离线自测 + 打包冒烟（Node 20/22 矩阵） |
| `.github/workflows/release.yml` | push tag `v*` | 版本校验 → 测试 → npm 发布 → GitHub Release |

## 四、常见问题

| 现象 | 原因 | 处理 |
|---|---|---|
| `EOTP`（This operation requires a one-time password） | 用了 token 发布，而 npm 从 2026-07-31 起不再允许 token 直接 publish（即使勾了 Bypass 2FA） | 改用 **Trusted Publishing（OIDC）**：删掉 `NODE_AUTH_TOKEN`，在 npm 站点登记 trusted publisher |
| `ENEEDAUTH` / `Unable to authenticate` | OIDC 换不到凭据：workflow 文件名、仓库、环境名与 npm 站点登记不一致，或缺 `id-token: write` | 逐字核对 npm 站点上的 `Workflow filename`（必须 `release.yml`）与仓库名；确认 `permissions: id-token: write` |
| `E404` / `E403` on publish | 包不存在 / 无权限 / 被 npmmirror 拦 | 首次发布前该包需在 npm 上存在才能登记 trusted publisher；`publishConfig.registry` 必须指向 npmjs |
| `EPUBLISHCONFLICT` | 该版本已发布 | 升版本号重新打 tag |
| `verify` job 失败 | tag 与 package.json 版本不一致 | 改 package.json 版本，或重打 tag |
| `provenance` 报错 | 仓库是私有 / 无 `id-token` | 删掉 `--provenance`，或公开仓库 |
| 装到 DSH 后没生效 | 版本没变 | 升版本号重发，或 `plugin_manager` 重新 install |

## 五、本地验证（不发布）

```bash
npm pack --dry-run            # 看白名单：只打 host/ client.js index.js locale/ icon cordis.patch.yml README LICENSE
npm run test                  # 四套离线自测
npm run test:full             # 再加 host-selftest 与 git-ocr.test（需本机 ocr）
```