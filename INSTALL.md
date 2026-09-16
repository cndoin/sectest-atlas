# Installation / 安装说明

SecTest Atlas is a static handbook. It does not require Node.js packages, a database, or a build process.

SecTest Atlas 是纯静态手册，不需要安装 Node.js 依赖、数据库或构建工具。

## Option 1 — Read online / 在线阅读

Visit: <https://cndoin.github.io/sectest-atlas/>

## Option 2 — Clone and serve locally / 克隆并本地运行

Requirements: Git and either Python 3 or any static-file server.

```bash
git clone https://github.com/cndoin/sectest-atlas.git
cd sectest-atlas
python -m http.server 8080
```

Then open <http://localhost:8080>.

随后访问 <http://localhost:8080>。

## Windows PowerShell

```powershell
git clone https://github.com/cndoin/sectest-atlas.git
Set-Location sectest-atlas
python -m http.server 8080
```

If Python is unavailable, opening `index.html` directly still provides the handbook, search, checklist, theme, and print features.

如果电脑没有 Python，也可以直接双击 `index.html` 使用手册、搜索、勾选、主题和打印功能。

## Update / 更新

```bash
git pull --ff-only
```

Checklist progress and theme preference are stored in the current browser's local storage. Updating files does not normally erase them.

勾选进度与主题偏好保存在当前浏览器本地存储中，更新仓库通常不会清除这些数据。

## Uninstall / 卸载

Stop the local server and delete the cloned `sectest-atlas` directory. No system service or global package is installed.

停止本地服务后删除克隆的 `sectest-atlas` 文件夹即可；项目不会安装系统服务或全局软件包。
