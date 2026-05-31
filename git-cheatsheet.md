# Git 常用命令速查表

日常开发中最常用的 Git 命令整理。

---

## 分支操作

```bash
# 创建并切换分支
git checkout -b feature/xxx

# 查看所有分支
git branch -a

# 删除本地分支
git branch -d feature/xxx

# 删除远程分支
git push origin --delete feature/xxx
```

## 提交操作

```bash
# 暂存所有更改
git add .

# 提交
git commit -m "feat: 描述"

# 修改上一次提交
git commit --amend

# 查看提交历史
git log --oneline -10
```

## 撤销操作

```bash
# 撤销工作区更改
git checkout -- <file>

# 撤销暂存
git reset HEAD <file>

# 回退到某个提交
git reset --hard <commit-hash>
```

## 远程操作

```bash
# 拉取最新代码
git pull origin main

# 推送
git push origin main

# 强制推送（谨慎使用）
git push --force-with-lease
```

---

## 常用别名配置

```bash
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
```
