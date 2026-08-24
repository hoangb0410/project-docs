# GitHub Actions — Cheatsheet cú pháp chung

Tài liệu tổng hợp các keyword hay dùng trong file `.github/workflows/*.yml`, dựa theo cấu trúc chung của GitHub Actions và các file thực tế trong repo.

---

## 1. Cấu trúc khung tổng quát

```yaml
name: <tên hiển thị trên tab Actions>

on: <sự kiện trigger>

permissions: <quyền GITHUB_TOKEN>

env: <biến môi trường dùng chung cả file>

jobs:
  <job_id>:
    runs-on: <máy chạy>
    needs: <job phải chạy xong trước>
    if: <điều kiện chạy job>
    permissions: <quyền riêng cho job này>
    environment: <môi trường deploy, nếu có>
    timeout-minutes: <giới hạn thời gian>
    steps:
      - <step 1>
      - <step 2>
```

---

## 2. `on` — sự kiện trigger

| Event                         | Ý nghĩa                                      | Hay dùng cho                          |
| ----------------------------- | -------------------------------------------- | ------------------------------------- |
| `push`                        | Có commit được push lên                      | CD (deploy sau khi merge)             |
| `pull_request`                | Có PR mở/update                              | CI (check trước khi merge)            |
| `issue_comment`               | Có comment trên issue/PR                     | Bot phản hồi theo comment (`@claude`) |
| `pull_request_review`         | Có review trên PR                            | Trigger theo review                   |
| `pull_request_review_comment` | Có comment trong review                      | Trigger theo comment review           |
| `issues`                      | Issue được tạo/gán                           | Tự động xử lý issue                   |
| `workflow_dispatch`           | Chạy thủ công bằng tay (nút Run trên GitHub) | Deploy thủ công, chạy script 1 lần    |
| `schedule`                    | Chạy theo lịch (cron)                        | Chạy report, dọn dẹp định kỳ          |
| `workflow_call`               | Được gọi bởi workflow khác                   | Reusable workflow                     |

**`pull_request.types`** — loại sự kiện con của PR:

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]
    # opened      = PR mới mở
    # synchronize = có commit mới push vào PR
    # reopened    = PR đóng rồi mở lại
```

**Lọc theo branch:**

```yaml
on:
  push:
    branches: [main, develop]
  pull_request:
    branches:
      - develop
      - release_v1
      - "pre-release_**" # wildcard
```

**Lọc theo đường dẫn file thay đổi:**

```yaml
on:
  pull_request:
    paths:
      - "src/**/*.ts"
      - "!src/**/*.spec.ts" # dấu ! = loại trừ
```

⚠️ Lưu ý: nếu dùng `paths` và bật check này thành "required" trong branch protection, PR nào không đụng vào các path đó sẽ bị treo "pending" mãi (workflow không chạy nên không bao giờ pass).

---

## 3. `permissions` — quyền của `GITHUB_TOKEN`

Theo nguyên tắc least privilege — chỉ cấp cái cần dùng:

```yaml
permissions:
  contents: read          # đọc code trong repo
  pull-requests: write    # comment/update PR
  pull-requests: read     # chỉ đọc thông tin PR
  issues: read            # đọc issue
  id-token: write         # bắt buộc khi dùng OIDC (vd đăng nhập AWS không cần access key)
  actions: read           # đọc kết quả CI job khác
```

Có thể khai báo ở cấp **file** (áp dụng cho mọi job) hoặc cấp **job** (ghi đè riêng cho job đó).

---

## 4. `jobs` — các job và quan hệ giữa chúng

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps: [...]

  deploy:
    needs: build # deploy chỉ chạy sau khi build xong và pass
    if: github.event_name == 'push' # điều kiện chạy job
    runs-on: ubuntu-latest
    environment: production # gắn với Environment trên GitHub (có thể set required reviewers)
    steps: [...]
```

**`needs`** — khai báo phụ thuộc:

```yaml
needs: build              # phụ thuộc 1 job
needs: [lint, test, build] # phụ thuộc nhiều job, tất cả phải pass
```

**`if`** — điều kiện chạy job/step, hay dùng các biến:

```yaml
if: github.event_name == 'push'
if: github.ref == 'refs/heads/main'
if: github.ref_name == 'release_v1'
if: github.base_ref == 'develop'
if: startsWith(github.head_ref, 'pre-release_')
if: contains(github.event.comment.body, '@claude')
if: success()        # job trước pass mới chạy (mặc định)
if: failure()        # chỉ chạy khi có job trước fail (vd gửi thông báo lỗi)
if: always()         # luôn chạy dù pass/fail (vd dọn dẹp, upload log)
```

---

## 5. `steps` — các bước trong 1 job

**Dùng action có sẵn (`uses`):**

```yaml
- name: Checkout code
  uses: actions/checkout@v4
  with:
    fetch-depth:
      0 # 0 = lấy full git history (cần cho diff giữa branch)
      # 1 = chỉ lấy commit mới nhất (nhanh hơn, mặc định)
```

**Chạy lệnh shell (`run`):**

```yaml
- name: Install dependencies
  run: npm ci
  # hoặc nhiều dòng:
  run: |
    npm ci
    npm run build
```

**Truyền biến môi trường cho 1 step:**

```yaml
- name: Run tests
  run: npm test
  env:
    NODE_OPTIONS: --max-old-space-size=4096
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

**Điều kiện cho từng step:**

```yaml
- name: Deploy
  if: github.ref == 'refs/heads/main'
  run: ./deploy.sh
```

**Bỏ qua lỗi của step (không làm fail cả job):**

```yaml
- name: Optional step
  run: some-command
  continue-on-error: true
```

**Đặt tên output để step sau dùng:**

```yaml
- name: Build image
  id: build-image
  run: |
    echo "image=myapp:latest" >> "$GITHUB_OUTPUT"

- name: Use output
  run: echo "Image is ${{ steps.build-image.outputs.image }}"
```

---

## 6. Biến & context hay dùng

| Biến                                      | Ý nghĩa                                        |
| ----------------------------------------- | ---------------------------------------------- |
| `${{ github.repository }}`                | tên repo dạng `owner/repo`                     |
| `${{ github.event.pull_request.number }}` | số PR                                          |
| `${{ github.base_ref }}`                  | branch đích của PR (vd `develop`)              |
| `${{ github.head_ref }}`                  | branch nguồn của PR (vd `feature/x`)           |
| `${{ github.ref }}`                       | ref đầy đủ (vd `refs/heads/main`)              |
| `${{ github.ref_name }}`                  | tên branch/tag ngắn gọn (vd `main`)            |
| `${{ github.sha }}`                       | commit hash hiện tại                           |
| `${{ github.event_name }}`                | loại event trigger (`push`, `pull_request`...) |
| `${{ secrets.XXX }}`                      | secret đã lưu trong Settings → Secrets         |
| `${{ steps.<id>.outputs.<name> }}`        | output từ step trước                           |
| `${{ env.XXX }}`                          | biến khai báo trong `env:`                     |

---

## 7. Bí danh (secrets) hay gặp trong repo

```yaml
secrets.GITHUB_TOKEN        # token mặc định GitHub tự cấp, không cần tạo
secrets.ANTHROPIC_API_KEY   # tự tạo trong Settings → Secrets and variables
secrets.SONAR_TOKEN
secrets.SONAR_HOST_URL
secrets.CLAUDE_CODE_OAUTH_TOKEN
```

---

## 8. `environment` — gắn job với môi trường deploy

```yaml
jobs:
  deploy:
    environment: production
```

Cho phép trên GitHub Settings → Environments:

- Đặt secret riêng theo từng môi trường (staging/production khác secret)
- Bật "Required reviewers" → job dừng lại chờ người approve (ranh giới Continuous Delivery vs Continuous Deployment)
- Giới hạn chỉ branch nào được deploy vào environment đó

---

## 9. Mẫu tối giản: 1 job CI đầy đủ

```yaml
name: Example CI

on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [develop]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: yarn

      - run: yarn install --frozen-lockfile

      - run: yarn test
```

---

## 10. Mẫu tối giản: 1 job CD đầy đủ

```yaml
name: Example CD

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write # cho OIDC login AWS

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<account-id>:role/<role-name>
          aws-region: ap-southeast-1

      - run: aws s3 sync ./dist s3://my-bucket --delete
```
