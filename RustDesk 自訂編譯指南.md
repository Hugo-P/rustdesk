# RustDesk 自訂伺服器編譯指南

本指南說明如何修改 RustDesk 源碼，使其支援透過 GitHub Actions 環境變數（Secrets）在編譯時動態嵌入自訂配置。

## 支援的環境變數

| 環境變數 | 用途 | 說明 |
|---------|------|------|
| `RS_PUB_KEY` | 公鑰 | 覆蓋預設的 RustDesk 公鑰 |
| `RENDEZVOUS_SERVER` | 中繼伺服器 | 設定您的自建中繼伺服器地址 |
| `API_SERVER` | API 伺服器 | 設定您的自建 API 伺服器地址 |
| `UNLOCK_PIN` | 解鎖 PIN | 設定遠端桌面的解鎖 PIN 碼 |
| `PERMANENT_PASSWORD` | 永久密碼 | 設定遠端桌面的永久連線密碼 |

---

## 第一步：複製本地專案與更新子模組

```bash
# 1. Clone 您 Fork 的主倉庫（請將 Hugo-P 替換為您的 GitHub 帳號）
git clone https://github.com/Hugo-P/rustdesk.git
cd rustdesk

# 2. 修改 .gitmodules 指向您 Fork 的子模組倉庫
git config -f .gitmodules submodule."libs/hbb_common".url "https://github.com/Hugo-P/hbb_common"

# 3. 同步子模組 URL 並拉取最新內容
git submodule sync
git submodule update --init --recursive --remote
```

---

## 第二步：修改原始碼與更新 Commit 指針

### 2.1 進入子模組目錄修改原始碼

```bash
cd libs/hbb_common
```

### 2.2 修改 `src/config.rs`

在 `src/config.rs` 中進行以下修改：

#### 修改項 1：RS_PUB_KEY（公鑰）

```rust
// 原本:
pub const RS_PUB_KEY: &str = "OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=";

// 替換為:
pub const PUBLIC_RS_PUB_KEY: &str = "OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=";
pub const RS_PUB_KEY: &str = match option_env!("RS_PUB_KEY") {
    Some(key) if !key.is_empty() => key,
    _ => PUBLIC_RS_PUB_KEY,
};
```

#### 修改項 2：PROD_RENDEZVOUS_SERVER（中繼伺服器）

```rust
// 原本:
pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new("".to_owned());

// 替換為:
pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new(match option_env!("RENDEZVOUS_SERVER") {
    Some(key) if !key.is_empty() => key,
    _ => "",
}.to_owned());
```

#### 修改項 3：get_unlock_pin()（解鎖 PIN）

```rust
// 原本:
pub fn get_unlock_pin() -> String {
    if Self::is_disable_unlock_pin() {
        return String::new();
    }
    CONFIG2.read().unwrap().unlock_pin.clone()
}

// 替換為:
pub fn get_unlock_pin() -> String {
    if Self::is_disable_unlock_pin() {
        return String::new();
    }
    if let Some(pin) = option_env!("UNLOCK_PIN") {
        if !pin.is_empty() {
            return pin.to_string();
        }
    }
    CONFIG2.read().unwrap().unlock_pin.clone()
}
```

#### 修改項 4：HARD_SETTINGS（永久密碼）

```rust
// 原本:
pub static ref HARD_SETTINGS: RwLock<HashMap<String, String>> = Default::default();

// 替換為:
pub static ref HARD_SETTINGS: RwLock<HashMap<String, String>> = {
    let mut map = HashMap::new();
    // 被控端預設密碼，固定密碼，讀取 Repository secrets 值
    map.insert(
        "password".to_string(),
        option_env!("PERMANENT_PASSWORD").unwrap_or("").into()
    );
    RwLock::new(map)
};
```

### 2.3 切回 main 分支並提交子模組改動

```bash
cd libs/hbb_common

# 確認目前分支狀態
git branch -a

# 如果顯示 (HEAD detached at xxxxxx)，必須切回 main
git checkout main

# 確認已切回 main
git branch -a

# 提交並推送子模組改動
git add src/config.rs
git commit -m "feat: 支援讀取環境變數配置"
git push origin HEAD
```

### 2.4 返回主倉庫目錄

```bash
cd ../..
```

### 2.5 修改主倉庫工作流

#### 2.5.1 修改 `flutter-build.yml`（加入環境變數）

編輯 `.github/workflows/flutter-build.yml`，在 `SIGN_BASE_URL` 下方加入：

```yaml
env:
  SIGN_BASE_URL: "${{ secrets.SIGN_BASE_URL }}-2"
  RS_PUB_KEY: "${{ secrets.RS_PUB_KEY }}"
  RENDEZVOUS_SERVER: "${{ secrets.RENDEZVOUS_SERVER }}"
  API_SERVER: "${{ secrets.API_SERVER }}"
  UNLOCK_PIN: "${{ secrets.UNLOCK_PIN }}"
  PERMANENT_PASSWORD: "${{ secrets.PERMANENT_PASSWORD }}"
```

#### 2.5.2 停用 push 觸發 CI（避免每次 push 都編譯）

編輯 `.github/workflows/flutter-ci.yml`，將 `push` 區塊註解掉：

```yaml
on:
  workflow_dispatch:
  pull_request:
    paths-ignore:
    - "docs/**"
    - "README.md"
#  push:
#    branches:
#      - master
#    paths-ignore:
#      - ".github/**"
#      - "docs/**"
#      - "README.md"
#      - "res/**"
#      - "appimage/**"
#      - "flatpak/**"
```

編輯 `.github/workflows/ci.yml`，同樣將 `push` 區塊註解掉：

```yaml
on:
  workflow_dispatch:
  pull_request:
    paths-ignore:
      - "docs/**"
      - "README.md"
#  push:
#    branches:
#      - master
#    paths-ignore:
#      - ".github/**"
#      - "docs/**"
#      - "README.md"
#      - "res/**"
#      - "appimage/**"
#      - "flatpak/**"
```

### 2.6 提交主倉庫改動

```bash
# 先確認目前分支名稱
git branch

# 提交並推送（將 main 替換為您的分支名稱，如 master）
git add .gitmodules libs/hbb_common .github/workflows/flutter-build.yml
git commit -m "chore: 更新子模組指針與 Actions 配置"
git push origin HEAD
```

---

## 第三步：寫入 Secrets 變數

### 方式一：命令列（推薦）

```bash
# 基本伺服器配置
gh secret set RENDEZVOUS_SERVER -b "您的伺服器IP或域名"
gh secret set API_SERVER -b "http://您的伺服器IP:21114"
gh secret set RS_PUB_KEY -b "您的公鑰Key"

# 安全配置（選用）
gh secret set UNLOCK_PIN -b "您的解鎖PIN"
gh secret set PERMANENT_PASSWORD -b "您的永久密碼"
```

### 方式二：GitHub 網頁手動設定

1. 打開您的 Fork 倉庫頁面
2. 點選 **Settings**（設定）
3. 左側選單找到 **Secrets and variables** → **Actions**
4. 點選 **New repository secret**
5. 依序新增以下 Secrets：

| Name | Value |
|------|-------|
| `RENDEZVOUS_SERVER` | 您的伺服器IP或域名 |
| `API_SERVER` | `http://您的伺服器IP:21114` |
| `RS_PUB_KEY` | 您的公鑰 Key |
| `UNLOCK_PIN` | 您的解鎖 PIN（選用） |
| `PERMANENT_PASSWORD` | 您的永久密碼（選用） |

### 設定 Workflow permissions

1. 在同一個 **Settings** 頁面
2. 左側選單找到 **Actions** → **General**
3. 往下找到 **Workflow permissions**
4. 選擇 **Read and write permissions**
5. 勾選 **Allow GitHub Actions to create and approve pull requests**
6. 點選 **Save**

> **注意**：如果不設定，Actions 可能無法正常推送或上傳構建产物。

### 參數說明

| 參數 | 格式範例 | 說明 |
|------|---------|------|
| `RENDEZVOUS_SERVER` | `rs.example.com` | 中繼伺服器域名或 IP |
| `API_SERVER` | `http://192.168.1.100:21114` | API 伺服器完整 URL |
| `RS_PUB_KEY` | `OeVuKk5nlHiXp+APNn0Y...` | Base64 格式公鑰 |
| `UNLOCK_PIN` | `1234` | 4-6 位數字 PIN 碼 |
| `PERMANENT_PASSWORD` | `MySecurePass123` | 任意字串密碼 |

---

## 第四步：觸發雲端編譯與下載檔案

### 方式一：建立標籤觸發（推薦）

```bash
# 建立版本標籤並推送，觸發 Actions 編譯
git tag v1.0.0
git push origin v1.0.0
```

### 方式二：手動觸發

```bash
# 命令列直接觸發 Actions 工作流開始編譯
gh workflow run flutter-build.yml
```

### 方式三：Nightly Build（每自動夜構建）

```bash
# 手動觸發 Nightly Build
gh workflow run flutter-nightly.yml
```

- 每天午夜 00:00 自動執行（排程：`cron: "0 0 * * *"`）
- 也可透過 GitHub 網頁手動觸發
- 構建产物會上傳到 `nightly` tag

### 查看進度與下載

```bash
# 即時查看雲端編譯狀態與進度
gh run watch

# 編譯完成後，一鍵下載構建出的客戶端安裝檔
gh run download
```

---

## 環境變數運作原理

所有環境變數皆使用 `option_env!` 巨集，在**編譯時**嵌入執行檔中：

- **有設定環境變數**：使用您設定的值
- **未設定環境變數**：使用 RustDesk 預設值

> **注意**：更改環境變數後需要重新編譯才會生效。

### 密碼驗證流程

1. 客戶端連線時，伺服器傳送 salt 給客戶端
2. 客戶端將密碼與 salt 進行 SHA256 雜湊
3. 伺服端比對雜湊值是否匹配
4. 優先檢查 `HARD_SETTINGS` 中的 `PERMANENT_PASSWORD`
5. 再檢查本地儲存的密碼
6. 最後檢查預設密碼

---

## 常見問題

### Q: 設定密碼後，原本的連線會受影響嗎？

A: 不會。環境變數密碼是**額外**的驗證來源，原有的臨時密碼和本地永久密碼仍然有效。

### Q: 可以只設定部分環境變數嗎？

A: 可以。未設定的環境變數會使用預設值，不影響其他功能。

### Q: PIN 和密碼有什麼差異？

A:
- **UNLOCK_PIN**：用於解鎖客戶端介面，防止未授權存取設定
- **PERMANENT_PASSWORD**：用於遠端連線驗證，是連線時需要輸入的密碼
