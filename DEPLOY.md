# 同学聚会网站 · 免费部署指南

## 文件结构

```
ClassReunion/
├── index.html          # 主页面（单文件包含所有内容）
├── photos/             # 20张老照片
├── content.json        # 内容数据（供脚本使用）
├── gen_html.py         # HTML生成脚本（如需修改内容重生成）
└── DEPLOY.md           # 本文件
```

## 方式一：GitHub Pages（推荐，全球可访问）

1. 注册 GitHub 账号：https://github.com
2. 新建仓库（Repository），名称任意，如 `class-reunion-2026`
3. 上传文件：
   ```bash
   git init
   git add index.html photos/
   git commit -m "Initial: class reunion page"
   git branch -M main
   git remote add origin https://github.com/你的用户名/class-reunion-2026.git
   git push -u origin main
   ```
4. 在仓库 Settings → Pages 中，选择分支 `main`，点 Save
5. 稍等1分钟，访问 `https://你的用户名.github.io/class-reunion-2026/`

## 方式二：Vercel（更快，推荐）

1. 注册 Vercel 账号：https://vercel.com （用 GitHub 登录）
2. 点击 "New Project" → 选择你的 GitHub 仓库
3. 直接点 Deploy，30秒完成
4. 获得域名如 `class-reunion.vercel.app`

## 方式三：Netlify（拖拽部署，最简单）

1. 打开 https://app.netlify.com
2. 注册后，直接把 `ClassReunion` 文件夹拖到页面上
3. 自动部署完成，获得 `xxx.netlify.app` 域名
4. 可在设置中修改子域名

## 方式四：直接分享（零门槛）

如果不想部署网站，可以直接发文件：

1. 把整个文件夹打成 zip：
   ```
   右键 ClassReunion 文件夹 → 发送到 → 压缩文件夹
   ```
2. 通过百度网盘/阿里云盘分享 zip 文件
3. 对方下载后解压，双击 `index.html` 即可浏览器查看

## 方式五：Gitee Pages（国内访问快）

1. 注册 Gitee 账号：https://gitee.com
2. 新建仓库，上传 index.html 和 photos/
3. 在 服务 → Gitee Pages 中开启
4. 获得 `xxx.gitee.io` 域名（国内访问速度快）

## 自定义域名

以上所有服务都支持绑定自己的域名（如果有的话），在设置中添加 CNAME 记录即可。

## 注意事项

- 照片文件较大（约4MB），首次加载可能需要几秒
- 建议在部署前用 https://tinypng.com 压缩照片，加快加载速度
- 手机端已适配，可以微信直接发链接打开
