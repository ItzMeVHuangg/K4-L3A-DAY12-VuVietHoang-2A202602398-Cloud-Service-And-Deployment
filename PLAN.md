# PLAN.md — Quy trình làm bài K4-L3A Ngày 12 (Cloud Service & Deployment)

Tài liệu này tổng hợp lại từ [README.md](README.md), [LAB_GUIDE.md](LAB_GUIDE.md),
[CHECKPOINTS.md](CHECKPOINTS.md), [RUBRIC.md](RUBRIC.md) thành **một quy trình
làm bài tuần tự**: mỗi bước ghi rõ *lệnh chạy*, *lệnh kiểm tra* và *output dự
kiến* để biết bước đó đã xong hay chưa. Trạng thái hiện tại của repo: toàn bộ
`app/` còn là khung với `TODO` / `NotImplementedError` — chưa bước nào hoàn
thành.

---

## Bước 0 — Đặt tên repo & Setup môi trường

**Lệnh chạy:**
```bash
# đổi tên repo GitHub theo mẫu K4-L3A-DAY12-<HoVaTen>-<MSSV>-CloudServicesAndDeployment
python -m venv .venv
.venv\Scripts\Activate.ps1          # Windows PowerShell
pip install -r requirements.txt
copy .env.example .env              # Windows
python -c "import secrets; print(secrets.token_urlsafe(32))"   # dán vào AGENT_API_KEY trong .env
docker compose up -d redis          # hoặc REDIS_URL=fake:// trong .env nếu chưa có Docker
```

**Lệnh kiểm tra:**
```bash
pytest tests/ -v -m "not docker"
```

**Output dự kiến:** pytest chạy được, không có `ModuleNotFoundError`/`ImportError`.
Phần lớn test **RỚT** — đúng như thiết kế vì `app/` chưa có code.

---

## Bước 1 — Block 1: `app/config.py`, `app/logging_utils.py`, `/health`

**Việc cần làm:**
- `app/config.py`: khai báo 6 trường trong `Settings` — đặc biệt `agent_api_key: str` **không có default**.
- `app/logging_utils.py`: `log_event()` in một dòng JSON duy nhất (không `indent`, có `ensure_ascii=False`).
- `app/main.py` → `/health`: trả `200 {"status":"ok","service":...,"version":...}`, hoặc `503 {"status":"shutting_down"}` khi `lifecycle.shutting_down`; **không được gọi Redis**.

**Lệnh chạy (thử tay):**
```bash
uvicorn app.main:app --reload --port 8000
curl -i http://localhost:8000/health
```

**Output dự kiến:** `HTTP/1.1 200 OK`, body `{"status":"ok","service":"day12-agent","version":"1.0.0"}`.

**Lệnh kiểm tra (checkpoint):**
```bash
pytest tests/test_cp1.py -v
```

**Output dự kiến:** tất cả test trong `test_cp1.py` PASS (15 điểm nếu full).

---

## Bước 2 — Block 2: Docker (`Dockerfile`, `.dockerignore`, `docker-compose.yml`)

**Việc cần làm (6 yêu cầu ghi trong chính `Dockerfile`):**
1. Multi-stage build: `builder` (compile) → `runtime` (`python:3.11-slim`, chỉ `COPY --from=builder`).
2. Thứ tự: `COPY requirements.txt` → `pip install` → `COPY app` (code copy sau để cache layer).
3. Non-root: `useradd --uid 10001 appuser` + `USER appuser`.
4. `HEALTHCHECK` gọi `/health`; `CMD` bind `0.0.0.0` và đọc `${PORT:-8000}`.
5. `.dockerignore` loại `.env`, `__pycache__`, `.git`, `.venv` (nhưng giữ `app`, `utils`, `requirements.txt`).
6. `docker-compose.yml` thêm service `agent`: build từ Dockerfile, `depends_on: redis`, `environment: AGENT_API_KEY: ${AGENT_API_KEY}`, `REDIS_URL: redis://redis:6379/0`, có healthcheck.

**Lệnh chạy:**
```bash
docker build -t day12-agent:prod .
docker images day12-agent:prod
docker compose up -d
curl http://localhost:8000/health
docker compose logs agent
```

**Output dự kiến:**
- Image build thành công, dung lượng **dưới 500MB**.
- `docker compose ps` → cột STATE là `running`/`healthy` cho cả `agent` và `redis`.
- `curl /health` → `200 {"status":"ok",...}`.

**Lệnh kiểm tra:**
```bash
pytest tests/test_cp2.py -v
pytest tests/test_cp2.py -v -m "not docker"   # kiểm tra nhanh phần cấu trúc, bỏ qua build
```

**Output dự kiến:** tất cả test `test_cp2.py` PASS (15 điểm nếu full).

---

## Bước 3 — Block 3: API Security (`auth.py`, `rate_limiter.py`, `cost_guard.py`, `/ask`)

**Việc cần làm:**
- `app/auth.py`: đọc header `X-API-Key`, so sánh bằng `secrets.compare_digest` (không dùng `==`); trả `user_id` từ `X-User-Id` (mặc định `ANONYMOUS_USER`); sai/thiếu key → `401`.
- `app/rate_limiter.py`: sliding window bằng Redis ZSET — `zremrangebyscore` → `zcard` (check) → `zadd` với member duy nhất `f"{now}:{uuid4().hex}"` → `expire`. **Kiểm tra trước, ghi sau.**
- `app/cost_guard.py`: `spent()`/`check()`/`record()`, key `cost:<user>:<YYYY-MM>`; `spent()` trả `0.0` khi Redis trả `None`.
- `/ask` trong `main.py`: đúng thứ tự `verify_api_key → limiter.check → guard.check → get_history → ask_llm → append×2 → guard.record → log_event`.

**Lệnh chạy (thử tay):**
```bash
# không key → 401
curl -i -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'

# có key → 200
curl -i -X POST http://localhost:8000/ask -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" -H "X-User-Id: sv01" -d '{"question":"Docker là gì?"}'

# gọi 15 lần → những lần cuối phải 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST http://localhost:8000/ask \
    -H "Content-Type: application/json" -H "X-API-Key: $AGENT_API_KEY" -H "X-User-Id: sv01" \
    -d '{"question":"test"}'
done; echo
```

**Output dự kiến:** lần 1 → `401`; lần 2 → `200` kèm JSON `{"answer":...,"cost_usd":...}`; chuỗi 15 request → các mã `200` đầu, chuyển sang `429` khi vượt `RATE_LIMIT_PER_MINUTE` (mặc định 10).

**Lệnh kiểm tra:**
```bash
pytest tests/test_cp3.py -v
```

**Output dự kiến:** tất cả test `test_cp3.py` PASS (20 điểm nếu full).

---

## Bước 4 — Block 4: Scaling & Reliability (`store.py`, `/ready`, `lifecycle.py`)

**Việc cần làm:**
- `app/store.py`: lịch sử hội thoại lưu trong Redis (`rpush`/`ltrim` giữ `HISTORY_MAX_MESSAGES` gần nhất, `expire` để tự dọn); `ping()` nuốt exception, trả `False`.
- `/ready`: `503 {"status":"shutting_down"}` khi đang tắt; `503 {"status":"not ready","redis":false}` khi `store.ping()` False; ngược lại `200 {"status":"ready","redis":true}`.
- `app/lifecycle.py`: đăng ký `SIGTERM`/`SIGINT`, bật cờ `shutting_down`, **gọi lại handler cũ** (không ghi đè uvicorn).

**Lệnh chạy:**
```bash
docker compose up -d --scale agent=3
docker compose ps

for i in $(seq 1 5); do
  curl -s -X POST http://localhost:8000/ask -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" -H "X-User-Id: sv01" -d '{"question":"lượt '$i'"}' \
    | python -c "import json,sys; print(json.load(sys.stdin)['history_length'])"
done
```

**Output dự kiến:** 3 container `agent` chạy song song; `history_length` **tăng dần liên tục** (1,2,3,4,5) dù request rơi vào container khác nhau — chứng minh state nằm trong Redis, không nằm trong process.

**Lệnh kiểm tra:**
```bash
pytest tests/test_cp4.py -v
```

**Output dự kiến:** tất cả test `test_cp4.py` PASS (20 điểm nếu full).

---

## Bước 5 — Block 5: Deploy lên Cloud (Railway hoặc Render)

**Việc cần làm:** chọn 1 platform, deploy bằng Dockerfile đã có, set biến môi trường trên dashboard (không hardcode), điền [DEPLOYMENT.md](DEPLOYMENT.md) và chụp ảnh vào `screenshots/`.

**Lệnh chạy (Railway):**
```bash
npm i -g @railway/cli
railway login
railway init
railway add --database redis
railway variables --set AGENT_API_KEY=<khóa của bạn> \
                  --set RATE_LIMIT_PER_MINUTE=10 \
                  --set MONTHLY_BUDGET_USD=10.0 \
                  --set LOG_LEVEL=INFO
railway up
railway domain
railway logs
```

**Lệnh chạy (Render):** push repo lên GitHub → render.com → New → Blueprint → chọn repo (đọc `render.yaml`) → điền `AGENT_API_KEY` khi được hỏi → Create.

**Lệnh kiểm tra (trên URL public):**
```bash
URL=https://<domain-cua-ban>
curl -i $URL/health          # 200 {"status":"ok"}
curl -i $URL/ready           # 200 {"status":"ready"} — chứng minh đã nối Redis
curl -i -X POST $URL/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'  # 401
```

**Output dự kiến:** cả 3 lệnh trả đúng mã trạng thái nêu trên. Thêm `DEPLOY_API_KEY=<khóa cloud>` vào `.env` cục bộ (không commit) để test tự động gọi được bản deploy có xác thực.

**Không deploy được?** Đặt `LOCAL_FALLBACK=true` trong `.env`, chạy `docker compose up -d`, chụp `docker compose ps` + `/health` vào `screenshots/`, ghi lý do vào cuối `DEPLOYMENT.md`. CP5 khi đó tối đa 9/15.

**Lệnh kiểm tra:**
```bash
pytest tests/test_cp5.py -v
```

**Output dự kiến:** tất cả test `test_cp5.py` PASS (15 điểm nếu full, 9/15 nếu local fallback).

---

## Bước 6 — Wrap-up: `exercises.md`, chấm điểm, nộp bài

**Việc cần làm:** trả lời đủ 10 câu trong [exercises.md](exercises.md) bằng quan sát thực tế (log JSON thật, số đo dung lượng image thật, lỗi deploy thật...).

**Lệnh kiểm tra toàn bộ:**
```bash
pytest tests/ -v                    # toàn bộ test, biết rõ test nào rớt và vì sao
python grade.py                     # chấm điểm, mục tiêu ≥ 75/100
python grade.py --no-bonus          # chấm nhanh, bỏ qua bonus CI/CD
```

**Output dự kiến:** báo cáo điểm theo từng CP (`CP1..CP5`, `exercises.md`), tổng điểm hiển thị, không có dòng lỗi runtime ngoài kết quả test rớt/pass.

**Lệnh kiểm tra an toàn trước khi nộp:**
```bash
git status --porcelain | grep -q "\.env$" && echo "DỪNG LẠI: .env đang bị theo dõi"
git ls-files | grep -E '(^|/)\.env$|\.(pem|key)$'   # phải KHÔNG in ra gì (ngoại trừ .env.example)
```

**Output dự kiến:** không có kết quả — `.env` và mọi file secret không nằm trong `git ls-files`.

**Lệnh commit & push:**
```bash
git add -A
git commit -m "Hoàn thành lab Day 12"
git push origin main
```

---

## (Tùy chọn) Bonus — CI/CD với GitHub Actions (+10 điểm)

Chỉ làm sau khi CP1–CP5 đã xanh. Không có file mẫu — tự viết `.github/workflows/ci.yml`.

**Yêu cầu tối thiểu:**
- `on: push/pull_request` nhánh `main`.
- Job `test`: checkout → setup-python → `pip install -r requirements.txt` → `pytest tests/ -v --ignore=tests/test_cp5.py --ignore=tests/test_bonus_cicd.py`, với `env: AGENT_API_KEY: ci-dummy`, `REDIS_URL: fake://`.
- Job `build`: `docker build` trên runner.
- Job `deploy`: `needs: [test, build]`, `if: github.ref == 'refs/heads/main' && github.event_name == 'push'`, dùng `${{ secrets.RAILWAY_TOKEN }}` hoặc deploy hook — **không hardcode token trong YAML**.
- Ghim version action (`actions/checkout@v4`, không dùng `@main`).
- Smoke test sau deploy: `curl -fsS $URL/health` sau `sleep 45`.
- Badge trong `README.md`: `![CI](https://github.com/<user>/<repo>/actions/workflows/ci.yml/badge.svg)`.

**Lệnh chạy:**
```bash
git add .github/workflows/ci.yml README.md
git commit -m "Thêm CI/CD với GitHub Actions"
git push
```

**Output dự kiến:** tab **Actions** trên GitHub chạy 3 job xanh (`test`, `build`, `deploy` khi push vào `main`); badge trong README hiển thị `passing`.

**Lệnh kiểm tra:**
```bash
pytest tests/test_bonus_cicd.py -v
```

**Output dự kiến:** tất cả test `test_bonus_cicd.py` PASS (+10 điểm, tổng cuối vẫn ≤ 100).

---

## Danh sách kiểm tra cuối cùng

- [ ] Repo đúng tên `K4-L3A-DAY12-<HoVaTen>-<MSSV>-CloudServicesAndDeployment`
- [ ] `pytest tests/ -v` chạy hết, biết rõ test nào còn rớt và vì sao
- [ ] `python grade.py` ≥ 75/100
- [ ] `exercises.md` đủ 10 câu, viết bằng lời của mình
- [ ] `DEPLOYMENT.md` có Public URL thật, không dán giá trị `AGENT_API_KEY`
- [ ] `screenshots/` có ảnh dashboard và ảnh gọi `/health`
- [ ] `.env` không nằm trong repo
- [ ] Không còn `NotImplementedError` trong `app/`
- [ ] Có commit ở nhiều mốc thời gian
- [ ] *(Bonus)* `.github/workflows/ci.yml` chạy xanh, badge README báo `passing`
