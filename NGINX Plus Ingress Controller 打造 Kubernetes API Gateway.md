https://clouddocs.f5.com/training/community/nginx/html/class11/class11.html
第 1 ⾴ – 主題與全貌

主題：在 Kubernetes 中以 NGINX Plus Ingress + App Protect 建構可治理的 API Gateway。

能⼒疊加

TLS Proxy 與 L7 路由

OpenAPI Schema 驗證（WAF）

JWT 授權

Rate Limiting

成果

安全可控的 API 入⼝

以 CRD 宣告式治理

⽀援灰階與⾃動化

適⽤場景

多租⼾ API 平台

零信任接入

受監管產業審計

---

第 2 ⾴ – 架構視圖與元件

Ingress（NGINX Plus）

以 DaemonSet ⽅式部署

終結 TLS，執⾏ L7 路由／重寫

透過 VirtualServer 、 Policy  CRD 管理

App Protect（WAF）

OpenAPI → WAF Policy

阻擋不符 Schema 的 Payload

⽀援 Support ID 追蹤

JWT 驗證

RSA 簽發 Token

JWK Secret 供驗簽

套⽤於敏感端點（POST）

Rate Limiting

按 Authorization Header 分流

10 r/s 範例策略，超量 429

k6 驗證限流⾏為

---

第 3 ⾴ – Lab 環境與基礎檢查

microk8s 單節點；Ingress 監聽 80/443

別名：k、kubectl、microk8s kubectl 等價

⼯具腳本置於 ~/.local/bin；⽀援 Tab 補⿑

# 健康檢查與資源觀察
microk8s-status.sh
list-all-k8s-lab-resources.sh  # 加 --start-over 可重置

# Ingress DaemonSet 檢視
k get daemonset nginx-ingress -n nginx-ingress

# Registry 察看
curl http://localhost:32000/v2/_catalog | jq

---

第 4 ⾴ – Task 01：建立 API 與前端服務

jobs.yaml → eclectic-jobs（toy API）

main.yaml → myapp（前端）

先以 NodePort 測試對外可⽤性

# 套⽤與驗證
k apply -f jobs.yaml
k apply -f main.yaml
k get svc

# 測試 URL
http://jobs.local:30020  # API
http://jobs.local:30010  # 前端（尚未打通 HTTPS API）

---

第 5 ⾴ – Task 02：TLS 與 VirtualServer 基礎

建立憑證與 Secret

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout jobs.local.key -out jobs.local.crt \
  -config openssl.cnf -extensions req_ext

kubectl create secret tls jobs-local-tls \
  --key jobs.local.key --cert jobs.local.crt

VirtualServer（節選）

\`\`\`yaml
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
\`\`\`

瀏覽器以 HTTPS 連線 https://jobs.local/get-job；接受⾃簽憑證警告。

---

第 6 ⾴ – Task 03：前端整合與路徑重寫

新增根路徑路由

\`\`\`yaml
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
\`\`\`

體驗驗證

前端 / 由 myapp 提供

JS 於瀏覽器端呼叫 /get-job 取得隨機職稱

按 F5 多次，確認動態更新

---

第 7 ⾴ – Task 04：OpenAPI Schema 驗證（WAF）

問題與⽬標

原 /add-job 接受任意 JSON（含錯誤型別）

以 OpenAPI 定義 Schema，僅允許

string[]

\`\`\`yaml
# 片段：jobs-openapi-spec.yaml
paths:
  /add-job:
    post:
      requestBody:
        content:
          application/json:
            schema:
              type: array
              items: { type: string }
\`\`\`

套⽤ App Protect

\`\`\`yaml
# appolicy（引⽤ OpenAPI）
apiVersion: appprotect.f5.com/v1
kind: APPolicy
metadata: { name: jobs-openapi-appolicy }
spec:
  policy:
    applicationLanguage: utf-8
    openApiFiles:
      - link: https://raw.githubusercontent.com/.../jobs-openapi-spec.yaml
---
# policy（WAF）
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata: { name: app-protect-policy }
spec:
  waf:
    apPolicy: jobs-openapi-appolicy
\`\`\`

將 app-protect-policy 套⽤⾄ /add-job 路由後，不符 Schema 的請求會被阻擋並回傳 Support ID。

---

第 8 ⾴ – Task 05：JWT 授權驗證

簽發與配置

\`\`\`bash
# 產⽣ RSA、簽發 JWT（腳本⽰意）
create-signed-jwt.sh

# 轉為 JWK 並存入 Secret
kubectl create secret generic jwk-secret   --from-file=jwk=/var/tmp/jwk/jwk.json   --type=nginx.org/jwk
\`\`\`

JWT Policy（節選）

\`\`\`yaml
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata: { name: jwt-policy }
spec:
  jwt:
    realm: api
    secret: jwk-secret
\`\`\`

VirtualServer 套⽤

\`\`\`yaml
spec:
  routes:
    - path: /add-job
      policies:
        - name: jwt-policy
        - name: app-protect-policy
      action: { pass: eclectic-jobs-svc-add }
\`\`\`

成功：Authorization: Bearer <jwt>

失敗：缺少或無效 JWT → 401

---

第 9 ⾴ – Task 06：Rate Limiting 限流策略

策略定義

\`\`\`yaml
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata: { name: rate-limit }
spec:
  rateLimit:
    rate: 10r/s
    zoneSize: 10M
    key: ${http_authorization}
    rejectCode: 429
\`\`\`

VirtualServer 路由套⽤

\`\`\`yaml
action: { pass: eclectic-jobs-svc }
policies:
  - name: rate-limit
\`\`\`

k6 驗證（節選）

\`\`\`javascript
// k6-jobs.js（片段）
import http from 'k6/http';
import { sleep } from 'k6';
export default function () {
  const headers = { 'Authorization': 'Bearer <jwt>' };
  http.get('https://jobs.local/get-job', { headers });
  sleep(0.05); // 控制 QPS
}
\`\`\`

觀察 http_reqs 達到約 10 r/s

超量返回 429，客⼾端降速

---

第 10 ⾴ – 可觀察性與⽇誌追蹤

K8s 與 Ingress

\`\`\`bash
kubectl describe virtualserver my-virtualserver
kubectl get policies
kubectl describe policy jwt-policy

# Ingress Pod ⽇誌（命名空間⽰例）
k logs -n nginx-ingress deploy/nginx-ingress
\`\`\`

WAF 事件（App Protect）

查詢被阻擋請求的 Support ID

比對對應的請求／回應

依情境微調 OpenAPI 或 WAF Policy

---

第 11 ⾴ – 故障排除重點清單

- DNS/Hosts：jobs.local 需正確解析⾄ Ingress IP
- TLS：Secret 名稱與 spec.tls.secret ⼀致
- 路由：VirtualServer path 與 pass ⽬標服務對應正確
- OpenAPI 連結：GitHub 原始檔 URL 可讀；網路可達
- JWT：JWK Secret key 名稱⼀致；Token 未過期；aud/iss/alg 匹配
- 限流：key 與實際 Header 對上；測試 QPS 合理

---

第 12 ⾴ – 版本控管與⾃動化建議

- 將所有 CRD/Manifests 納入 Git（GitOps）
- 以環境參數化（dev/stage/prod）⽣成對應變體
- CI 驗證 YAML 結構（kubeval、kubeconform）
- CD 同步變更（Argo CD / Flux）
- 安全掃描與合規：加入 OpenAPI 測試與 WAF 規則校正
