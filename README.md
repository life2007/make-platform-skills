# make-platform-skills
make 平台的 skill

# 安装
```
  npx skills add qfeius/make-platform-skills
```
# 升级
```
  npx skills update  qfeius/make-platform-skills
```

## 可用 Skill 列表

### makecli
指导如何使用 `makecli` 命令行

#### 升级 skill
```bash
npx skills update makecli
```

**使用场景**
- 你需要指导使用 `makecli` 命令

### makedsl
指导如何生成 dsl 文件

#### 升级 skill
```bash
npx skills update makedsl
```

**使用场景**
- 根据业务的需求生成服务要求的 dsl 文件

### canvas-table-integration
指导如何在消费侧项目中接入 `@qfei-design/canvas-table`

#### 升级 skill
```bash
npx skills update canvas-table-integration
```

**使用场景**
- 在页面里接入 `@qfei-design/canvas-table`
- 接普通表格、分页表格、分组表格
- 把 JSON meta 转成 `IColumn[]`

### make-app-auth
指导 Make App 前端接入 `@qfeius/make-app-auth`，统一处理 token 模式、统一登录、ngrok、Cookie、logout 和 `/api/make/**` 401/403。

#### 升级 skill
```bash
npx skills update make-app-auth
```

**使用场景**
- Make App 前端需要接入统一登录或本地 token 调试
- 所有 Make 后端请求需要通过 `auth.api` 和共享 API adapter 统一处理
- 需要验证 `/api/make/**` 代理、401 重登、403 无权限和退出账号流程
- 使用 ngrok 或外部 HTTPS 域名测试 Org 统一登录回调
