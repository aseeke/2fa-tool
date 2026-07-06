# 2FA / MFA 验证码

一个纯前端的 TOTP 工具，支持批量输入 `secret` 或 `otpauth://` 链接，适配 Oracle、AWS、Google Authenticator 等常见 2FA / MFA 场景。

## 特点

- 仅在浏览器内计算，不保存密钥到数据库
- 支持 Base32 `secret`
- 支持 `otpauth://totp/...` 链接
- 支持多行批量输入
- 右上角显示北京时间
- 默认折叠的 API 示例区，方便后续接后端

## 本地开发

```bash
npm install
npm run dev
```

## 构建

```bash
npm run build
```

构建产物输出到 `dist/`。

## API 示例

页面里的 `API` 折叠区展示的是接口参数格式示例，不代表当前仓库已经包含后端服务。如果你后面要接真实 API，可以直接沿用这些参数约定。
