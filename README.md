# dsh-tool-encoding

DSH 编码/哈希工具插件 —— UTF-8 文本的 base64/base64url/url/hex 编解码 + 哈希 + UUID。零依赖、零进程、纯函数。

> 身份说明（AUDIT-ENC-01）：本仓库是 **source 仓库**，`package.json` 包名为 `@dsh-external/dsh-tool-encoding`（独立发布用）；插件 `name` 导出与 monorepo 集成后的包名统一为 `@deepseek-ai/dsh-tool-encoding`（集成用）。`main`/`types` 指向的 `lib/` 产物由 DSH monorepo 顶层构建生成，本仓库不内置 build 脚本。

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

## 动机

Agent 处理编码/哈希是日常高频操作：查看 API 响应里的 base64 字段（JWT payload）、构造 query 参数、校验文件完整性（sha256）、生成 UUID。当前做法 `bash -c "echo ... | base64"` 的问题：进程开销、**引号转义地狱**（base64 串嵌进 bash 命令再嵌进 JSON 参数，双层转义错误率极高）、跨平台工具命名不一致（`md5` vs `md5sum` vs `shasum -a 256`）。

## 安全模型

无 `eval`、无 `new Function`。所有操作是 `Buffer`/`node:crypto`/`TextDecoder`/`encodeURIComponent` 的纯函数组合：

- **UTF-8 完整性**：解码用 fatal 模式（`TextDecoder('utf-8', { fatal: true })`），非法字节抛 `encoding: invalid UTF-8 output`；合法 U+FFFD/控制字符不误伤（`00` NUL、`0a` 换行、`efbfbd` U+FFFD 均合法，`ff` 非法）
- **base64 严格校验**：无空白、`=` 仅末尾且 ≤2、长度必须为 4 的倍数、**RFC 4648 canonical unused bits**（`Zh==` 等非 canonical 编码拒绝，防多串映射同文本）
- **孤立 surrogate 统一拒绝**：所有文本输入过 `String.prototype.isWellFormed()`，避免静默替换与 URIError 行为不一致
- **字节上限**：输入 1 MB / 输出 4 MB（各 1,000,000 / 4,000,000 字节，`Buffer.byteLength`；输出上限为分配前预估 + 最终检查的保险丝）
- **哈希算法白名单**：`Object.hasOwn` 查表（md5/sha1/sha256/sha512）
- 错误统一 `encoding:` 前缀，不透传底层 `URIError`/`TypeError`

## 架构

```
DSH Agent
    │ ctx.tools.register()
    ▼
src/index.ts（Cordis 插件入口 + action 分发 + 独立校验）
    │
    ▼
src/encoding.ts
    ├── b64Encode/b64Decode — 标准 base64（严格校验）
    ├── b64UrlEncode/b64UrlDecode — base64url（canonical 无 padding，解码兼容）
    ├── urlEncode/urlDecode — component 语义（URIError 包装）
    ├── hexEncode/hexDecode — UTF-8 字节 hex（fatal 解码）
    ├── digest — 白名单哈希
    ├── newUuid — crypto.randomUUID()
    └── validateUnicode/assertInputBytes/decodeUtf8Strict — 校验器
```

## 工具声明

```ts
ctx.tools.register(defineTool({
  name: 'encoding',
  parameters: {
    action: {
      type: 'string', required: true,
      enum: ['base64_encode','base64_decode','base64url_encode','base64url_decode',
             'url_encode','url_decode','hex_encode','hex_decode','hash','uuid'],
    },
    input:     { type: 'string', description: 'Input string for encode/decode/hash' },
    algorithm: { type: 'string', enum: ['md5','sha1','sha256','sha512'], description: 'For hash' },
  },
  output: { schema: { type: 'json' }, render: (_a, v) => [{ type: 'text', text: JSON.stringify(v) }] },
  execute: (args) => Promise.resolve(executeAction(args.action, args) as JsonValue),
  timeoutMs: 1000,
}))
```

## 支持的操作

| action | 说明 | 示例 |
|--------|------|------|
| `base64_encode` / `base64_decode` | RFC 4648 标准 base64（严格校验） | `"foobar"` → `"Zm9vYmFy"` |
| `base64url_encode` / `base64url_decode` | JWT 风格，输出无 padding，解码兼容 `+/` 与可选 padding | `"\uFEFF"` → `"77u_"` |
| `url_encode` / `url_decode` | **component 语义**（非完整 query）：空格 `%20` 非 `+`，`!'()*` 不转义，`decode("+")` → `"+"` | `"a b"` → `"a%20b"` |
| `hex_encode` / `hex_decode` | UTF-8 字节 hex；解码结果必须合法 UTF-8 | `"AB"` → `"4142"` |
| `hash` | md5/sha1/sha256/sha512 hex 摘要；**仅非安全用途** | `sha256("")` → `e3b0c442...` |
| `uuid` | UUID v4 字符串（`crypto.randomUUID()`） | `"550e8400-..."` |

语义契约：

- **v1 是 UTF-8 文本工具**：所有输入输出为字符串；二进制内容用 hex 表示；解码结果必须合法 UTF-8
- **所有 action 返回字符串**（含 `uuid`）
- **不要对机密材料使用本工具**：tool 参数会记录进会话日志

## 接入方式

1. 放入 monorepo：`cp -r dsh-tool-encoding ~/.dsh/source/master/packages/tools/dsh-tool-encoding`
2. `apps/cli/package.json` 加 `"@deepseek-ai/dsh-tool-encoding": "workspace:^"`
3. `apps/cli/config/base.cordis.yml` 加 `- id: tool-encoding / name: '@deepseek-ai/dsh-tool-encoding'`
4. `tsconfig.host.json` references 加 `{ "path": "./packages/tools/dsh-tool-encoding" }`
5. `pnpm install && pnpm test` 本地验证；集成后由 DSH monorepo 根 `pnpm run build` 构建，再 `dsh -p "把 foobar 做 base64"` 验证

## 已知限制

1. 分发链路：`@deepseek-ai/dsh-tools` 私有，需 monorepo workspace
2. 任意二进制（不可打印字节）编解码需 v2 的 `output: "utf8" | "hex"` 模式
3. hash 仅摘要，无 HMAC/加盐/密钥派生；MD5/SHA-1 仅兼容性/非安全完整性校验
4. URL 是 component 语义；表单编码（空格 → `+`）需独立 action（v2）

## 测试

```bash
pnpm test
```

功能/错误/攻击载荷共 41 个用例（RFC 4648/1321/6234 已知向量、UTF-8 边界区分、canonical padding 边界、surrogate 统一拒绝等）。完整清单见本地维护的设计文档。

## 许可

MIT
