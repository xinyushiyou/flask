# dbapp — Flask 后端模板

一个基于 Flask + SQLAlchemy + Flask-Migrate 的后端示例项目，内置模块化蓝图、统一异常/日志处理、文件上传、会话登录（Cookie + Session）与示例模块 example（含模型、CRUD、分页筛选、关联查询、文件上传）。

## 环境要求
- Python 3.12+
- Windows（示例命令基于 PowerShell）

## 快速开始

1) 安装依赖
```powershell
pip install -r requirements.txt
```

2) 初始化数据库
```powershell
flask db init
flask db migrate -m "init"
flask db upgrade
```
- 如修改了模型，需要生成迁移并升级：
```powershell
flask db migrate -m "update models"
flask db upgrade
```

3) 启动服务
```powershell
flask run
```
说明：根目录已提供 .flaskenv，Flask 会读取 FLASK_APP=run.py 和 FLASK_DEBUG=1。

## 代码结构
```
config.py                 # 配置（数据库、SECRET_KEY、UPLOAD_FOLDER 等）
run.py                    # Flask 入口（create_app 工厂）
requirements.txt          # 依赖
app/
  __init__.py             # 应用工厂、注册蓝图、初始化扩展、创建上传目录
  extensions.py           # db、migrate 实例
  modules/
    __init__.py           # 统一注册模块蓝图（/api/auth, /api/example）
    auth/                 # 认证模块（Cookie + Session）
    example/              # 示例模块（模板）
  utils/
    decorators.py         # login_required、permission_required 装饰器
    exception_handler.py  # 统一异常处理
    response_handler.py   # 统一成功响应封装
    log_handler.py        # 日志初始化（logs/app.log）
logs/
  app.log                 # 运行日志
migrations/               # Flask-Migrate 迁移目录
uploads/                  # 上传目录（运行时自动创建）
```

## 认证方式
- 采用 Cookie + Session：登录成功后设置 session['user_id']，浏览器自动携带 Cookie 访问受保护接口。
- 受保护接口使用 @login_required；资源操作再叠加 @permission_required 校验资源所有者。

## example 模块（模板）说明

模型（app/modules/example/models.py）
- ExampleCategory(id, name) — 分类
- ExampleItem(id, name, description, timestamp, file_url, user_id, category_id)
  - 关系：category <-> items（一对多），author <-> example_items（用户与项目一对多）
  - to_dict() 用于 API 序列化，包含 author_username、category_name 等关联信息

主要接口（均挂载在 /api/example）
- 分类
  - POST /category/create（登录）
    - body: { "name": "分类名" }
  - GET  /category/list
- 项目
  - POST /item/create（登录）
    - body: { "name": "标题", "description": "描述", "category_id": 1 }
  - GET  /item/<id>
  - POST /item/update（登录 + 权限）
    - body: { "id": 1, "name"?: "新标题", "description"?: "新描述", "category_id"?: 2 }
    - 说明：permission_required 要求 body 中携带 id；会校验资源 user_id == 当前用户
  - POST /item/delete（登录 + 权限）
    - body: { "id": 1 }
  - GET  /item/list?search=&category_id=&user_id=&page=1&per_page=10
    - 支持名称/描述模糊搜索、分类/用户过滤、分页、按时间倒序
  - POST /item/upload-file（登录）
    - form-data: item_id=<id>, file=<文件>
    - 支持扩展名：.jpg/.jpeg/.png/.gif/.pdf/.txt
    - 成功后会写入 item.file_url，并可通过 GET /uploads/<filename> 访问

示例调用顺序（Postman/前端）
1. 注册 -> POST /api/auth/register {username,password}
2. 登录 -> POST /api/auth/login {username,password}
3. 创建分类 -> POST /api/example/category/create {name}
4. 创建项目 -> POST /api/example/item/create {name,description,category_id}
5. 更新/删除项目 -> 携带会话 Cookie，POST /api/example/item/update|delete

## 常见问题
- 405 Method Not Allowed：确认方法是否匹配。当前更新/删除接口使用 POST。
- 数据库变更未生效：生成迁移并升级（flask db migrate + flask db upgrade）。
- 无法访问上传文件：确保 uploads 目录存在（应用启动会自动创建）。

## 开发提示
- 修改模型后记得迁移（migrate/upgrade）。
- 受保护接口需先登录，前端需保持 Cookie（CORS 已开启 supports_credentials=True）。

---
如需基于 example 扩展新模块，复用其 models/routes 的结构与注释，即可快速搭建包含模型、CRUD、分页、文件上传与权限校验的完整业务骨架。
