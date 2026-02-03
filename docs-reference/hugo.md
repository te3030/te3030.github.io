# Hugo 配置 GitHub Actions 实现自动化 GitHub Pages

# 添加 GitHub Pages

> 在项目中创建 `.github/workflows/hugo.yml`

```yaml
name: 推送到GitHub时，自动构建并部署到github pages
on:
  push:
    branches:
      - main

jobs:
  build:
    name: Deploy
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Deploy the site
        uses: benmatselby/hugo-deploy-gh-pages@main
        env:
          HUGO_VERSION: 0.146.7
          TARGET_REPO: te3030/te3030.github.io
          TARGET_BRANCH: main
          TOKEN: ${{ secrets.ACTION_ACCESS_TOKEN }}
          # HUGO_ARGS: '-t hextra'
          CNAME: te3030.github.io
```

# 添加 ACTION_ACCESS_TOKEN

- 打开(githun设置)[https://github.com/settings/tokens]
- 选择 Personal access tokens (classic)
- 选择 New personal access token (classic)
- 勾选 `repo` 和 `admin:repo_hook`
- 保存，得到 token
- 回到项目中 例子： https://github.com/youname/you_project/settings/secrets/actions
- 选择 Actions secrets and variables
- 在 Repository secrets 选择 New repository secret
- 填写 Name 为 `ACTION_ACCESS_TOKEN` ，Secret 在中添加上面获取的 token


