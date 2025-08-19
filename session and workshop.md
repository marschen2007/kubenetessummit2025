# Beyond NGINX：重新定義 Ingress Controller 的部署邊界與彈性架構

## 簡述
許多人可能會好奇，F5 與 Kubernetes 有什麼關聯？  
其實，我們不僅將 Kubernetes 作為軟體部署平台，更將其融入產品核心設計，從 **硬體設備** 到 **雲原生功能（CNF）**，再到直接部署在用戶的 **Kubernetes 集群** 中，發揮不同價值。

在這個背景下，本議程將介紹三種 F5 提供的 Ingress 架構選項：

1. **NGINX OSS 與 Plus**  
   - 支援 WAF 與 Gateway API  
   - 滿足雲原生場景的擴展與安全需求  

2. **BIG-IP（硬體或虛擬機）+ CIS 外部設備感知模式**  
   - 整合資料中心與 Kubernetes  
   - 提供跨環境的應用交付能力  

3. **容器化 BIG-IP 方案**  
   - 可運行於標準伺服器或 DPU  
   - 兼具高效能與部署彈性  

> 透過此次分享，將帶您快速比較這三種架構，並思考如何在不同場景下突破 Ingress Controller 的部署邊界。

---

# 工作坊主題  
**NGINX Plus Ingress Controller 打造 Kubernetes API Gateway 全攻略**  
🗓 10/22 下午 3:30–5:00  
📍 Kubernetes Summit

---

## 課程簡介
本工作坊將帶領參與者在 **Kubernetes 環境** 中，使用 **NGINX Plus Ingress Controller** 建立高效、安全、可擴展的 API Gateway。  

透過實作，您將學會如何以 **API 為核心**，結合：  
- TLS 加密  
- API 規範驗證  
- JWT 授權  
- 流量控制  

全面提升 API 服務的安全性與穩定性。

---

## 主要內容
### 1. 環境準備與基礎設施
- 建立 Kubernetes 實驗環境  
- 熟悉 Ingress、DaemonSet、Service 等核心概念  
- 為 API Gateway 實作做好基礎準備  

### 2. API Gateway 核心功能實作
- **TLS 代理與路由**  
  根據 Host Header 與路徑將 HTTPS 請求轉送至對應 API 服務  
- **API 架構驗證**  
  使用 OpenAPI Spec 定義 API 架構，並透過 NGINX App Protect 強制執行規範  
- **授權驗證**  
  驗證簽署的 JWT，確保 API 訪問者具備正確授權  
- **流量限制**  
  對 API 端點進行每客戶端 Session 的速率限制，確保資源公平分配  

---

## 適合對象
- Kubernetes 與雲原生應用架構工程師  
- API 開發與運維人員  
- 對 API 安全與 API Gateway 實務感興趣的 DevOps 與資安工程師  

---

## 學習成果
完成本工作坊後，您將能：
- 在 Kubernetes 上部署並配置 **NGINX Plus Ingress Controller** 作為 API Gateway  
- 實作 **TLS 加密**、**API 架構驗證**、**JWT 授權** 與 **流量限制**  
- 掌握 **API Gateway 與應用安全整合** 的最佳實務  

