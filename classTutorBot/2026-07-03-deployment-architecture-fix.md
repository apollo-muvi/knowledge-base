# ClassTutorBot 部署架構問題與修正（2026-07-03）

## 問題發生時間
2026-07-03，小h（hp8100）部署時發生 PostgreSQL 連接失敗。

## 部署失敗根本原因

**docker-compose.yml 架構缺陷**

| 機器 | Branch | PostgreSQL | docker-compose.yml | 問題 |
|:-----|:-------|:-----------|:-------------------|:-----|
| Idea3 | dev | 外部容器（classtutorbot-pg） | ❌ 無 postgres 服務 | 開發規格書設計 |
| 小h | main | ❌ 無 | ❌ 無 postgres 服務 | **架構缺陷** |
| Pi4 | main | ❌ 無 | ❌ 無 postgres 服務 | **架構缺陷** |

## 我犯的錯

1. **未檢查實際需求**：
   - 開發規格書 10.7 說明是「Idea3 專用」
   - 但小h 和 Pi4 都是正式部署機器
   - 我沒有思考正式環境需要獨立的 PostgreSQL

2. **盲目執行步驟**：
   - 照著開發規格書的 dev branch 步驟
   - 沒有確認這個架構是否適合小h

3. **假設錯誤**：
   - 假設 main branch 會包含 postgres 服務
   - 實際上 main 和 dev branch 都沒有

## 正確做法

**docker-compose.yml 必須包含 postgres 服務**：

```yaml
services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: classtutorbot
      POSTGRES_PASSWORD: classtutorbot123
      POSTGRES_DB: classtutorbot
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U classtutorbot -d classtutorbot"]
      interval: 10s
      timeout: 5s
      retries: 5

  qdrant:
    ...

  backend:
    ...
    depends_on:
      postgres:
        condition: service_healthy
      qdrant:
        condition: service_started
    ...

  frontend:
    ...
    depends_on:
      - backend

volumes:
  postgres_data:
  qdrant_data:
  uploads:
```

## 部署架構差異

### 開發環境（Idea3）
- 使用外部容器 `classtutorbot-pg`
- port 5433（避免與系統 PostgreSQL 衝突）
- DATABASE_URL: `postgresql://classtutorbot:classtutorbot123@localhost:5433/classtutorbot`

### 正式環境（小h、Pi4）
- docker-compose.yml 包含 postgres 服務
- port 5432
- DATABASE_URL: `postgresql://classtutorbot:classtutorbot123@postgres:5432/classtutorbot`

## 已修正的文件

1. **docker-compose.yml** - 加入 postgres 服務
2. **開發規格書.md** - 更新第 10.7、10.8 節說明部署架構差異

## Git 提交

```
commit 3eaa8ea
fix(docker): Add postgres service to docker-compose.yml and update deployment docs

- Add postgres service to docker-compose.yml with healthcheck
- Update dev specs to distinguish deployment arch:
  - Idea3: external container classtutorbot-pg (port 5433)
  - Production (smallh, Pi4): postgres service included (port 5432)
- Fix root cause: main/dev branches missing postgres service in compose file
- Resolve connection refused errors in production deployments
```

## 重要教訓

1. **不要假設**：開發規格書的設計不一定適合所有環境
2. **檢查實際需求**：正式機和開發機的架構需求可能不同
3. **驗證架構**：部署前先檢查 docker-compose.yml 是否包含所有必要服務

## 參考資料
- [開發規格書 v2.0](https://github.com/apollo-muvi/cram_service/blob/main/docs/%E9%96%8B%E7%99%BC%E8%A6%8F%E6%A0%BC%E6%9B%B8.md)
- [docker-compose.yml](https://github.com/apollo-muvi/cram_service/blob/main/docker-compose.yml)