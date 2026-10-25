# Codex Skills — 计量实证分析

本仓库包含两个用于计量经济学实证分析的 Codex skill：

| Skill | 说明 |
| --- | --- |
| [`econ-empirical`](econ-empirical) | 传统计量实证分析工作流：描述统计 → 基准回归 → 稳健性 → 内生性 → 异质性 → 机制 → 讨论交付。Stata 优先（本地 MCP），未连接时自动降级 Python 替身。 |
| [`dml-empirical`](dml-empirical) | 双重机器学习（DML）因果推断实证分析：基准 → 稳健性 → 内生性（PLIV）→ 异质性（BLP/GATES）→ 机制。Stata（ddml）优先，未连接时自动降级 Python（DoubleML + scikit-learn）。 |

## 安装

### 方式一：skill 安装器（推荐）

在 Codex 中让 skill 安装器从本仓库安装（把 `OWNER` 替换为你的 GitHub 用户名）：

```
install-skill-from-github.py --repo OWNER/codex-skills --path econ-empirical --path dml-empirical
```

### 方式二：手动复制

把 `econ-empirical/` 与 `dml-empirical/` 两个文件夹复制到 Codex 的 skills 目录：

- Windows：`C:\Users\<用户名>\.codex\skills\`
- macOS / Linux：`~/.codex/skills/`

安装完成后，新开一轮对话即可使用。

## 依赖

- 两个 skill 都优先使用 `stata` skill（本地 Stata MCP）真跑；Stata 未连接时自动降级为 Python，并在结果中标注「Python 替身」。
- `dml-empirical` 的 Python 路径依赖 `doubleml` 与 `scikit-learn`。

## 目录结构

- `econ-empirical/`：`SKILL.md` + `references/`（工作流与输入模板）
- `dml-empirical/`：`SKILL.md` + `references/`（工作流、输入、三线表、诊断与敏感性分析）
