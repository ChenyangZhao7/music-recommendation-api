# 音乐推荐 API

[English](README.md) | **中文**

一个数据驱动的音乐发现与个性化推荐 RESTful API，用 FastAPI + SQLite 构建。

## 技术栈

| 层 | 选择 |
| --- | --- |
| 框架 | FastAPI |
| 数据库 | SQLite，经 SQLAlchemy 2.x ORM |
| 认证 | JWT（PyJWT）+ bcrypt 密码哈希 |
| 数据集 | Spotify Tracks Dataset（Kaggle） |

## 项目结构

```
app/
  main.py            # FastAPI 应用 + 启动时建表
  config.py          # 从环境变量读配置
  database.py        # engine、session、get_db 依赖
  auth/              # JWT + 密码哈希
  models/            # User、Artist、Song、Rating
  schemas/           # Pydantic 请求/响应模型
  routers/           # auth、songs、artists、ratings、recommend、analytics
  services/          # 基于内容的推荐器
tests/               # API 测试（每次用独立的临时数据库）
data/                # 数据集导入脚本（外加 dataset.csv，不提交）
docs/                # Swagger UI PDF、幻灯片、技术报告
```

## 安装

```bash
python -m venv venv
# Windows: venv\Scripts\activate
source venv/bin/activate
pip install -r requirements.txt
```

可选：导入演示数据（需要把 Kaggle 的 CSV 放到 `data/dataset.csv`）。

```bash
python data/import_spotify.py            # 导入前 5000 行
python data/import_spotify.py data/dataset.csv 2000
```

数据库文件 `music.db` 在首次启动时自动创建，已在 .gitignore 里；要重建就跑上面的导入脚本。

## 配置

所有配置都有开发用默认值，可用环境变量覆盖。把 `.env.example` 复制成 `.env` 再改（装了 `python-dotenv` 会自动加载），或者在 shell 里直接导出：

| 变量 | 默认 | 作用 |
| --- | --- | --- |
| `MRA_SECRET_KEY` | 开发用 key | JWT 签名密钥 —— **上线必须改** |
| `MRA_ACCESS_TOKEN_EXPIRE_MINUTES` | `30` | token 有效期 |
| `MRA_DATABASE_URL` | `sqlite:///./music.db` | SQLAlchemy 连接串 |
| `MRA_PAGE_SIZE_DEFAULT` / `MRA_PAGE_SIZE_MAX` | `20` / `100` | 列表分页 |
| `MRA_RECOMMEND_LIMIT_DEFAULT` / `MRA_RECOMMEND_LIMIT_MAX` | `10` / `50` | 推荐条数 |
| `MRA_MAX_SIMILAR_CANDIDATES` | `5000` | 每次推荐扫描多少首歌 |

`MRA_SECRET_KEY` 保持默认时，启动会打一条警告。

## 运行

```bash
uvicorn app.main:app --reload
```

- Swagger UI：http://127.0.0.1:8000/docs （静态资源已放进
  `app/static/swagger`，离线也能用）
- API 文档 PDF：`docs/Music Recommendation API - Swagger UI.pdf`
- 幻灯片：`docs/Music_Recommendation_API.pptx`
- 技术报告：`docs/technical_report.pdf`

## 认证

```bash
curl -X POST http://127.0.0.1:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","email":"alice@example.com","password":"testpass123"}'

curl -X POST http://127.0.0.1:8000/auth/login \
  -d "username=alice&password=testpass123"
```

把返回的 token 放进请求头：`Authorization: Bearer <token>`。

## 接口

### 认证

| 方法 | 路径 | 需认证 | 说明 |
| --- | --- | --- | --- |
| POST | `/auth/register` | - | 注册 |
| POST | `/auth/login` | - | 取 JWT access token |
| GET | `/auth/me` | 是 | 当前用户 |

### 曲库

| 方法 | 路径 | 需认证 | 说明 |
| --- | --- | --- | --- |
| GET | `/songs/` | - | 歌曲列表（`skip`、`limit`） |
| GET | `/songs/{id}` | - | 歌曲详情 |
| POST | `/songs/` | 是 | 新建歌曲 |
| PUT | `/songs/{id}` | 是 | 更新歌曲 |
| DELETE | `/songs/{id}` | 是 | 删除歌曲（连同评分） |
| GET | `/artists/` | - | 歌手列表 |
| GET | `/artists/{id}` | - | 歌手详情 |
| POST | `/artists/` | 是 | 新建歌手 |
| PUT | `/artists/{id}` | 是 | 更新歌手 |
| DELETE | `/artists/{id}` | 是 | 删除歌手（连同其歌曲） |

### 评分

| 方法 | 路径 | 需认证 | 说明 |
| --- | --- | --- | --- |
| POST | `/ratings/` | 是 | 给歌曲打分（1–5）。重复打分是更新，返回 200 |
| GET | `/ratings/me` | 是 | 我的评分 |
| DELETE | `/ratings/{song_id}` | 是 | 取消我的评分 |

### 推荐

| 方法 | 路径 | 需认证 | 说明 |
| --- | --- | --- | --- |
| GET | `/recommend/similar/{song_id}` | - | 内容相似的歌曲 |
| GET | `/recommend/mood?mood=happy` | - | happy、sad、energetic、calm（其他值返回 400） |
| GET | `/recommend/genre?genre=pop` | - | 某流派内的歌曲，按热度排序 |
| GET | `/recommend/for-me` | 是 | 基于我的评分生成个性化推荐 |

### 统计

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/analytics/genre-trends` | 按歌曲数排名的流派 |
| GET | `/analytics/mood-distribution` | 情绪分布 + 百分比（空库也不会出错） |
| GET | `/analytics/top-artists` | 按平均热度排名的歌手 |
| GET | `/analytics/audio-features-summary` | 音频特征均值 |

## 推荐怎么算

- **相似歌曲**：对 `danceability`、`energy`、`valence`、`tempo`（归一化到
  0–1）、`acousticness`、`speechiness` 做余弦相似度。
- **为我推荐**：用我的评分（`score - 2.5`）构建加权口味画像，再对未评分的歌
  打分：75% 音频相似度 + 25% 流派偏好。没有评分的用户拿到的是热门歌曲，
  `reason` 字段会说明这是一次回退。

所有排序都是确定的：打分前先按 `id` 排序候选。

## 测试

```bash
pytest -q
```

每个测试用独立的临时 SQLite 数据库（依赖覆盖），不会碰真实的 `music.db`。

## 已知限制

- 只有基于内容的推荐；没有协同过滤 / 矩阵分解。
- 没有 refresh token、token 吊销或登录限流。
- 没有 Alembic 迁移；改表结构要重建 `music.db`。
- 本地开发时 `CORS` 是全开的（`*`）。

## 协议

MIT
