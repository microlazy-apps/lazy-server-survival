# lazy-server-survival — 维护笔记

懒猫微服 wrapper for [pshenok/server-survival](https://github.com/pshenok/server-survival)。

## 仓库结构

```
lazy-server-survival/
├── vendor/server-survival/                  上游 git subtree（保持原样）
├── patches/
│   └── 01-lazycat-dockerfile.patch          新增 Dockerfile + nginx.conf
├── lazycat/
│   ├── package.template.yml
│   ├── lzc-manifest.template.yml
│   ├── lzc-build.yml
│   ├── appstore.yml
│   ├── icon.png                             占位（部署后换 UI 截图）
│   └── screenshots/
└── .github/workflows/
    ├── release.yml
    └── bootstrap-app.yml
```

## 架构

最简单的「single web process」模式：纯静态站点，nginx serve。

- nginx:1.27-alpine 镜像
- COPY index.html / style.css / game.js / src/ / assets/ / docs/
- nginx 监听 0.0.0.0 + [::]:80（dual-stack 防止 wget healthcheck 解析 ::1 失败）
- `/healthz` 200 ok 端点

无后端，无持久化，无 deploy params。游戏全部数据 in-memory，浏览器 reload 即重置。

## 上游 CDN 依赖

`index.html` 引用：
- `https://cdn.tailwindcss.com`
- `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js`

懒猫盒子需要能访问这两个 CDN。如果遇到 GFW 拦截 cdnjs，把 three.min.js 拉进 vendor + 改 patch 把 script src 替换为本地路径。

## Patch 流程

vendor 上游保持纯净，所有改动通过 `patches/01-lazycat-dockerfile.patch`。

新增/修改 patch：

```sh
git apply patches/01-lazycat-dockerfile.patch -p1 --directory=vendor/server-survival
# 编辑 vendor/server-survival/{Dockerfile,nginx.conf}
git add -N vendor/server-survival/<new-files>
git diff --no-color --relative=vendor/server-survival vendor/server-survival/ \
  > patches/01-lazycat-dockerfile.patch
git checkout HEAD -- vendor/server-survival/<modified-files>
rm vendor/server-survival/<new-files>
git apply --check patches/01-lazycat-dockerfile.patch -p1 --directory=vendor/server-survival
```

## 发布

第一次：

```sh
gh workflow run bootstrap-app.yml -F version=0.0.1
```

之后：

```sh
git tag v0.0.2 && git push origin v0.0.2
```

## 上游同步

```sh
git subtree pull --prefix=vendor/server-survival \
  https://github.com/pshenok/server-survival.git main --squash
git apply --check patches/01-lazycat-dockerfile.patch -p1 --directory=vendor/server-survival
```
