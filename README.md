# 2FA / MFA 生成器

一个纯前端的 TOTP 验证码工具，支持直接粘贴 `secret` 或 `otpauth://` 链接。

## 特点

- 本地浏览器内完成计算，不需要后端
- 支持常见身份验证器格式
- 带实时倒计时和自动刷新
- 适合直接部署到 Cloudflare Pages

## 本地开发

```bash
npm install
npm run dev
```

## 构建

```bash
npm run build
```

构建产物会输出到 `dist/`。

## Cloudflare Pages

- Build command: `npm run build`
- Build output directory: `dist`
- Framework preset: `Vite`

如果你后面想加自定义域名或路由，我也可以继续帮你补。
