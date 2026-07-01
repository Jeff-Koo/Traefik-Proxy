# Traefik Reverse Proxy 配置指南

本專案使用 Traefik 作為全域的反向代理（Reverse Proxy），採用**選擇性跳轉架構**。
這允許我們在同一個伺服器環境中，同時並存「強制 HTTPS 的安全服務」與「僅支援純 HTTP 的老舊服務」。

此外，Traefik Dashboard 預設已開啟，並透過 **HTTPS 連線** 與 **Basic Auth (基本身分驗證)** 進行雙重保護。

---

## 🛠️ 前置作業：生成 Basic Auth 密碼

為了保護 Traefik Dashboard，我們使用 Docker 執行 `htpasswd` 命令來生成安全加密的 `bcrypt` 密碼雜湊，完全不需要在本地主機安裝任何額外套件。

### 步驟 1：使用 Docker 容器生成加密字串
在終端機執行以下命令（請將 `authusername` 換成你想設定的用戶名）：

```bash
docker run --rm -ti httpd:alpine htpasswd -B -n authusername

```

*執行後，系統會提示你輸入並確認密碼（密碼輸入時不會顯示在螢幕上）。*

**輸出範例：**

```text
authusername:$2y$05$vI8K.../xxxxxxxxx

```

*註：`-B` 參數確保使用高品質的 bcrypt 演算法；`--rm` 會在生成結束後自動銷毀臨時容器，不佔用空間。*

### ⚠️ 重要：Docker Compose 的 `$` 轉義規則

當你將生成的字串複製到 `compose.yaml` 時，**必須將所有的單個 `$` 符號手動改寫為兩個 `$$**`。

* **原始輸出：** `authusername:$2y$05$vI8K...`
* **Compose 標籤寫法：** `authusername:$$2y$$05$$vI8K...`
*原因：Docker Compose 會將單個 `$` 誤判為環境變數，必須使用 `$$` 進行轉義才能正常運作。*

---

## 📦 核心配置與初始化部署

在專案根目錄建立 `compose.yaml` 後，請**務必**依照以下步驟初始化憑證檔案，否則 Traefik 將因安全機制拒絕啟動憑證解析器。

---

### 🛠️ 憑證檔案初始化三部曲

> ⚠️ **為什麼這條流水線至關重要？**
> 1. **防止資料夾誤建**：若不先手動 `touch` 建立檔案，Docker 掛載 Volume 時會自動將它們建立為「資料夾」，導致 Traefik 報錯。
> 2. **符合 Traefik 安全規範**：Let's Encrypt 的私鑰會存於此處，Traefik 強制要求權限必須為 `600`（僅擁有者可讀寫），且強烈建議將擁有者設為 `root`，以防 Docker 運行時發生權限衝突。

#### 步驟 1：建立憑證 JSON 空檔案
```bash
touch ./acme.json ./acme-staging.json
```

#### 步驟 2：強制修正擁有者與安全權限

```bash
# 1. 先將檔案擁有者轉移給最高權限 root
sudo chown root:root ./acme.json ./acme-staging.json

# 2. 限制檔案權限為僅擁有者(root)可讀寫 (0600)
sudo chmod 600 ./acme.json ./acme-staging.json
```

#### 步驟 3：正式啟動 Traefik 容器

確認檔案與權限皆就緒後，便可安全部署：

```bash
docker compose up -d
```

---

## 🚀 後端服務接入範例

由於本環境採用**選擇性跳轉**，你在部署其他容器時，請根據需求選擇以下其中一種寫法：

### 應用 A：常規服務（需要 HTTPS & 自動跳轉）

同時配置 `http` 與 `https` 兩個路由，並在 http 路由掛載 `global-redirect-to-https@docker`：

```yaml
  labels:
    - "traefik.enable=true"
    # HTTP 路由：自動跳轉
    - "traefik.http.routers.myapp-http.entrypoints=web"
    - "traefik.http.routers.myapp-http.rule=Host(`app.mydomain.com`)"
    - "traefik.http.routers.myapp-http.middlewares=global-redirect-to-https@docker"
    # HTTPS 路由：提供加密服務
    - "traefik.http.routers.myapp-https.entrypoints=websecure"
    - "traefik.http.routers.myapp-https.rule=Host(`app.mydomain.com`)"
    - "traefik.http.routers.myapp-https.tls=true"
    - "traefik.http.routers.myapp-https.tls.certresolver=myresolver"

```

### 應用 B：特殊服務（維持純 HTTP、拒絕 SSL）

只配置一個聽 `web` (80) 的路由，**絕不**掛載任何中間件：

```yaml
  labels:
    - "traefik.enable=true"
    - "traefik.http.routers.my-legacy.entrypoints=web"
    - "traefik.http.routers.my-legacy.rule=Host(`legacy.mydomain.com`)"

```

---

## 📂 運維與檢查

1. **啟動服務：**
```bash
docker compose up -d
```


2. **檢查日誌：**
日誌會自動緩衝並寫入主機的 `./logs/traefik.log` 中。


3. **驗證安全性：**
嘗試瀏覽 `http://traefik.mydomain.com/dashboard/`，應會自動跳轉至 `https://`，且在連線成功加密後才會彈出輸入密碼的視窗。
