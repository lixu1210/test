
1.[[#一、UI点击迁移]]
2.[[#二、Git 镜像克隆 + 镜像推送（手动创建GitLab项目）]]
3.[[#三、提供仓库列表，进行循环迁移]]
4.[[#四、使用GitHub API自动获取仓库列表]]
5.[[#五、权限管理]]

**注：获取组织仓库的token可以是Owner的token，也可以是组织成员的token，但需要token有 read:org + repo 权限**

## 一、UI点击迁移
## github
**github密钥生成**

![](attachment/f7408db0e27a12c3e6a2bcef2658be8e.png)
**Developer setings**
![](attachment/70ce82cb28748b23fcb841d417c5f66d.png)
**选择密钥（Fine-grained tokens精细令牌,Tokens (classic)经典令牌）**
![](attachment/f675b951604c69891c31cab923c22906.png)
**选择要迁移的仓库**
![](attachment/df6db0fe54e0563493ae6cf17a381025.png)
**这两个权限保证对仓库的可读，（Administration可以读取仓库设置（如分支保护规则））**
![](attachment/76ad4fcb651d32413497d056250b36a5.png)



## gitlab
**点击项目**
![](attachment/afebd56d831d745fabec9884f12b3e82.png)

**添加项目**
![](attachment/e49cbb69d1e9ecd8a6ca61d1ef4c0e12.png)

**下面导入项目**
![](attachment/c44432124d48c948c37a4946aa1486f1.png)

**可以选择项目来源（设置里可以开放导入源，目前只添加了github）**
![](attachment/d6500e6395d6b2e26711de885b032d0a.png)

**输入github密钥（保证密钥权限）**
![](attachment/47194bdcedadd40fd4884d4bf2fea7d0.png)

**选择仓库导入**
![](attachment/bcf9862c79f0d1ce16a37c1040af9c30.png)


## 二、Git 镜像克隆 + 镜像推送（手动创建GitLab项目）
### 镜像同步

**使用 Git 镜像命令（推荐用于重要或大量仓库）**

这种方法通过命令行操作，能提供最完整、可控的迁移效果，尤其适合确保万无一失的场景。

1. **从GitHub克隆裸仓库**：在一台能同时访问GitHub和内网GitLab的机器上执行。`--mirror`参数会完整复制所有分支、标签和提交历史。
```shell
   git clone --mirror https://github.com/username/repository.git
   cd repository.git
```

2. **推送到GitLab**：首先在GitLab上创建一个空的**目标仓库**，然后执行推送。`--mirror`推送会确保所有引用都得到同步。
```shell
   git push --mirror https://your-internal-gitlab.com/group/project.git
```

#### **迁移后验证**

完成后，请务必在GitLab上检查：

- 所有分支是否都已存在。
    
- 提交历史是否完整（可以对比几个关键提交的Hash值）。
    
- 所有标签是否都已迁移。


## 三、提供仓库列表，进行循环迁移

```shell
#!/bin/bash
# migrate-repos.sh

GITHUB_ORG="your-org"
GITLAB_URL="https://gitlab.com"
GITLAB_GROUP="your-group"

# TOKEN
GITHUB_TOKEN="${GITHUB_TOKEN}"
GITLAB_TOKEN="${GITLAB_TOKEN}"

REPOS=("repo1" "repo2" "repo3" "frontend" "backend" "api-service")

for REPO in "${REPOS[@]}"; do
    echo "=== 迁移仓库: $REPO ==="

    git clone --mirror \
      "https://${GITHUB_TOKEN}@github.com/${GITHUB_ORG}/${REPO}.git"

    cd "${REPO}.git"

    git push --mirror \
      "https://oauth2:${GITLAB_TOKEN}@gitlab.com/${GITLAB_GROUP}/${REPO}.git"

    cd ..
    rm -rf "${REPO}.git"

    echo "✅ $REPO 迁移完成"
done

```

##  四、使用GitHub API自动获取仓库列表
**注：自动获取选择不了仓库**

```shell
#!/bin/bash
set -e

GITHUB_ORG="your-org"
GITLAB_URL="https://gitlab.com"
GITLAB_GROUP_ID="123"
GITLAB_GROUP_PATH="your-group"

GITHUB_TOKEN="${GITHUB_TOKEN}"
GITLAB_TOKEN="${GITLAB_TOKEN}"

REPOS_JSON=$(curl -s \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/orgs/${GITHUB_ORG}/repos?per_page=100&type=all")

echo "$REPOS_JSON" | jq -r '.[] | .name' | while read -r REPO_NAME; do
  echo "🚀 迁移 $REPO_NAME"

  # 创建 GitLab 项目（若不存在）
  curl -s -X POST \
    -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"name\":\"$REPO_NAME\",\"namespace_id\":$GITLAB_GROUP_ID}" \
    "$GITLAB_URL/api/v4/projects" > /dev/null || true

  # 克隆
  git clone --mirror \
    "https://${GITHUB_TOKEN}@github.com/${GITHUB_ORG}/${REPO_NAME}.git"

  cd "${REPO_NAME}.git"

  # 推送
  git push --mirror \
    "https://oauth2:${GITLAB_TOKEN}@gitlab.com/${GITLAB_GROUP_PATH}/${REPO_NAME}.git"

  cd ..
  rm -rf "${REPO_NAME}.git"

  echo "✅ $REPO_NAME 完成"
  sleep 1
done

```



**测试仓库列表**
获取个人仓库列表
```shell
curl -H "Authorization: token $token" \
     https://api.github.com/users/个人名字/repos | jq -r '.[].name' 
```

获取组织列表
```shell
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/orgs/组织名/repos \
  | jq -r '.[].name'
```


## 五、权限管理

创建group和subgroup，把Owner分给Group，就有了最高权限。把项目分给subgroup，把成员分给subgroup，成员就有了项目仓库的权限。
（给不同的subgroup Developer 权限，这是最小的权限，可以 push 代码、提交 MR、触发 pipeline，不能管理分支保护）



**权限介绍**

| 等级         | GitLab权限   | 描述           |
| ---------- | ---------- | ------------ |
| Guest      | Guest      | 只读访问         |
| Reporter   | Reporter   | 查看+创建issue   |
| Developer  | Developer  | 代码提交+合并请求    |
| Maintainer | Maintainer | 合并+保护分支+管理标签 |
| Owner      | Owner      | 完全控制         |
**能够进行提交代码、创建分支的最低权限为Developer**


**GitHub → GitLab 权限映射表**

| GitHub 权限  | GitLab 映射    | 说明                |
| ---------- | ------------ | ----------------- |
| Org Owner  | Group Owner  | 组织最高权限            |
| Repo Admin | Maintainer   | 项目级最高（不直接给 Owner） |
| Maintain   | Maintainer   | 基本等价              |
| Write      | Developer    | 日常开发              |
| Triage     | Reporter     | Issue / MR 管理     |
| Read       | **Reporter** | ⚠️ Guest 太弱       |
| 无权限        | No access    | 无访问               |


**组织对比**

| 对比项  | GitHub     | GitLab                     |
| ---- | ---------- | -------------------------- |
| 组织层级 | Org → Repo | Group → Subgroup → Project |
| 权限管理 | Team       | Group 继承                   |
| 子组   | ❌（无真正子组）   | ✅                          |
| 精细权限 | 强（Team）    | 强（继承）                      |



**权限回收**
```shell
curl --request DELETE --header "PRIVATE-TOKEN: $ADMIN_TOKEN" \
"https://gitlab.com/api/v4/groups/<group_id>/members/<user_id>"
```
后续可以写一个脚本，对权限自动化回收