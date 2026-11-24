# 上游同步指南（Upstream Sync）

## 一、远程仓库配置

```
origin    git@github.com:liaozhimingandy/ChatAIM.git      # 自己的仓库
upstream  https://github.com/LibreChat-AI/LibreChat.git   # 官方仓库
```

## 二、上游分支语义（官方 AGENTS.md 约定）

| 分支 | 用途 | 同步策略 |
|---|---|---|
| upstream/main | 发布分支 | ChatAIM `main` 保持 fast-forward 同步，不打开任何到 main 的 backport PR |
| upstream/dev | 开发分支（PR 合入目标） | ChatAIM `develop` 定期合并 upstream/dev |

关键规则：
- main 永远是 dev 的 fast-forward，合入 dev 的内容下次同步自动到达 main
- PR 默认目标是 main（`gh pr create` 的默认行为），**必须显式 `--base dev`**
- `Fixes #N` 只在默认分支生效，dev/canary 合并后需手动关闭 issue

## 三、标准同步流程（main 保持镜像）

```bash
git fetch upstream main dev
git checkout main
git merge --ff-only upstream/main     # 只允许 fast-forward
git push origin main
```

若 `--ff-only` 失败，说明 main 上出现了自有提交——不允许（main 应保持纯镜像），需将自有提交迁回 develop。

## 四、自定义开发同步流程（develop 吸收上游更新）

```bash
git fetch upstream
git checkout develop
git merge upstream/dev                # 或 rebase（若 develop 尚未共享）
# 解决冲突后：
npx tsc --noEmit                      # 在改过的每个 workspace 里跑
npm run static-checks -- --against upstream/dev
git push origin develop
```

**冲突处理原则：**
1. 官方文件以 upstream 为准，自有改动重建其上（保持修改可追踪、最小化）
2. 自有目录（`docs/also-ai/`、后续的 `chataim/` 等）无冲突风险
3. 大规模自定义前，先评估能否以"官方支持的配置/插件机制"实现（librechat.yaml、configSchema、接口注入），避免修改核心文件

## 五、减少未来冲突的纪律

1. **零核心修改优先**：能用配置（configSchema/librechat.yaml）和外围包实现的，不动 `api/`、`packages/api`、`client/src`
2. **自有代码独立成文件/目录**：新文件永远比改官方文件冲突概率低
3. **高频同步**：上游 dev 每天有数十个提交，建议每周同步一次，避免积压导致冲突成片
4. **PR 式自查**：同步后先在本地跑 `npm run static-checks` 与受影响 workspace 的 `tsc --noEmit`

## 六、版本回退

```bash
# 查看同步历史
git log --oneline main | head
# 回退 main 到某次同步点（仅在 push 前）
git reset --hard <sync-commit>
# 已 push：不强制重写远端，改用 revert
git revert -m 1 <merge-commit>
```

## 七、Worktree 与 Stash 警告（官方要求）

多个 worktree 共享一个 stash 栈，**禁止裸 `git stash pop`**。中断工作时优先 WIP commit；确需 stash，按标签精确 `git stash apply stash@{n}`。
