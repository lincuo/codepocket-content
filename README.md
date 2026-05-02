# CodePocket Content

这是 CodePocket 小程序的远程每日精选内容仓库。

## 文件结构

```text
manifest.json
daily/
  2026-05-02.json
docs/
  content-format.md
  hermes-github-sync.md
```

## 小程序配置

发布到 GitHub Pages 后，在小程序中填写：

```text
https://<your-name>.github.io/codepocket-content/manifest.json
```

入口：

```text
我的 → 内容源配置 → Manifest URL
```

## 更新规则

- 每天新增一个 `daily/YYYY-MM-DD.json`。
- 更新 `manifest.json` 的 `latest`。
- 在 `manifest.json.files` 中追加当天日期。
- 不要删除历史日期。

详细规则见 `docs/content-format.md`。
