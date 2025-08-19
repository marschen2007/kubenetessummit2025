## 第 1 頁 – 主題與全貌
- **主題**：在 Kubernetes 中以 NGINX Plus Ingress + App Protect 建構可治理的 API Gateway  
- **能力疊加**  
  - TLS Proxy 與 L7 路由  
  - OpenAPI Schema 驗證（WAF）  
  - JWT 授權  
  - Rate Limiting  
- **成果**  
  - 安全可控的 API 入口  
  - 以 CRD 宣告式治理  
  - 支援灰階與自動化  
- **適用場景**  
  - 多租戶 API 平台  
  - 零信任接入  
  - 受監管產業審計  

---

## 第 2 頁 – 架構視圖與元件
- **Ingress（NGINX Plus）**
  - 以 DaemonSet 方式部署  
  - 終結 TLS，執行 L7 路由／重寫  
  - 透過 VirtualServer、Policy CRD 管理  
- **App Protect（WAF）**
  - OpenAPI → WAF Policy  
  - 阻擋不符 Schema 的 Payload  
  - 支援 Support ID 追蹤  
- **JWT 驗證**
  - RSA 簽發 Token  
  - JWK Secret 供驗簽  
  - 套用於敏感端點（POST）  
- **Rate Limiting**
  - 按 Authorization Header 分流  
  - 10 r/s 範例策略，超量回傳 429  
  - 使用 k6 驗證限流行為  

---

## 第 3 頁 – Lab 環境與基礎檢查
- microk8s 單節點；Ingress 監聽 80/443  
- 別名：`k`、`kubectl`、`microk8s kubectl` 等價  
- 工具腳本置於 `~/.local/bin`，支援 Tab 補齊  

```bash
# 健康檢查與資源觀察
microk8s-status.sh
list-all-k8s-lab-resources.sh  # 加 --start-over 可重置

# Ingress DaemonSet 檢視
k get daemonset nginx-ingress -n nginx-ingress

# Registry 察看
curl http://localhost:32000/v2/_catalog | jq
第 4 頁 – Task 01：建立 API 與前端服務
jobs.yaml → eclectic-jobs（toy API）

main.yaml → myapp（前端）

先以 NodePort 測試對外可用性

bash
複製
編輯
# 套用與驗證
k apply -f jobs.yaml
k apply -f main.yaml
k get svc

# 測試 URL
http://jobs.local:30020  # API
http://jobs.local:30010  # 前端
第 5 頁 – Task 02：TLS 與 VirtualServer 基礎
建立憑證與 Secret：

bash
複製
編輯
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout jobs.local.key -out jobs.local.crt \
  -config openssl.cnf -extensions req_ext

kubectl create secret tls jobs-local-tls \
  --key jobs.local.key --cert jobs.local.crt
VirtualServer（節選）：

yaml
複製
編輯
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: my-virtualserver
spec:
  host: jobs.local
  tls:
    secret: jobs-local-tls
  routes:
    - path: /get-job
      action:
        pass: eclectic-jobs-svc
    - path: /add-job
      action:
        pass: eclectic-jobs-svc-add
第 6 頁 – Task 03：前端整合與路徑重寫
yaml
複製
編輯
spec:
  routes:
    - path: /
      action:
        pass: myapp-svc
    - path: /get-job
      action:
        pass: eclectic-jobs-svc
      # 可選：重寫 /get-job → /
      # matches:
      #   - conditions: [{ variable: $request_uri, value: ^/get-job$ }]
      #     action: { pass: eclectic-jobs-svc, rewritePath: / }
第 7 頁 – Task 04：OpenAPI Schema 驗證（WAF）
OpenAPI 定義：

yaml
複製
編輯
paths:
  /add-job:
    post:
      requestBody:
        content:
          application/json:
            schema:
              type: array
              items: { type: string }
App Protect 配置：

yaml
複製
編輯
apiVersion: appprotect.f5.com/v1
kind: APPolicy
metadata: { name: jobs-openapi-appolicy }
spec:
  policy:
    applicationLanguage: utf-8
    openApiFiles:
      - link: https://raw.githubusercontent.com/.../jobs-openapi-spec.yaml
---
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata: { name: app-protect-policy }
spec:
  waf:
    apPolicy: jobs-openapi-appolicy
第 8 頁 – Task 05：JWT 授權驗證
簽發與配置：

bash
複製
編輯
# 簽發 JWT
create-signed-jwt.sh

# 存入 Secret
kubectl create secret generic jwk-secret \
  --from-file=jwk=/var/tmp/jwk/jwk.json \
  --type=nginx.org/jwk
JWT Policy：

yaml
複製
編輯
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata: { name: jwt-policy }
spec:
  jwt:
    realm: api
    secret: jwk-secret
第 9 頁 – Task 06：Rate Limiting 限流策略
策略定義：

yaml
複製
編輯
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata: { name: rate-limit }
spec:
  rateLimit:
    rate: 10r/s
    zoneSize: 10M
    key: ${http_authorization}
    rejectCode: 429
k6 驗證：

javascript
複製
編輯
import http from 'k6/http';
import { sleep } from 'k6';
export default function () {
  const headers = { 'Authorization': 'Bearer <jwt>' };
  http.get('https://jobs.local/get-job', { headers });
  sleep(0.05);
}
第 10 頁 – 可觀察性與日誌追蹤
bash
複製
編輯
kubectl describe virtualserver my-virtualserver
kubectl get policies
kubectl describe policy jwt-policy
k logs -n nginx-ingress deploy/nginx-ingress
第 11 頁 – 故障排除重點清單
DNS/Hosts：jobs.local 需正確解析至 Ingress IP

TLS：Secret 名稱需正確

路由：VirtualServer path 與服務對應正確

OpenAPI：GitHub 原始檔可達

JWT：JWK Secret 名稱一致，Token 未過期

限流：Header 對上、測試 QPS 合理

第 12 頁 – 版本控管與自動化建議
將所有 CRD/Manifests 納入 Git（GitOps）

環境參數化（dev/stage/prod）

CI 驗證 YAML 結構（kubeval、kubeconform）

CD 同步變更（Argo CD / Flux）

安全掃描與合規：加入 OpenAPI 測試與 WAF 規則校正

markdown
複製
編輯

要不要我幫你把這份 Markdown 再轉成 **投影片格式（如 Reveal.js 或者簡單的 PPTX）**，方便做簡報展示？







你說：
這不是正確的格式 然後另外請你把第幾頁 第1頁、第2頁、第3頁這些的表 已頁次的都移掉 因為這是以前的文件裡面的東西 我希望變成一個完整的操作指引即可


ChatGPT 說：

已思考 38s
明白。以下是一份「去除頁次、可直接複製貼上」的完整操作指引（Markdown 區塊）：

markdown
複製
編輯
# 在 Kubernetes 以 NGINX Plus Ingress + App Protect 建構可治理的 API Gateway

## 摘要
- **主題**：以 NGINX Plus Ingress 與 App Protect（WAF）在 Kubernetes 實作可治理的 API Gateway。  
- **能力疊加**：TLS Proxy 與 L7 路由、OpenAPI Schema 驗證（WAF）、JWT 授權、Rate Limiting。  
- **成果**：安全可控的 API 入口、CRD 宣告式治理、支援灰階/自動化。  
- **適用場景**：多租戶 API 平台、零信任接入、受監管產業審計。  

---

## 架構與元件
- **Ingress（NGINX Plus）**
  - 以 DaemonSet 方式部署。
  - 終結 TLS，執行 L7 路由/重寫。
  - 透過 `VirtualServer`、`Policy` CRD 管理。
- **App Protect（WAF）**
  - 由 OpenAPI 生成/對應 WAF Policy。
  - 阻擋不符 Schema 的 Payload，提供 Support ID 追蹤。
- **JWT 驗證**
  - RSA 簽發 Token；以 JWK Secret 驗簽。
  - 針對敏感端點（POST）強制授權。
- **Rate Limiting**
  - 以 `Authorization` Header 作為限流 Key。
  - 範例策略：10 r/s，超量 429；以 k6 驗證行為。

---

## 先決條件
- 已部署：Kubernetes（例如 microk8s 單節點）。
- 已安裝：NGINX Plus Ingress Controller 與 App Protect CRDs。
- DNS/Hosts 可解析 `jobs.local` 至 Ingress IP（或使用本機 `/etc/hosts` 映射）。
- 已安裝工具：`kubectl`（或 `microk8s kubectl`）、`jq`、`openssl`、`k6`（限流驗證用）。

---

## 環境檢查
```bash
#（microk8s 範例）檢查健康
microk8s-status.sh

# 列出 Lab 物件（加 --start-over 可重置）
list-all-k8s-lab-resources.sh --start-over

# Ingress DaemonSet
kubectl get daemonset nginx-ingress -n nginx-ingress

# 本地 Registry（如有）
curl http://localhost:32000/v2/_catalog | jq
建立 API 與前端服務（Task 01）
jobs.yaml：部署 toy API（eclectic-jobs）。

main.yaml：部署前端（myapp）。

先以 NodePort 驗證對外可用性。

bash
複製
編輯
kubectl apply -f jobs.yaml
kubectl apply -f main.yaml
kubectl get svc
# 測試
# API：  http://jobs.local:30020
# 前端： http://jobs.local:30010
啟用 TLS 與 VirtualServer（Task 02）
建立自簽憑證與 TLS Secret：

bash
複製
編輯
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout jobs.local.key -out jobs.local.crt \
  -config openssl.cnf -extensions req_ext

kubectl create secret tls jobs-local-tls \
  --key jobs.local.key --cert jobs.local.crt
VirtualServer（節選）：

yaml
複製
編輯
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: my-virtualserver
spec:
  host: jobs.local
  tls:
    secret: jobs-local-tls
  routes:
    - path: /get-job
      action: { pass: eclectic-jobs-svc }
    - path: /add-job
      action: { pass: eclectic-jobs-svc-add }
測試：瀏覽器連線 https://jobs.local/get-job（接受自簽憑證警告）。

前端整合與路徑重寫（Task 03）
yaml
複製
編輯
spec:
  routes:
    - path: /
      action: { pass: myapp-svc }
    - path: /get-job
      action: { pass: eclectic-jobs-svc }
      # 可選：重寫 /get-job → /
      # matches:
      #   - conditions: [{ variable: $request_uri, value: ^/get-job$ }]
      #     action: { pass: eclectic-jobs-svc, rewritePath: / }
驗證：前端 / 由 myapp 提供；瀏覽器端 JS 呼叫 /get-job 取得動態職稱。

OpenAPI Schema 驗證（WAF）（Task 04）
OpenAPI 片段（限制 /add-job 僅接受 string[]）：

yaml
複製
編輯
paths:
  /add-job:
    post:
      requestBody:
        content:
          application/json:
            schema:
              type: array
              items: { type: string }
App Protect 與 Policy：

yaml
複製
編輯
apiVersion: appprotect.f5.com/v1
kind: APPolicy
metadata: { name: jobs-openapi-appolicy }
spec:
  policy:
    applicationLanguage: utf-8
    openApiFiles:
      - link: https://raw.githubusercontent.com/.../jobs-openapi-spec.yaml
---
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata: { name: app-protect-policy }
spec:
  waf:
    apPolicy: jobs-openapi-appolicy
在 VirtualServer 將 app-protect-policy 套用於 /add-job 路由後，不符 Schema 的請求會被阻擋並回傳 Support ID。

JWT 授權驗證（Task 05）
產生 RSA 與簽發 JWT（示意）：

bash
複製
編輯
# 產生/簽發腳本（自行實作）
create-signed-jwt.sh
# 轉為 JWK 並存入 Secret
kubectl create secret generic jwk-secret \
  --from-file=jwk=/var/tmp/jwk/jwk.json \
  --type=nginx.org/jwk
JWT Policy：

yaml
複製
編輯
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata: { name: jwt-policy }
spec:
  jwt:
    realm: api
    secret: jwk-secret
將 JWT 與 WAF 疊加到敏感端點：

yaml
複製
編輯
spec:
  routes:
    - path: /add-job
      policies:
        - name: jwt-policy
        - name: app-protect-policy
      action: { pass: eclectic-jobs-svc-add }
成功：Authorization: Bearer <jwt>；失敗：401（缺少/無效 JWT）。

Rate Limiting 限流策略（Task 06）
Policy：

yaml
複製
編輯
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata: { name: rate-limit }
spec:
  rateLimit:
    rate: 10r/s
    zoneSize: 10M
    key: ${http_authorization}
    rejectCode: 429
套用於路由：

yaml
複製
編輯
# 例：/get-job
action: { pass: eclectic-jobs-svc }
policies:
  - name: rate-limit
k6 驗證（片段）：

javascript
複製
編輯
import http from 'k6/http';
import { sleep } from 'k6';
export default function () {
  const headers = { Authorization: 'Bearer <jwt>' };
  http.get('https://jobs.local/get-job', { headers });
  sleep(0.05); // 約 20 req/s → 受限至 ~10 r/s
}
可觀察性與日誌
bash
複製
編輯
# 物件檢視
kubectl describe virtualserver my-virtualserver
kubectl get policies
kubectl describe policy jwt-policy

# Ingress 日誌（命名空間依實際為準）
kubectl logs -n nginx-ingress deploy/nginx-ingress
WAF 事件：以 Support ID 對照被阻擋請求/回應；依情境微調 OpenAPI 與 WAF Policy。

故障排除清單
DNS/Hosts：jobs.local 正確解析至 Ingress IP。

TLS：Secret 名稱與 spec.tls.secret 一致；憑證涵蓋主機名。

路由：VirtualServer.routes[].path 與 action.pass 目標服務正確。

OpenAPI：GitHub 原始檔 URL 可讀、網路可達。

JWT：JWK Secret 名稱正確；Token 未過期；aud/iss/alg 匹配。

限流：key 與實際 Header 對上；測試 QPS 合理（避免本機快取干擾）。

版本控管與自動化
將所有 CRD/Manifests 納入 Git（GitOps）。

以環境參數化（dev/stage/prod）生成變體。

CI 驗證 YAML 結構（kubeval、kubeconform）。

CD 同步變更（Argo CD / Flux）。

安全掃描與合規：將 OpenAPI 測試與 WAF 規則校正納入 Pipeline。

延伸（前瞻性實務）
灰階/藍綠：以 VirtualServer/Policy 組合路由權重與策略差異。

零信任強化：JWT 結合 mTLS、CIDR allowlist、地理/UA/行為條件門檻。

可觀察性：接入 OTEL、集中化 WAF 事件分析與追蹤。

RASP/Schema 演進：將 OpenAPI 版控與 CI 驗證綁定 PR Gate（阻擋破壞性更動）
