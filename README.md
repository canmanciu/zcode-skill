# zcode-skill

个人 ZCode skills（用户级，`~/.zcode/skills/`），私有仓库，多机同步用。

## 包含的 skills

| Skill | 用途 |
|---|---|
| `memos-knowledge-base` | 读写 note.canmanciu.top（Memos）上的笔记/博客，走 REST API |

## 新机器安装

```bash
# 1. clone 到本地（位置自选，这里以 ~/Personal/workspace 为例）
git clone git@github.com:canmanciu/zcode-skill.git ~/Personal/workspace/zcode-skill

# 2. 软链 skill 到 ZCode 用户级目录
mkdir -p ~/.zcode/skills
ln -s ~/Personal/workspace/zcode-skill/memos-knowledge-base ~/.zcode/skills/memos-knowledge-base

# 3. 配置 Memos token（不要提交到本仓库）
#    方式 A：复用已有 token（从旧机器 rsync ~/.memos.conf 过来）
#    方式 B：新申请 —— 登录 https://note.canmanciu.top → Settings → Access Tokens
echo 'MEMOS_TOKEN="<your-token>"' > ~/.memos.conf
chmod 600 ~/.memos.conf

# 4. 依赖：curl 自带，jq 需要安装
brew install jq

# 5. 重启 ZCode 会话，说「查一下我笔记里关于 X 的内容」验证触发
```

## 维护约定

- 改 skill 直接改本仓库里的文件（软链指向这里），改完 commit + push。
- 新增 skill：在本仓库建目录，然后同样软链到 `~/.zcode/skills/<name>`。
- **任何 token / 密钥都不进本仓库**，统一放 `~/.memos.conf`（600）或环境变量。
