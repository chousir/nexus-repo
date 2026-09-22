# Nexus Repository 離線倉庫驗收指南

Maven / Docker / Helm 上傳下載驗證，含網域劫持（domain-hijack）測試。下列 ✅ 標記皆為實測結果（括號內為日期），非理論推測。

## 目錄

1. [目前架構](#目前架構)
2. [佈署指令](#佈署指令)（正式環境／本機測試）
3. [Maven](#maven--上傳--下載測試) / [Docker](#docker--上傳--下載測試) / [Helm](#helm--上傳--下載測試) / [PyPI](#pypi--上傳--下載測試)
4. [疑難排解](#疑難排解)
5. [附錄 A — Maven/sbt 大型專案離線相依作業流程](#附錄-a--mavensbt-大型專案離線相依作業流程)
6. [附錄 B — 離線環境佈署 Ansible 本身](#附錄-b--離線環境佈署-ansible-本身)

---

## 目前架構

容器化 **Nexus 3** + 容器化 **nginx** TLS 反向代理，同一個 `docker compose` 網路。**佈署步驟／參數／目錄結構見 `ansible/README.md`**；本文件只涵蓋佈署完成後的驗收測試。

| 項目 | 內容 |
|---|---|
| 版本 | `sonatype/nexus3:3.93.2`（Community Edition） |
| Hosted 儲存庫 | `docker-hosted`、`maven-hosted`（MIXED）、`helm-hosted`、`pypi-hosted`，皆用內建 `default` blob store |
| 尚未實作 | `raw-hosted`、`cargo-hosted`（規劃中，同 `helm-hosted` 模式：path-based，免額外 nginx 設定） |
| 存取 | 匿名讀寫，免帳密——**除了 pypi-hosted 的上傳**：`pip install` 仍匿名，`twine upload` 需 `pypi-uploader` 帳密（見下） |
| 反向代理 | 容器化 nginx，單一自簽憑證涵蓋全部主機名；vhost 依格式拆在 `conf.d/` |

**主機名對應**（預設值，可在 `inventory/group_vars/all.yml` 覆寫）

| 主機名 | 機制 | 用途 |
|---|---|---|
| `nexus.lab` | 一般反代 → `nexus:8081` | UI / API / Maven／PyPI（直接路徑）/ Helm |
| `registry.lab` | 一般反代 → `nexus:5000` | Docker registry |
| `docker.elastic.co`（範例） | 網域劫持：同一 connector 多掛一個 `server_name` ＋憑證 SAN | Docker 劫持，`nexus_registry_extra_server_names` |
| `repo1.maven.org`、`repo.maven.apache.org` | 網域劫持＋路徑改寫：`/maven2/` → `/repository/maven-hosted/` | Maven Central 劫持，`nexus_maven_extra_server_names` |
| `pypi.org` | 網域劫持＋路徑改寫：`/simple/`、`/packages/` → `/repository/pypi-hosted/...` | PyPI 安裝劫持（匿名），`nexus_pypi_extra_server_names` |
| `upload.pypi.org` | 網域劫持＋路徑改寫：`/legacy/` → `/repository/pypi-hosted/` | PyPI 上傳劫持（需 `pypi-uploader` 帳密），`nexus_pypi_upload_extra_server_names` |

> docker 劫持只是多掛一個 `server_name`（Docker Registry API 路徑本來就跟 Nexus 一致，不用改路徑）；maven 劫持要做路徑改寫（Maven Central 用 `/maven2/...`，Nexus 用 `/repository/<repo>/...`，格式不同）。pypi 也要路徑改寫，且拆成**兩個**網域——真正的 PyPI 本來就是讀（`pypi.org`）跟寫（`upload.pypi.org`）分屬不同主機，這裡如實對應。helm 不需要劫持——chart repo URL 本來就是自訂的，沒有像 Docker Hub／Maven Central／PyPI 那樣全世界共用的網域可劫持，直接用 `nexus.lab` 即可，raw/cargo 之後同理。

**憑證**：所有主機名（含四個劫持清單）收斂進 `nexus_cert_sans`。異動 `inventory/group_vars/all.yml` 的 `nexus_registry_extra_server_names`／`nexus_maven_extra_server_names`／`nexus_pypi_extra_server_names`／`nexus_pypi_upload_extra_server_names` 後重跑 `ansible-playbook site.yml` 即可——憑證會自動偵測 SAN 差異、重簽＋reload nginx，不需手動處理。

> ⚠️ compose 預設把 `8081`/`5000` 綁在 `nexus_bind_addr`（預設 `127.0.0.1`，僅供健康檢查）；正式環境勿改綁 `0.0.0.0`，否則外部可繞過 nginx 明文直連。匿名讀寫＝任何連得到的人都能推送/覆寫。

---

## 映像搬運（連網環境 → 離線環境）

Nexus／nginx 映像不存放在 `ansible/` 專案目錄裡——需要時自己在有網路的環境準備、搬進離線環境：

```bash
docker pull sonatype/nexus3:3.93.2 && docker save sonatype/nexus3:3.93.2 -o nexus3.tar
docker pull nginx:1.27-alpine       && docker save nginx:1.27-alpine       -o nginx.tar
```

用實體介質把兩個 `.tar` 傳到離線主機，`docker load -i nexus3.tar`／`docker load -i nginx.tar` 之後即可執行下方測試／佈署。**tag 必須跟 `inventory/group_vars/all.yml` 的 `nexus_image`／`nginx_image` 完全一致**——compose 預設 `pull_policy: missing`，tag 對不上就不會用到本地已載入的映像，離線環境也無法臨時 pull。

---

## 佈署指令

**正式環境**（映像已依上一節 `docker load` 好）：

```bash
cd ansible
ansible-playbook site.yml --become        # 路徑預設 /opt/nexus，--become 是因為要寫 /opt
```

覆寫任一變數用 `-e`，例如換主機名：`ansible-playbook site.yml --become -e nexus_server_name=repo.corp.local`。

**本機測試**（重現本文件所有驗證用，免 sudo）：

```bash
cd ansible
docker pull sonatype/nexus3:3.93.2 && docker pull nginx:1.27-alpine   # 本機測試環境本身有網路，直接 pull 即可
ansible-playbook site.yml -e @test-vars.yml                            # 免 sudo，路徑寫到 ./.deploy
```

`test-vars.yml` 只是本機測試用的覆寫檔（**不是**正式環境設定）：把 `nexus_project_dir` 從預設的 `/opt/nexus/compose`（需要 root 才能建立）換成 repo 內的 `ansible/.deploy`（一般使用者可寫），並把 `nexus_data_volume_type` 從 `bind`（掛載 `/opt/nexus/nexus-data`，同樣需要 root）換成 `volume`（Docker 具名 volume，不需要主機路徑權限）——兩者合起來讓整個佈署全程免 sudo、產物都留在 repo 目錄裡，方便重複測試、清除。正式上線用第一種指令，不要加 `-e @test-vars.yml`。

證書位置：`ansible/.deploy/certs/nexus.crt`（正式環境對應 `<nexus_project_dir>/certs/nexus.crt`，預設 `/opt/nexus/compose/certs/nexus.crt`）。

以下指令用 `curl --resolve <host>:443:127.0.0.1` 模擬「網域已被劫持解析到本機」，免 sudo、免改 `/etc/hosts`——`--resolve` 只在單次 curl 呼叫內覆寫 DNS，效果等同真的改了 DNS（仍走真正的 TLS SNI，被 nginx 依 `server_name` 正確路由）。正式環境要讓一般用戶端（`docker`/`mvn`/`helm` 本身無此參數）透明打過來，才需要真的改 `/etc/hosts`／內部 DNS，並讓對應用戶端信任 `nexus.crt`（各段落已列出方法）。

---

## Maven — 上傳 / 下載測試

### 直接路徑（`nexus.lab`）

```bash
CERT=ansible/.deploy/certs/nexus.crt

# 上傳（PUT，匿名，回 201）
curl -sS -o /dev/null -w '%{http_code}\n' \
  --resolve nexus.lab:443:127.0.0.1 --cacert "$CERT" \
  --upload-file artifact.jar \
  "https://nexus.lab/repository/maven-hosted/org/example/probe/1.0/probe-1.0.jar"

# 下載
curl -sS --resolve nexus.lab:443:127.0.0.1 --cacert "$CERT" \
  "https://nexus.lab/repository/maven-hosted/org/example/probe/1.0/probe-1.0.jar" -o out.jar
```

✅ **2026-09-17** — 上傳回 `201`；下載內容與原檔 `cmp` 逐位元組相同。

### 網域劫持路徑（`repo1.maven.org` / `repo.maven.apache.org`）

改用劫持網域＋`/maven2/` 路徑（同一份資料，不用重傳）：

```bash
curl -sS --resolve repo1.maven.org:443:127.0.0.1 --cacert "$CERT" \
  "https://repo1.maven.org/maven2/org/example/probe/1.0/probe-1.0.jar" -o via-repo1.jar

curl -sS --resolve repo.maven.apache.org:443:127.0.0.1 --cacert "$CERT" \
  "https://repo.maven.apache.org/maven2/org/example/probe/1.0/probe-1.0.jar" -o via-apache.jar
```

✅ **2026-09-17** — 兩者下載內容都與原始上傳的檔案逐位元組相同，證明 `/maven2/` → `/repository/maven-hosted/` 路徑改寫正確（不是巧合回 200）。

### 真實用戶端（`mvn` / `sbt`），正式環境

讓**完全沒改過設定**的 Maven/sbt 專案透明打到 `maven-hosted`：

```bash
# 1) 讓網域解析到 nginx 主機（正式環境用內部 DNS；測試機可用 /etc/hosts，需 sudo）
echo "<NGINX_IP> repo1.maven.org repo.maven.apache.org" | sudo tee -a /etc/hosts

# 2) 讓 JVM 信任 nexus.crt —— sbt/mvn 走 Java 的 truststore，
#    系統的 update-ca-certificates 對它無效
CACERTS="$(dirname "$(dirname "$(readlink -f "$(which java)")")")/lib/security/cacerts"
sudo keytool -importcert -alias nexus-lab -file nexus.crt -keystore "$CACERTS" -storepass changeit -noprompt
```

之後 `mvn install`／`sbt update` 不用寫 `<mirrors>`/`resolvers`：預設就是打 `repo1.maven.org`，劫持後即打到 `maven-hosted`。Push 同理，`publishTo`/`<distributionManagement>` 指向 `https://repo1.maven.org/maven2/` 即可（匿名讀寫，免帳密）。

> ⚠️ **劫持後該機器連不到真正的 Maven Central**：`maven-hosted` 只有你放進去的 artifact，若專案還相依公開套件（如 `scala-library`）會 404。完整取代 Central 需把那些相依一併灌進 `maven-hosted`——見**附錄 A：Maven/sbt 大型專案離線相依作業流程**（Spark + sbt 完整案例，含批次上傳與零修改驗證）。

---

## Docker — 上傳 / 下載測試

### 直接連 connector（伺服器本機，走 `localhost`，免憑證免登入）

```bash
docker tag myimage:tag 127.0.0.1:5000/myimage:tag
docker push 127.0.0.1:5000/myimage:tag
docker pull 127.0.0.1:5000/myimage:tag
```

✅ **2026-09-17**（`alpine:3.20`）— push/pull 皆成功，回傳 digest 與 pull 回來的 image ID 一致。

### 經 nginx TLS（`registry.lab`，一般用）

```bash
CERT=ansible/.deploy/certs/nexus.crt
curl -sS -o /dev/null -w '%{http_code}\n' --resolve registry.lab:443:127.0.0.1 --cacert "$CERT" https://registry.lab/v2/
```

**預期 `401`**：Docker Registry v2 的 Bearer Token 協定本來就是「先跟 `/v2/` 要挑戰、再拿 token 重試」，真正的 `docker` client 會自動處理；`curl` 直接 `GET /v2/` 看到 `401` 是正常、不是壞掉（`200` 也算通過，代表沒開 Bearer 驗證）。

### 網域劫持路徑（`docker.elastic.co`，用一兩個當代表即可）

```bash
curl -sS -o /dev/null -w '%{http_code}\n' --resolve docker.elastic.co:443:127.0.0.1 --cacert "$CERT" https://docker.elastic.co/v2/
```

✅ **2026-09-17** — `registry.lab` 與 `docker.elastic.co` 都回 `401`（同一個 Bearer 挑戰），證明兩者確實路由到同一個 `docker-hosted` connector，且憑證 SAN 正確涵蓋劫持網域（若 SAN 沒涵蓋，`--cacert` 驗證會直接失敗、連 HTTP 狀態碼都拿不到）。另加 `quay.io` 這類第二個劫持網域也驗證過，一樣即時生效（見 `nexus_registry_extra_server_names`）。

### 真實用戶端（`docker pull/push`），正式環境

```bash
# 1) 網域解析到 nginx 主機
echo "<NGINX_IP> registry.lab docker.elastic.co" | sudo tee -a /etc/hosts

# 2) 讓 docker daemon 信任 nexus.crt（每個劫持網域各建一份 ca.crt）
sudo mkdir -p /etc/docker/certs.d/registry.lab /etc/docker/certs.d/docker.elastic.co
sudo cp nexus.crt /etc/docker/certs.d/registry.lab/ca.crt
sudo cp nexus.crt /etc/docker/certs.d/docker.elastic.co/ca.crt

# 3) 在 Nexus 主機本機（走 localhost:5000，免憑證）先把要劫持的映像存進去，
#    存的是「retag 後的路徑」，跟用哪個網域 pull 無關：
docker tag docker.elastic.co/elasticsearch/elasticsearch:9.0.0 localhost:5000/elasticsearch/elasticsearch:9.0.0
docker push localhost:5000/elasticsearch/elasticsearch:9.0.0

# 4) 測試機（或任何解析得到 <NGINX_IP> 的機器）
docker pull docker.elastic.co/elasticsearch/elasticsearch:9.0.0   # 以為連官方 Elastic，實際來自本地 Nexus
docker pull registry.lab/myimage:tag                              # 一般用法
```

---

## Helm — 上傳 / 下載測試

Helm 沒有網域劫持，統一走 `nexus.lab`：

```bash
CERT=ansible/.deploy/certs/nexus.crt

helm create demo && helm package demo   # 產生 demo-0.1.0.tgz

# 上傳（PUT 到 repo 根目錄，Nexus 會自動維護 index.yaml）
curl -sS -o /dev/null -w '%{http_code}\n' \
  --resolve nexus.lab:443:127.0.0.1 --cacert "$CERT" \
  --upload-file demo-0.1.0.tgz "https://nexus.lab/repository/helm-hosted/"

# index.yaml 應該要列出剛才上傳的 chart
curl -sS --resolve nexus.lab:443:127.0.0.1 --cacert "$CERT" \
  https://nexus.lab/repository/helm-hosted/index.yaml

# 下載
curl -sS --resolve nexus.lab:443:127.0.0.1 --cacert "$CERT" \
  "https://nexus.lab/repository/helm-hosted/demo-0.1.0.tgz" -o roundtrip.tgz
```

✅ **2026-09-17** — 上傳回 `200`；`index.yaml` 正確列出 `demo` 0.1.0；下載回來的 `.tgz` 與原檔 `cmp` 逐位元組相同。

### 真實用戶端（`helm repo add`），正式環境

```bash
echo "<NGINX_IP> nexus.lab" | sudo tee -a /etc/hosts
sudo cp nexus.crt /usr/local/share/ca-certificates/nexus-lab.crt   # 檔名需以 .crt 結尾
sudo update-ca-certificates

helm repo add nexus https://nexus.lab/repository/helm-hosted/
helm repo update
helm search repo nexus
```

系統已信任 `nexus.crt` 就不用額外參數；沒信任的話可加 `--ca-file nexus.crt`（新版 Helm 是 `--ca-file`，不是 `--cafile`）。

---

## PyPI — 上傳 / 下載測試

`pip install`（讀）全程匿名；`twine upload`（寫）需要 `pypi-uploader` 帳密——本專案唯一「讀寫權限不同」的格式（其餘三個格式皆匿名讀寫）。

### 直接路徑（`nexus.lab`）

```bash
CERT=ansible/.deploy/certs/nexus.crt

# 上傳（twine，需 pypi-uploader 帳密）
TWINE_USERNAME=pypi-uploader TWINE_PASSWORD=<nexus_pypi_uploader_password> \
  twine upload --repository-url https://nexus.lab/repository/pypi-hosted/ \
  --cert "$CERT" dist/*

# 下載（pip，匿名，免帳密）
pip install --index-url https://nexus.lab/repository/pypi-hosted/simple/ --cert "$CERT" <pkg>
```

✅ **2026-09-22** — wheel + sdist 上傳皆成功；直接路徑安裝、import 正常。

### 網域劫持路徑（`pypi.org` / `upload.pypi.org`）

```bash
curl -sS --resolve pypi.org:443:127.0.0.1 --cacert "$CERT" https://pypi.org/simple/<pkg>/
```

✅ **2026-09-22** — 確認 `pypi-hosted` 的 simple index 回傳**相對路徑**的 href（`../../packages/<pkg>/<version>/<file>#sha256=...`），所以劫持 vhost 要同時改寫 `/simple/` 與 `/packages/` 兩個路徑——跟 Maven 只需要改寫一個 `/maven2/` 不同。

### 真實用戶端（`pip install` / `twine upload`），正式環境

pip/twine 用內建的 `certifi` CA 清單驗證 TLS，**不是** OS 信任庫（跟 curl `--cacert`、docker 的 `certs.d/`、JVM `keytool` 都不一樣）：

```bash
export PIP_CERT=nexus.crt
export TWINE_CERT=nexus.crt
echo "<NGINX_IP> pypi.org upload.pypi.org" | sudo tee -a /etc/hosts

# 上傳 — 走 pypi-uploader 帳密，非匿名
TWINE_USERNAME=pypi-uploader TWINE_PASSWORD=<nexus_pypi_uploader_password> twine upload dist/*

# 安裝 — 匿名，免帳密
pip install <pkg>
```

✅ **2026-09-22**（本沙箱無 sudo，改用一個掛進 nginx 所在 compose network、`--add-host` 直接指到 nginx 容器 IP 的 container 模擬劫持，效果等同改 `/etc/hosts`）——**完全未帶 `--index-url`/`--repository-url`** 的 `twine upload`／`pip install` 皆成功：twine 印出 `Uploading distributions to https://upload.pypi.org/legacy/`（預設值，證明真的打中劫持網域，不是意外成功）；pip 印出 `Downloading https://pypi.org/packages/...`（證明相對 href 正確解析回劫持網域）；下載回來的套件 `import` 正常。另外驗證：`twine upload` 用錯密碼被拒 `401 Unauthorized`（證明上傳確實有認證閘門）；`pip install` 全程無任何帳密。

> ⚠️ **劫持後這台機器連不到真正的 PyPI**——跟 Maven Central 劫持一樣的取捨，且已實測驗證：同一個劫持過的 container 想額外 `pip install twine`（走真正的 PyPI）會逾時失敗，因為 `pypi.org` 已經解析到本地 Nexus。若還要用**附錄 B**「連網環境下載 ansible-core」，那一步務必在**沒有套用這個劫持**的機器上執行，否則 `pip download` 會改抓（通常是空的）本地 `pypi-hosted`，而非真正的 PyPI，且不會有明顯錯誤（只是找不到套件或裝到舊版）。

---

## 疑難排解

| 症狀 | 處理 |
|---|---|
| `x509: certificate signed by unknown authority` | 對應主機名的 `ca.crt` 未放到 `/etc/docker/certs.d/<主機名>/`（docker），或系統/JVM 未信任 `nexus.crt`（見各段落「真實用戶端」小節） |
| `x509: certificate is valid for X, not Y` | 該主機名不在 `nexus_cert_sans` 裡。加進 `inventory/group_vars/all.yml` 的 `nexus_registry_extra_server_names`／`nexus_maven_extra_server_names`，重新跑 `ansible-playbook site.yml`——憑證會自動偵測 SAN 異動並重簽＋reload nginx |
| `server gave HTTP response to HTTPS client` | 連到了明文的 `8081`/`5000`；外部應該用 `nexus.lab`/`registry.lab`（經 nginx 443） |
| 匿名 push 被拒（401/403） | 檢查對應格式的匿名權限（`nx-anon-docker-rw`/`nx-anon-maven-rw`/`nx-anon-helm-rw`）是否已掛在 `nx-anon-rw` 角色上；正常情況下每次執行都會確保這件事，若懷疑沒生效可重跑 `ansible-playbook site.yml --tags docker,maven,helm` |
| 劫持測試拉不到（curl 直接連線失敗，不是 404） | 先確認憑證 SAN 有涵蓋該名稱（`openssl x509 -in nexus.crt -noout -text \| grep DNS`），沒有就照上一條處理；SAN 有涵蓋但仍失敗，檢查 nginx 是否已 reload（`docker compose -p nexus exec nginx nginx -T \| grep server_name`） |
| Docker `GET /v2/` 回 401 | **正常**——Bearer Token 挑戰，真正的 `docker` client 會自動處理；用 `docker pull/push` 測試即可 |
| 容器啟動即結束、log 權限錯誤 | `nexus-data` 目錄需 UID 200 擁有；`nexus.yml` 已自動處理初始化，若手動改過該目錄需重新跑一次讓它修正權限 |
| Maven 權限建立失敗 / 名稱格式錯 | Nexus API 的 privilege `format` 要填 `maven2`（不是 `maven`）；建 repo 的端點才是 `.../repositories/maven/hosted` |
| sbt 報 SSL / PKIX `unable to find valid certification path` | sbt 走 JVM truststore，需用 `keytool` 把 `nexus.crt` 匯入 Java cacerts；系統的 `update-ca-certificates` 對 JVM 無效 |
| `pip`/`twine` 報 `CERTIFICATE_VERIFY_FAILED` | pip/twine 走內建 `certifi` CA 清單，**不是** OS 信任庫；系統的 `update-ca-certificates` 對它們無效。設定 `PIP_CERT`/`TWINE_CERT` 指向 `nexus.crt`（或 `pip install --cert`/`twine upload --cert`） |
| `twine upload` 回 `401 Unauthorized` | pypi-hosted 的上傳不是匿名——確認 `TWINE_USERNAME=pypi-uploader`／`TWINE_PASSWORD` 正確；純讀取（`pip install`）不需要這組帳密 |

---

## 附錄 A — Maven/sbt 大型專案離線相依作業流程

正文「Maven — 上傳/下載測試」只用 trivial 檔案驗證劫持轉送是否正確，並點出限制：**專案若相依公開套件（如 `scala-library`），未鏡像會 404**。本附錄補上「真實 Spark 專案」的完整離線流程——**網路環境**用 Docker 容器解析並下載一個 sbt + Spark 專案的**全部**相依，搬進**離線環境**灌進 `maven-hosted`，最後讓**一行都沒改**的專案在離線端 `sbt assembly` 成功。

流程中的 `nexus.crt` 即「佈署指令」／`ansible/README.md` 產生的那份憑證（單一憑證涵蓋所有 SAN，含 `repo1.maven.org`）。

| 階段 | 環境 | 做什麼 |
|---|---|---|
| A.1 | 網路 | 用 `sbtscala/scala-sbt` 容器暖 coursier 快取、隔離 dry-run 驗證捕捉完整 |
| A.2 | 網路 | `docker save` 映像 ＋ 打包 coursier 快取裡的 Maven 樹 |
| A.3 | 離線 | `docker load`，把整棵 Maven 樹批次 PUT 進 `maven-hosted` |
| A.4 | 離線 | 零修改專案在容器內 `sbt clean assembly`，全程只從 `maven-hosted` 取件 |

### A.0 前置與已知限制（請先讀）

**1) Spark 3.2 官方不支援 Java 17 —— 但本附錄的範圍不受影響**

| 動作 | JDK 17 + Spark 3.2 |
|---|---|
| `sbt update` / `compile` / `assembly`（產出 fat jar） | **可正常執行**（只處理 bytecode / classpath，不初始化 Spark runtime） |
| 啟動 `SparkSession`（`sbt run`、建立本地 SparkContext 的測試、`spark-submit`） | **失敗**：`java.lang.IllegalAccessError` / `InaccessibleObjectException`。Spark 官方 JDK 17 支援始於 **3.3**（SPARK-33772），3.2 沒有 |

A.1–A.4 全部是「解析 + 打包」，落在可正常執行那一列。`sbt-assembly` 自 **1.0.0 起預設不跑測試**，故 A.4 不會被「測試啟動 SparkSession」拖累（範例 `build.sbt` 仍明確設 `assembly / test := {}`）。

**2) 映像**

正確名稱是 **`sbtscala/scala-sbt`**（官方；非 `scala/scala-sbt`）。符合 sbt 1.8.3 + Java 17 的 tag：

```
sbtscala/scala-sbt:eclipse-temurin-jammy-17.0.5_8_1.8.3_2.13.10
  digest sha256:54e286fe8bbb82b7627392f753da6bb1b55d6000deef2ea54a8ba120ffe549c9
```

映像內建的 Scala（2.13.10）與專案無關：專案 build 依 `build.sbt` 的 `scalaVersion`（Spark 3.2 → **2.12.15**），sbt 於首次 build 自行解析。**用 digest 釘死映像**——tag 是唯一會在網路端與離線端之間悄悄改變的東西。

**3) 相依捕捉手法（核心）**

sbt 1.8.3 預設用 **coursier**，其快取本身就是 Maven 版面配置、且 `.sha1` 旁檔已隨附：

```
~/.cache/coursier/v1/https/repo1.maven.org/maven2/<group>/<artifact>/<version>/…
```

流程是「暖快取 → `tar` 這一個目錄 → 逐檔 PUT 進 `maven-hosted`」，**不需**從相依圖重建 repo，校驗和問題同時解決。

- **完整性 gate**：暖快取後 `ls ~/.cache/coursier/v1/https/`，**只能出現 `repo1.maven.org`**。若冒出 `repo.scala-sbt.org` 等其他 host，代表該 artifact 離線會 404（劫持只改寫 `/maven2/`）。
- **不要**把 `repo.scala-sbt.org` 加進 `/etc/hosts`：留著讓 DNS 快速失敗、sbt fall through 到 Central；加了反而讓 plugin resolver 打到只認 `/maven2/` 的 vhost 拿 404。
- sbt **自己的 boot 相依**（`scala-compiler` 2.12.17、sbt modules）首次啟動也會進同一 coursier 快取，會一併被鏡像——否則離線端 sbt 還沒讀到 `build.sbt` 就 404。

**4) sbt-assembly 不裝在映像上，而是像相依一樣從 `maven-hosted` 解析**

`project/plugins.sbt` 宣告 `addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "2.1.5")`——專案檔，線上／離線都不改。sbt 一啟動、載入 build 時就會把外掛 JAR ＋ transitive 相依從 `repo1.maven.org` 解析（劫持後 = `maven-hosted`）；sbt-assembly 2.1.5 發佈在 Central（`/maven2/`，劫持涵蓋）。所以「離線能跑 `sbt assembly`」的前提是 **A.1 Step 2 已把外掛 ＋ 相依抓進快取 → A.3 已灌進 `maven-hosted`**——映像本身只給 JDK 17 ＋ sbt 1.8.3 launcher，不含任何專案外掛。

（選配）要讓外掛完全不經 `maven-hosted`：網路端用 `docker commit`／Dockerfile 把暖好的 `~/.sbt` ＋ coursier 快取烤進映像。本附錄不採此法（A.4 用全新快取才能證明 `maven-hosted` 完整）；A.2 的 `bundle/caches` 選配即為此 fallback。

> **範圍與代價**：本附錄把 Spark 相依打進 fat jar（compile scope），是對鏡像完整性最強的驗證，代價是 über-jar 數百 MB、`bundle/maven-repo` 約 0.5–1 GB、artifact 約 150–250 個，A.2 打包時預留空間。
>
> **附註（供之後 `spark-submit` 用，非本附錄核心）**：若最終要 `spark-submit`，慣例把 Spark 改成 `% Provided`（叢集自帶 Spark），鏡像流程不變，只差 jar 內容大小。目標叢集需求：JVM 8u201+／11（Spark 3.2 官方支援範圍，JDK 17 上 Spark 3.2 本身起不來，與這份 jar 無關）＋ Scala 2.12 build 一致；JDK 17 只是拿來編譯，Scala 2.12.15 一律產出 Java 8 bytecode（class 版本 52），不會污染 jar（勿加 `-release:17` 之類拉高 target）。Java 11 + Arrow 需另加 `-Dio.netty.tryReflectionSetAccessible=true`。

---

### A.1 網路環境：解析並下載完整相依

在一台可連網、裝好 Docker 的機器執行。

**Step 1 — 範例專案骨架**（真實專案請直接用自己的，骨架僅示範必要設定）

```
offline-spark-demo/
├── build.sbt
├── project/
│   ├── build.properties
│   └── plugins.sbt
└── src/main/scala/example/Job.scala
```

`project/build.properties`：

```
sbt.version=1.8.3
```

`project/plugins.sbt`：

```scala
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "2.1.5")
```

`build.sbt`：

```scala
ThisBuild / scalaVersion := "2.12.15"      // Spark 3.2 的 Scala 2.12 線
ThisBuild / organization := "com.example"
ThisBuild / version      := "0.1.0"

lazy val root = (project in file("."))
  .settings(
    name := "offline-spark-demo",
    libraryDependencies ++= Seq(
      "org.apache.spark" %% "spark-core" % "3.2.4",   // compile scope：打包進 jar
      "org.apache.spark" %% "spark-sql"  % "3.2.4"
    ),

    // sbt-assembly >= 1.0.0 預設就不跑測試，明示以防未來版本改動
    assembly / test := {},
    assembly / assemblyJarName := s"${name.value}-assembly-${version.value}.jar",

    // Spark über-jar 常見衝突的處理；如遇 deduplicate 衝突，在此補規則（見 Step 2 註）
    assembly / assemblyMergeStrategy := {
      case PathList("META-INF", "services", _*)         => MergeStrategy.concat
      case PathList("META-INF", "MANIFEST.MF")          => MergeStrategy.discard
      case PathList("META-INF", "NOTICE" | "NOTICE.txt" | "LICENSE" | "LICENSE.txt" | "DEPENDENCIES")
                                                        => MergeStrategy.discard
      case PathList("META-INF", xs @ _*) if xs.nonEmpty &&
             Set(".sf", ".dsa", ".rsa").exists(xs.last.toLowerCase.endsWith) => MergeStrategy.discard
      case PathList("META-INF", "versions", _*)         => MergeStrategy.first
      case "module-info.class"                          => MergeStrategy.discard
      case x if x.endsWith("/module-info.class")        => MergeStrategy.discard
      case "reference.conf" | "application.conf"        => MergeStrategy.concat
      case x if x.endsWith(".properties")               => MergeStrategy.first
      case x if x.endsWith(".proto")                    => MergeStrategy.first
      case x if x.endsWith(".class")                    => MergeStrategy.first
      case _                                            => MergeStrategy.deduplicate
    }
  )
```

`src/main/scala/example/Job.scala`——**一定要有真的用到 Spark 的程式碼**，否則 compile 成功不代表 Spark 相依有解析到：

```scala
package example

import org.apache.spark.sql.SparkSession

object Job {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder().appName("offline-demo").master("local[1]").getOrCreate()
    println(spark.range(10).count())
    spark.stop()
  }
}
```

**Step 2 — 用與離線端相同的映像暖快取**（coursier / sbt / ivy 快取掛成持久 volume）

```bash
IMG=sbtscala/scala-sbt@sha256:54e286fe8bbb82b7627392f753da6bb1b55d6000deef2ea54a8ba120ffe549c9
mkdir -p bundle/caches/coursier bundle/caches/sbt bundle/caches/ivy

docker run --rm \
  -v "$PWD/offline-spark-demo":/work -w /work \
  -v "$PWD/bundle/caches/coursier":/root/.cache/coursier \
  -v "$PWD/bundle/caches/sbt":/root/.sbt \
  -v "$PWD/bundle/caches/ivy":/root/.ivy2 \
  "$IMG" sbt -Dsbt.color=false \
    "reload" "update" "Test/update" "compile" "Test/compile" "assembly"
```

- `update` + `Test/update` 涵蓋 compile / test 兩個 config 的解析。
- `assembly` 會拉下 sbt-assembly 外掛本身及其相依，並實際做一次打包。
- sbt 首次啟動也會把自己的 boot 相依（`scala-compiler` 2.12.17 等）解析進同一 coursier 快取。

> 若 `assembly` 因 `deduplicate: different file contents found in the following` 失敗 → **在這裡（網路端）** 反覆調 `assemblyMergeStrategy` 直到過，不要帶進離線機房。
>
> 若映像非以 root 執行（先 `docker run --rm "$IMG" id` 確認），把上面三個 volume 改掛到該使用者的 home，或加 `--user root`。

**Step 3 — 完整性 gate**

```bash
ls bundle/caches/coursier/v1/https/
# 期望只有: repo1.maven.org
# 若出現 repo.scala-sbt.org 或其他 host → 見 A.0(3)：該 artifact 離線會 404，需處理

# 記錄規模
find bundle/caches/coursier/v1/https/repo1.maven.org/maven2 -type f | wc -l
du -sh  bundle/caches/coursier/v1/https/repo1.maven.org/maven2
```

**Step 4 — 出貨前的網路隔離 dry-run（唯一能證明捕捉完整的步驟）**

```bash
docker run --rm --network none \
  -v "$PWD/offline-spark-demo":/work -w /work \
  -v "$PWD/bundle/caches/coursier":/root/.cache/coursier \
  -v "$PWD/bundle/caches/sbt":/root/.sbt \
  -v "$PWD/bundle/caches/ivy":/root/.ivy2 \
  "$IMG" sbt -Dsbt.color=false --offline "clean" "assembly"
```

- 成功產出 `target/scala-2.12/offline-spark-demo-assembly-0.1.0.jar` → 捕捉完整。
- 失敗（`unresolved dependency`）→ 把缺的座標暫時加進 `build.sbt` 的 `libraryDependencies`，回 Step 2 重跑（`update` 會把它一併拉進快取）。

> `--network none` 時 DNS 直接失敗、sbt 容忍無法連線的 resolver；真實離線端 `repo1.maven.org` 會解析到 nginx，行為略有不同，但「所有需要的檔案都在快取裡」這件事兩者一致——這正是這一步要證明的。

---

### A.2 打包搬運

```bash
# 1) 映像：用 digest 存 tar 會遺失 tag（load 後變 <none>:<none>）→ 先固定成本地 tag
docker pull "$IMG"
docker tag  "$IMG" scala-sbt-offline:1.8.3-jdk17
mkdir -p bundle/images
docker save scala-sbt-offline:1.8.3-jdk17 -o bundle/images/scala-sbt-1.8.3-jdk17.tar

#    網路端先做一次 load 往返，確認 tag 還在
docker rmi scala-sbt-offline:1.8.3-jdk17 && docker load -i bundle/images/scala-sbt-1.8.3-jdk17.tar && docker images | grep scala-sbt-offline

# 2) 只取 repo1.maven.org 那棵樹（這就是要灌進 maven-hosted 的內容）
mkdir -p bundle/maven-repo
cp -a bundle/caches/coursier/v1/https/repo1.maven.org/maven2/. bundle/maven-repo/

# 3) 產生清單
{
  echo "image        : $IMG"
  echo "local tag    : scala-sbt-offline:1.8.3-jdk17"
  echo "sbt          : 1.8.3"
  echo "scala (專案) : 2.12.15"
  echo "spark        : 3.2.4"
  echo "sbt-assembly : 2.1.5"
  echo "artifacts    : $(find bundle/maven-repo -type f | wc -l) files, $(du -sh bundle/maven-repo | cut -f1)"
} > bundle/MANIFEST.txt

# 4) 打包（bundle/caches 為選配 fallback，會讓體積翻倍）
tar -czf nexus-maven-offline-bundle.tar.gz \
    bundle/images/scala-sbt-1.8.3-jdk17.tar bundle/maven-repo bundle/MANIFEST.txt
sha256sum nexus-maven-offline-bundle.tar.gz > nexus-maven-offline-bundle.tar.gz.sha256
```

以實體介質傳入離線環境。

---

### A.3 離線環境：推送至 maven-hosted

**前置**：Nexus 已依 `ansible/README.md` 佈署完成，`maven-hosted` 存在、匿名讀寫已開、`/maven2/` 劫持生效。以下在 **Nexus 主機本機**執行，走 `localhost:8081`（不經 nginx、不需憑證、匿名寫入）。

**Step 1 — 解包、載入映像**

```bash
sha256sum -c nexus-maven-offline-bundle.tar.gz.sha256
tar -xzf nexus-maven-offline-bundle.tar.gz
docker load -i bundle/images/scala-sbt-1.8.3-jdk17.tar
```

**Step 2 — 先試灌一組（驗 `layoutPolicy: STRICT`）**

```bash
NEXUS=http://localhost:8081/repository/maven-hosted
BASE=bundle/maven-repo/org/apache/spark/spark-core_2.12/3.2.4

curl -sf --upload-file "$BASE/spark-core_2.12-3.2.4.pom" \
  "$NEXUS/org/apache/spark/spark-core_2.12/3.2.4/spark-core_2.12-3.2.4.pom"      # 期望 201
curl -sf --upload-file "$BASE/spark-core_2.12-3.2.4.jar" \
  "$NEXUS/org/apache/spark/spark-core_2.12/3.2.4/spark-core_2.12-3.2.4.jar"
```

> coursier 對**固定版本**通常不抓 group-level `maven-metadata.xml`，所以 `bundle/maven-repo` 裡多半沒有這種檔——屬正常。Step 3 的 `find` 會把「有的」一起帶上；若 A.4 解析時 sbt 仍向 `.../<artifact>/maven-metadata.xml` 要（少見，多為 SNAPSHOT 或版本範圍），才需從網路端另外抓那一支補進來。

被 STRICT 擋（`400` / `Invalid path`）→ 對**既有** repo 暫時放寬版面檢查，灌完再收回：

```bash
NEXUS_API=http://localhost:8081
ADMIN_PW='<nexus_admin_password 的值>'

# 取現況 → 改 layoutPolicy → PUT 回
curl -s -u "admin:$ADMIN_PW" "$NEXUS_API/service/rest/v1/repositories/maven/hosted/maven-hosted" > /tmp/mh.json
sed -i 's/"layoutPolicy"[[:space:]]*:[[:space:]]*"STRICT"/"layoutPolicy":"PERMISSIVE"/' /tmp/mh.json
curl -s -u "admin:$ADMIN_PW" -X PUT "$NEXUS_API/service/rest/v1/repositories/maven/hosted/maven-hosted" \
  -H 'Content-Type: application/json' --data-binary @/tmp/mh.json     # 回 204
# …執行 Step 3…
# 灌完收回 STRICT（把上面的 sed 反過來再 PUT 一次）
```

**Step 3 — 批次上傳整棵樹**

```bash
: > /tmp/upload-fail.log
cd bundle/maven-repo
find . -type f ! -name '_remote.repositories' ! -name '*.lastUpdated' | sed 's|^\./||' \
| while read -r p; do
    code=$(curl -s -o /dev/null -w '%{http_code}' --upload-file "$p" "$NEXUS/$p")
    case "$code" in
      200|201|204) ;;
      *) echo "FAIL $code  $p" >> /tmp/upload-fail.log ;;
    esac
  done
cd -
[ -s /tmp/upload-fail.log ] && { echo "有失敗："; cat /tmp/upload-fail.log; } || echo "全部上傳成功"
```

- `.sha1` / `.md5` 旁檔是普通檔，會一起被上傳——coursier 之後才驗得過。
- 已排除 coursier 內部檔（`_remote.repositories`、`*.lastUpdated`）。
- 要加速可把管線換成 `xargs -P8 -I{} curl …`；STRICT/PERMISSIVE 下順序不影響結果。

**Step 4 — 抽驗**

```bash
curl -sI "$NEXUS/org/scala-lang/scala-library/2.12.15/scala-library-2.12.15.jar" | head -1   # 200
curl -sI "$NEXUS/org/apache/spark/spark-sql_2.12/3.2.4/spark-sql_2.12-3.2.4.jar"     | head -1   # 200
```

---

### A.4 離線環境：驗證（專案零修改）

**目標**：把**未加任何 `resolvers`、一行都沒改**的專案，在離線端的容器裡 `sbt clean assembly` 成功，全程只從 `maven-hosted` 取件。

**前置**（離線端測試機，一次性）

- 準備一份「佈署指令」段落產生的 `nexus.crt`（SAN 需含 `repo1.maven.org`）。
- 知道 nginx 主機 IP（下稱 `<NGINX_IP>`）。
- **容器的 `/etc/hosts` 不繼承主機的**——正文測試段設在主機 `/etc/hosts` 的劫持，容器看不到，必須用 `--add-host` 再給一次。

**Step 1 — 確認映像使用者**

```bash
docker run --rm scala-sbt-offline:1.8.3-jdk17 id
```

若非 root：直接寫 `$JAVA_HOME/lib/security/cacerts` 會權限失敗（易誤判成憑證問題）。下面的指令用 `--user root` 繞過；或改把 cacerts 複製到可寫路徑，再以 `-Djavax.net.ssl.trustStore=/tmp/cacerts` 指過去。

**Step 2 — 零修改專案跑 assembly**

```bash
docker run --rm --user root \
  --add-host repo1.maven.org:<NGINX_IP> \
  --add-host repo.maven.apache.org:<NGINX_IP> \
  -v "$PWD/offline-spark-demo":/work -w /work \
  -v "$PWD/nexus.crt":/tmp/nexus.crt:ro \
  -e COURSIER_CACHE=/tmp/cr-fresh \
  scala-sbt-offline:1.8.3-jdk17 bash -c '
    keytool -importcert -alias nexus-lab -file /tmp/nexus.crt \
      -keystore "$JAVA_HOME/lib/security/cacerts" -storepass changeit -noprompt
    exec sbt -Dsbt.color=false "clean" "assembly"
  '
```

- **不 mount** 任何 A.1 的快取 volume、`COURSIER_CACHE` 指向全新目錄 → 證明相依真的來自 `maven-hosted`，不是殘留快取。
- **不加** `--offline` → 讓 sbt 真的走網路到 `repo1.maven.org`（= nginx = `maven-hosted`）。
- `--user root` ＋ mount `offline-spark-demo` → 容器產生的 `target/` 會是 root 所有。要重跑或清理時用 `sudo rm -rf offline-spark-demo/target`（A.1 的 `bundle/caches` 同理）。

**預期**：解析全部相依 → compile → assembly → 產出 `target/scala-2.12/offline-spark-demo-assembly-0.1.0.jar`。

**驗證 fat jar**（Spark 已打包進 jar）：

```bash
unzip -l offline-spark-demo/target/scala-2.12/offline-spark-demo-assembly-0.1.0.jar \
  | grep -E 'example/Job.class|org/apache/spark/sql/SparkSession.class'
```

兩個都在 → 整條 Spark transitive closure 都從 `maven-hosted` 解析且打包成功。

> **（選配）用 A.1 快取隔離問題**：若 Step 2 失敗、想確認是「鏡像灌製」還是「相依捕捉」出錯——改 mount A.1 的 coursier 快取 volume ＋ 加 `--offline` 再跑一次 `assembly`。能過 → 問題在 A.3 灌製；還是不能過 → 問題在 A.1 捕捉。
>
> **註腳（非本附錄範圍）**：日後要在 JDK 17 上實際跑 Spark 3.2（非官方支援），需加一組 `--add-opens` module 開放旗標（見 Spark 3.3+ `JavaModuleOptions` 原始碼，此處不贅列）。

---

### A.5 疑難排解（附錄專屬，補正文表格）

| 症狀 | 處理 |
|---|---|
| A.1 Step 4 dry-run `unresolved dependency: X` | 漏抓某 config / classifier。把座標暫加進 `build.sbt` 的 `libraryDependencies`，回 Step 2 重跑 |
| `ls coursier/v1/https/` 出現 `repo.scala-sbt.org` 等 | 某 sbt 外掛非從 Central 解析。改用 Central 上有的外掛版本（sbt-assembly 2.x 在 Central）；**勿**把該 host 加進 `/etc/hosts` |
| A.1 `assembly` 報 `deduplicate: different file contents` | **在網路端**補 `assemblyMergeStrategy` 規則直到過，不要帶進離線機房 |
| A.3 上傳回 `400` / `Invalid path` | `maven-hosted` `layoutPolicy` 暫設 `PERMISSIVE`（A.3 Step 2），灌完收回 `STRICT`；或改 `mvn deploy:deploy-file` |
| A.3 上傳回 `401` / `403` | 匿名缺 `maven2` 的 `ADD` / `EDIT`（Nexus privilege 的 `format` 要 `maven2`、且已指派給匿名角色） |
| A.4 `keytool` 報 `Permission denied` 寫 cacerts | 映像非 root。加 `--user root`，或複製 cacerts 到可寫路徑並用 `-Djavax.net.ssl.trustStore=` 指過去 |
| A.4 `PKIX path building failed` / `unable to find valid certification path` | `nexus.crt` 沒匯入**容器內**的 JVM cacerts；或憑證 SAN 未含 `repo1.maven.org`（重簽 → 重匯 → 重載 nginx） |
| A.4 解析中途 `404 maven-metadata.xml` | 先確認 `bundle/maven-repo` 是否真有該檔（coursier 對固定版本通常不抓）；有就補傳，沒有就從網路端單獨抓 `https://repo1.maven.org/maven2/<group>/<artifact>/maven-metadata.xml` 再灌 |
| A.4 能過但懷疑吃到殘留快取 | 確認沒 mount 任何 A.1 快取 volume、`COURSIER_CACHE` 指向全新目錄 |
| A.4 `sbt` 一啟動就 404（還沒讀 `build.sbt`） | sbt 自己的 boot 相依（`scala-compiler` 2.12.17、sbt modules）沒鏡像進 `maven-hosted`。回 A.1 用同一映像暖快取後重灌 |
| 執行 Spark 時 `InaccessibleObjectException` | JDK 17 + Spark 3.2 的已知限制。加 A.4 註腳的 `--add-opens`，或改用 JDK 11 跑 runtime |

---

## 附錄 B — 離線環境佈署 Ansible 本身

正文與 `ansible/README.md` 都預設**目標機器上已經有 `ansible-playbook` 可用**——README 的「Run」一節直接從 `ansible-playbook site.yml --become` 開始。若目標機器**完全沒有網路**（連 apt/pip 來源都連不到，不只是連不到 Docker Hub / Maven Central），`ansible-playbook` 這個指令本身要怎麼上去，是 README 沒覆蓋的最後一塊拼圖。本附錄：在**連網環境**打包 Ansible 執行檔＋相依，搬進離線機器裝起來，再跑通整個 `ansible/` 專案。

### B.0 前置與已知限制（請先讀）

**1) 這個專案不需要 Ansible Collections，大幅簡化離線打包**

`roles/nexus/` 底下所有 task 用的模組全部是 `ansible.builtin.*`；`docker compose` 是用 `ansible.builtin.command` 直接呼叫 CLI，不是 `community.docker` 的模組。這代表**只要 `ansible-core` 本身能跑，就不需要 `ansible-galaxy collection install`**、也不用打包 collection tarball——離線清單只有「Python + `ansible-core` 套件本身」。

**2) 只需要 `ansible-core`，不要裝 `ansible`**

PyPI／apt 上的 `ansible`（不帶 `-core`）是會順便拉進幾百個 collection 的 meta package，對只用 `ansible.builtin.*` 的這個專案完全用不到，只會徒增體積與失敗面。只下載、只裝 `ansible-core`。

**3) `ansible_connection=local`，目標機＝控制機**

`inventory/hosts` 是 `localhost ansible_connection=local`——這個專案不是控制機透過 SSH 佈署到遠端機器，而是直接在同一台機器上執行 `ansible-playbook`。所以「離線佈署 Ansible」的目標就是**讓那台無網路的機器本機能執行 `ansible-playbook`**，不涉及 SSH／`ansible_python_interpreter`。前提（視同已具備）：機器已有 Python 3、Docker Engine ＋ `docker compose` plugin，且使用者在 `docker` 群組。

**4) 打包載體用 pip wheel，不用系統套件管理員**

`ansible-core` 與其相依（`jinja2`、`PyYAML`、`cryptography`、`packaging`、`resolvelib`、`MarkupSafe`、`cffi`、`pycparser`）在 PyPI 上都有預編譯 manylinux wheel，離線端不需要編譯工具鏈。**連網端與離線端的 CPU 架構（x86_64/arm64）與 Python 大版本（如 3.11）要一致**，否則 `pip install --no-index` 會報 `No matching distribution found`——最保險是在跟離線機器同架構、同 Python 大版本的連網機器上執行 B.1。

---

### B.1 連網環境：下載 Ansible 執行檔＋相依，順便存好映像

⚠️ 這一步要在**沒有套用 PyPI 劫持**（見主文「PyPI — 上傳 / 下載測試」的 `pypi.org` 網域劫持）的機器上執行——劫持後 `pip download` 會改抓本地（通常是空的）`pypi-hosted`，而非真正的 PyPI，且不一定會報明顯錯誤。

**在 `ansible/` 專案目錄的上一層**建一個純暫存的搬運目錄（不進版控、不進專案目錄，用完即丟）：

```bash
mkdir -p stage/ansible-offline
pip download ansible-core -d stage/ansible-offline
ls stage/ansible-offline/*.whl > stage/MANIFEST-ansible.txt

docker pull sonatype/nexus3:3.93.2 && docker save sonatype/nexus3:3.93.2 -o stage/nexus3.tar
docker pull nginx:1.27-alpine       && docker save nginx:1.27-alpine       -o stage/nginx.tar
```

✅ **2026-09-18**（anaconda pip，Python 3.11）— `pip download` 共抓下 9 個 wheel（`ansible_core-2.19.13`、`jinja2-3.1.6`、`pyyaml-6.0.3`、`cryptography-50.0.1`、`cffi-2.1.1`、`pycparser-3.0`、`packaging-26.3`、`resolvelib-1.2.1`、`markupsafe-3.0.3`），全部是 `manylinux*`／`py3-none-any` 預編譯檔，沒有任何 sdist 需要現場編譯。

---

### B.2 打包搬運

從 `ansible/` 的上一層執行，把暫存目錄與專案原始碼（不含任何二進位檔）一起打包：

```bash
tar -czf nexus-offline-bundle.tar.gz stage ansible
sha256sum nexus-offline-bundle.tar.gz > nexus-offline-bundle.tar.gz.sha256
```

以實體介質（USB、燒錄光碟等）傳入離線環境。

---

### B.3 離線環境：安裝 Ansible ＋ 執行佈署

**Step 1 — 解包、校驗**

```bash
sha256sum -c nexus-offline-bundle.tar.gz.sha256
tar -xzf nexus-offline-bundle.tar.gz    # 產生 ./ansible/ 與 ./stage/
```

**Step 2 — 建一個獨立 venv，完全離線安裝 `ansible-core`**

```bash
python3 -m venv /opt/ansible-venv
/opt/ansible-venv/bin/pip install --no-index --find-links=stage/ansible-offline ansible-core
```

`--no-index` 讓 pip 完全不嘗試連 PyPI——離線套件目錄裡缺任何一個相依 wheel，這一步會直接報錯，不會靜默 fallback 到網路（也沒有網路可 fallback），所以這一步成功本身就是「不需要網路」的證明。

✅ **2026-09-18** — 全新 venv 中 `pip install --no-index --find-links=...` 成功安裝 9 個套件；`ansible-playbook --version` 正確回報 `ansible-core 2.19.13`、Python 3.11.7，`ansible collection location` 列出的路徑實際上是空的（本專案不用 collection，見 B.0(1)）。

**Step 3 — 跑通整個專案**

```bash
docker load -i stage/nexus3.tar
docker load -i stage/nginx.tar
cd ansible
/opt/ansible-venv/bin/ansible-playbook site.yml --become        # 正式環境；本機測試可改 -e @test-vars.yml
```

> ⚠️ 若 `ansible-playbook` 報 `Ansible requires blocking IO on stdin/stdout/stderr`（常見於管線化輸出的終端環境），改用 `... > run.log 2>&1` 導向檔案再讀 log。

✅ **2026-09-18**（`-e @test-vars.yml` 本機模式，離線裝好的 `ansible-core 2.19.13` 執行）— 用 B.2 打包的同一份 `ansible/` 專案（含 docker-registry、maven-registry、helm）跑出 `failed=0`；跑完立刻 `curl` 打 `helm-hosted/index.yaml` 拿到 `200`，證明離線裝的 Ansible 執行出來的部署真的可用，不只是 playbook 沒報錯。測試後立刻 `docker compose -p nexus down` 清乾淨。

---

### B.4 疑難排解（附錄專屬）

| 症狀 | 處理 |
|---|---|
| `pip install --no-index` 報 `No matching distribution found for cryptography` 之類 | 連網端與離線端 CPU 架構或 Python 大版本不一致，manylinux wheel tag 對不上。回 B.1，改在同架構／同 Python 版本的機器上執行 `pip download` |
| `ansible-playbook: command not found` | 沒把 venv 的 `bin/` 加進 `PATH`，或忘了打完整路徑 `/opt/ansible-venv/bin/ansible-playbook`；也可 `source /opt/ansible-venv/bin/activate` 後直接下指令 |
| `ERROR: Ansible requires blocking IO on stdin/stdout/stderr` | 輸出被導到 pipe／非阻塞 handle。改成 `> run.log 2>&1` 導向檔案 |
| `couldn't resolve module/action` 或找 collection 相關錯誤 | 確認沒有在 task 裡誤用 `community.*`／`ansible.posix.*` 等模組——本專案設計上只用 `ansible.builtin.*`，若改動過 role 加了別的模組，離線環境會因缺該 collection 而失敗，需回連網端額外打包 |
| `docker load` 之後 `image not found` | 確認 `inventory/group_vars/all.yml` 的 `nexus_image`/`nginx_image` 版本字串跟 `docker save` 時的 tag 完全一致——compose 預設 `pull_policy: missing`，tag 對不上就不會用到本地已載入的映像，離線環境也無法臨時 pull |
