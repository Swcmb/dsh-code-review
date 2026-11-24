# 发布流程（npm + GitHub Actions）

> 打 tag 即自动发布：`git tag vX.Y.Z && git push origin vX.Y.Z` → GitHub Actions 跑测试 → 发布到 npm → 建 GitHub Release。

> 仓库归属：`github.com/jkin8010/dsh-code-review`（GitHub 个人仓库，不变）；npm scope：`@jkin8010`（npm 组织）。
> 两者一致：GitHub 仓库与 npm scope 都是 `jkin8010`，CI 的 `NPM_TOKEN` 需属于 `@jkin8010` 组织成员。

## 一、一次性配置

### 1. npm 账号与 scope

- 包名：`@jkin8010/dsh-code-review`（scope = `@jkin8010`，`publishConfig.access=public`）
- 首次发布前确认 npm 账号拥有该 scope：`npm login` → `npm whoami` → `npm view @jkin8010/dsh-code-review`（首次应返回 404）
- 若首次发布仍报 `E404`/`E403`，先手动发布一次占位版本，或到 npm 站点创建 `@jkin8010` 组织：

```bash
npm login                                  # 浏览器授权
npm publish --access public                # 首次手动发布（也可直接靠 CI）
```

### 2. 生成 npm Automation Token

npm 站点 → Access Tokens → Generate New Token → **Automation**（CI 用，不需要 OTP）。

> 注意：Automation token 只对 **npmjs.org** 有效，npmmirror 镜像不可发布。

### 3. 写进 GitHub Secrets

仓库 → Settings → Secrets and variables → Actions → New repository secret：

| Name | Value |
|---|---|
| `NPM_TOKEN` | 上一步生成的 Automation token |

> 工作流里也声明了 `id-token: write`，用于 npm provenance（OIDC 签名）。若不想用 provenance，删掉 `--provenance` 即可。
> 若改用 npm Trusted Publishing（OIDC，无需 token），可去掉 `NODE_AUTH_TOKEN`，并把 `environment: npm` 在 npm 站点登记为 trusted publisher。

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
| `E404` / `E403` on publish | token 无权限 / scope 不存在 / 被 npmmirror 拦 | 确认 token 是 npmjs 的 Automation token，且 `publishConfig.registry` 指向 npmjs |
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