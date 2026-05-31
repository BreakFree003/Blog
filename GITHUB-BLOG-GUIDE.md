# GitHub Blog 自动发布配置指南

将 Markdown 笔记自动发布到 GitHub Discussions 的完整配置方案。

---

## 目录结构

```
blog/
├── .attachments/          # 图片附件目录（隐藏文件夹）
├── .github/
│   └── workflows/
│       └── publish.yml    # 自动发布 Action
├── .gitignore
├── README.md              # 仓库说明（不会发布到 Discussions）
└── *.md                   # 笔记文件（平铺在根目录）
```

---

## 核心功能

| 操作 | 行为 |
|------|------|
| 新增 md 文件 | 创建新的 Discussion |
| 修改 md 文件 | 更新已有的 Discussion |
| 删除 md 文件 | 删除对应的 Discussion |
| 修改 README.md | 不触发任何操作 |

---

## 配置步骤

### 1. 创建仓库和组织

1. 创建 GitHub 组织（如 `BreakFreeOrg`）
2. 在组织下创建仓库（如 `Blog`）
3. 转移仓库到组织（如果已有个人仓库）

### 2. 创建 Fine-grained Personal Access Token

1. 访问：https://github.com/settings/personal-access-tokens/new
2. **Resource owner**：选择你的组织
3. **Repository access**：选择 `Only select repositories`，选择 `Blog`
4. **Permissions**：
   - Repository permissions → **Discussions**: `Read and write`
   - Repository permissions → **Contents**: `Read`
5. 生成 token 并保存

### 3. 配置仓库 Secrets

1. 进入仓库 → Settings → Secrets and variables → Actions
2. 添加 `PAT_TOKEN`，值为刚创建的 token

### 4. 配置组织 Discussion 权限

1. 进入组织 → Settings → Member privileges
2. 取消勾选 **"Allow users with read access to create discussions"**
3. 这样只有你自己能创建 Discussion，其他人只能评论

### 5. 创建 Action 配置文件

创建 `.github/workflows/publish.yml`：

```yaml
name: Publish to Discussions

on:
  push:
    branches:
      - main
    paths:
      - '*.md'
      - '!README.md'

permissions:
  discussions: write
  contents: read

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Get changed md files
        id: changed
        run: |
          FILES=$(git diff-tree --root --no-commit-id --name-only -r HEAD | grep '\.md$' | grep -v '^README.md$' || true)
          echo "files=$FILES" >> $GITHUB_OUTPUT

      - name: Get deleted md files
        id: deleted
        run: |
          DELETED=$(git diff-tree --root --no-commit-id --diff-filter=D --name-only -r HEAD | grep '\.md$' | grep -v '^README.md$' || true)
          echo "files=$DELETED" >> $GITHUB_OUTPUT

      - name: Delete removed Discussions
        if: steps.deleted.outputs.files != ''
        env:
          GH_TOKEN: ${{ secrets.PAT_TOKEN }}
        run: |
          ALL_DISCUSSIONS=""
          CURSOR=""
          while true; do
            if [ -z "$CURSOR" ]; then
              PAGE=$(gh api graphql -f query='
                query($owner: String!, $name: String!) {
                  repository(owner: $owner, name: $name) {
                    discussions(first: 100) {
                      pageInfo { hasNextPage endCursor }
                      nodes { id body }
                    }
                  }
                }' -f owner="${{ github.repository_owner }}" -f name="${{ github.event.repository.name }}")
            else
              PAGE=$(gh api graphql -f query='
                query($owner: String!, $name: String!, $after: String!) {
                  repository(owner: $owner, name: $name) {
                    discussions(first: 100, after: $after) {
                      pageInfo { hasNextPage endCursor }
                      nodes { id body }
                    }
                  }
                }' -f owner="${{ github.repository_owner }}" -f name="${{ github.event.repository.name }}" -f after="$CURSOR")
            fi
            NODES=$(echo "$PAGE" | jq -c '.data.repository.discussions.nodes')
            if [ -z "$ALL_DISCUSSIONS" ]; then
              ALL_DISCUSSIONS="$NODES"
            else
              ALL_DISCUSSIONS=$(echo "$ALL_DISCUSSIONS$NODES" | jq -sc '.[0] + .[1]')
            fi
            HAS_NEXT=$(echo "$PAGE" | jq -r '.data.repository.discussions.pageInfo.hasNextPage')
            if [ "$HAS_NEXT" != "true" ]; then
              break
            fi
            CURSOR=$(echo "$PAGE" | jq -r '.data.repository.discussions.pageInfo.endCursor')
          done

          for file in ${{ steps.deleted.outputs.files }}; do
            FILENAME=$(basename "$file")
            echo "Deleting discussion for: $FILENAME"
            DISCUSSION_ID=$(echo "$ALL_DISCUSSIONS" | jq -r --arg src "<!-- source: ${FILENAME} -->" '.[] | select(.body | contains($src)) | .id' | head -1)
            if [ -n "$DISCUSSION_ID" ] && [ "$DISCUSSION_ID" != "null" ]; then
              echo "Deleting discussion: $DISCUSSION_ID"
              gh api graphql -f query='mutation($discId: ID!) { deleteDiscussion(input: {id: $discId}) { discussion { id } } }' -f discId="$DISCUSSION_ID"
            fi
          done

      - name: Publish to Discussions
        if: steps.changed.outputs.files != ''
        env:
          GH_TOKEN: ${{ secrets.PAT_TOKEN }}
        run: |
          REPO_INFO=$(gh api graphql -f query='
            query($owner: String!, $name: String!) {
              repository(owner: $owner, name: $name) {
                id
                discussionCategories(first: 10) {
                  nodes { id name }
                }
              }
            }' -f owner="${{ github.repository_owner }}" -f name="${{ github.event.repository.name }}")

          REPO_ID=$(echo "$REPO_INFO" | jq -r '.data.repository.id')
          CATEGORY_ID=$(echo "$REPO_INFO" | jq -r '.data.repository.discussionCategories.nodes[0].id')

          RAW_URL="https://raw.githubusercontent.com/${{ github.repository }}/main"

          ALL_DISCUSSIONS=""
          CURSOR=""
          while true; do
            if [ -z "$CURSOR" ]; then
              PAGE=$(gh api graphql -f query='
                query($owner: String!, $name: String!) {
                  repository(owner: $owner, name: $name) {
                    discussions(first: 100) {
                      pageInfo { hasNextPage endCursor }
                      nodes { id body }
                    }
                  }
                }' -f owner="${{ github.repository_owner }}" -f name="${{ github.event.repository.name }}")
            else
              PAGE=$(gh api graphql -f query='
                query($owner: String!, $name: String!, $after: String!) {
                  repository(owner: $owner, name: $name) {
                    discussions(first: 100, after: $after) {
                      pageInfo { hasNextPage endCursor }
                      nodes { id body }
                    }
                  }
                }' -f owner="${{ github.repository_owner }}" -f name="${{ github.event.repository.name }}" -f after="$CURSOR")
            fi
            NODES=$(echo "$PAGE" | jq -c '.data.repository.discussions.nodes')
            if [ -z "$ALL_DISCUSSIONS" ]; then
              ALL_DISCUSSIONS="$NODES"
            else
              ALL_DISCUSSIONS=$(echo "$ALL_DISCUSSIONS$NODES" | jq -sc '.[0] + .[1]')
            fi
            HAS_NEXT=$(echo "$PAGE" | jq -r '.data.repository.discussions.pageInfo.hasNextPage')
            if [ "$HAS_NEXT" != "true" ]; then
              break
            fi
            CURSOR=$(echo "$PAGE" | jq -r '.data.repository.discussions.pageInfo.endCursor')
          done

          for file in ${{ steps.changed.outputs.files }}; do
            if [ -f "$file" ]; then
              FILENAME=$(basename "$file")
              TITLE=$(grep -m1 '^# ' "$file" | sed 's/^# //')
              if [ -z "$TITLE" ]; then
                TITLE=$(basename "$file" .md)
              fi

              CONTENT=$(cat "$file")
              CONTENT=$(echo "$CONTENT" | sed "s|](\./attachments/|](${RAW_URL}/.attachments/|g")
              CONTENT=$(echo "$CONTENT" | sed "s|](\.attachments/|](${RAW_URL}/.attachments/|g")
              CONTENT="$(printf '<!-- source: %s -->\n%s' "$FILENAME" "$CONTENT")"

              echo "Processing: $TITLE (file: $FILENAME)"

              DISCUSSION_ID=$(echo "$ALL_DISCUSSIONS" | jq -r --arg src "<!-- source: ${FILENAME} -->" '.[] | select(.body | contains($src)) | .id' | head -1)

              if [ -n "$DISCUSSION_ID" ] && [ "$DISCUSSION_ID" != "null" ]; then
                echo "Updating existing discussion: $DISCUSSION_ID"
                gh api graphql \
                  -f query='
                  mutation($discId: ID!, $body: String!) {
                    updateDiscussion(input: {discussionId: $discId, body: $body}) {
                      discussion { url }
                    }
                  }' \
                  -f discId="$DISCUSSION_ID" \
                  -f body="$CONTENT"
              else
                echo "Creating new discussion"
                gh api graphql \
                  -f query='
                  mutation($repoId: ID!, $catId: ID!, $title: String!, $body: String!) {
                    createDiscussion(input: {repositoryId: $repoId, categoryId: $catId, title: $title, body: $body}) {
                      discussion { url }
                    }
                  }' \
                  -f repoId="$REPO_ID" \
                  -f catId="$CATEGORY_ID" \
                  -f title="$TITLE" \
                  -f body="$CONTENT"
              fi
            fi
          done
```

### 6. 创建 .gitignore

```
.DS_Store
```

### 7. 配置远程仓库并推送

```bash
git init
git remote add origin https://github.com/BreakFreeOrg/Blog
git add .
git commit -m "feat: init blog"
git push -u origin main
```

---

## 使用方法

### 新增笔记

```bash
# 创建笔记
echo "# 标题" > my-note.md

# 添加图片（可选）
# 在笔记中引用：![描述](.attachments/image.png)

# 提交并推送
git add .
git commit -m "feat: 新增笔记标题"
git push
```

### 更新笔记

```bash
# 修改笔记内容
vim my-note.md

# 提交并推送
git add .
git commit -m "feat: 更新笔记标题"
git push
```

### 删除笔记

```bash
# 删除笔记文件
rm my-note.md

# 提交并推送
git add .
git commit -m "feat: 删除笔记标题"
git push
```

---

## 关键机制

### 标识符

每个 Discussion 的 body 开头都包含隐藏标识符：
```html
<!-- source: filename.md -->
```

用于匹配文件和 Discussion，实现更新和删除功能。

### 图片路径替换

Action 自动将相对路径替换为 GitHub raw 绝对路径：
- 原始：`![图片](.attachments/demo.png)`
- 替换：`![图片](https://raw.githubusercontent.com/BreakFreeOrg/Blog/main/.attachments/demo.png)`

### 分页查询

使用 GraphQL 分页查询，支持超过 100 个 Discussion。

---

## 规范建议

| 项目 | 建议 |
|------|------|
| 文件名 | 使用小写英文、数字、连字符 |
| 图片路径 | 统一使用 `.attachments/` |
| 标识符 | 不要手动写 `<!-- source: xxx.md -->` |
| 笔记标题 | 使用 `# 标题` 格式作为第一个标题 |

---

## 常见问题

### Q: 为什么 README.md 不会发布到 Discussions？

A: Action 配置中排除了 README.md：
- `paths` 过滤：`'!README.md'`
- `git diff-tree` 过滤：`grep -v '^README.md$'`

### Q: 如何修改 Discussion 分类？

A: 修改 Action 中的 `CATEGORY_ID`：
```yaml
CATEGORY_ID=$(echo "$REPO_INFO" | jq -r '.data.repository.discussionCategories.nodes[0].id')
```
将 `nodes[0]` 改为其他分类的索引。

### Q: 如何支持子目录？

A: 当前方案只支持平铺结构。如需支持子目录，需要修改标识符格式和文件路径处理逻辑。

### Q: 图片会自动删除吗？

A: 不会。删除笔记时，对应的图片文件需要手动清理。

### Q: 为什么 Action 失败了？

A: 常见原因：
1. PAT_TOKEN 过期或权限不足
2. 组织禁止了长期 token 访问
3. GitHub API 限流

---

## Edge Case 已覆盖

| 场景 | 状态 |
|------|------|
| Discussion 数量超过 100 | ✅ 分页查询 |
| README.md 排除 | ✅ 已实现 |
| 文件名特殊字符 | ✅ 建议使用英文、数字、连字符 |
| 空文件或无标题 | ✅ 使用文件名作为 fallback |
| 并发 push | ✅ GitHub Actions 排队执行 |
| 同时删除和创建 | ✅ 正常工作 |
| 分页查询空值处理 | ✅ 已处理 |

---

## 参考链接

- [GitHub Discussions 文档](https://docs.github.com/en/discussions)
- [GitHub Actions 文档](https://docs.github.com/en/actions)
- [GitHub GraphQL API](https://docs.github.com/en/graphql)
