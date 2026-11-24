# ClaudIA 5GC 與 free5GC PCF UE Policy Control 實作差異分析

原分析日期：2026-11-23（Asia/Taipei）  
設計統一更新：2026-11-24（Asia/Taipei）；本次僅更新文件並核對 R17 規範，未重跑第 7 節歷史測試。

範圍：唯讀程式碼分析、公開 git 歷史查核與既有選定測試。本次未修改受分析專案、建立 commit 或 push；本文件為分析結果的 Markdown 匯出。

## 1. 結論

**free5GC 目前只有 UE Policy Control 的路由與介面基礎，四個 Handler 都直接回覆 HTTP 501；ClaudIA 則已有 URSP Encoder、設定來源及 Association Create/Delete，但仍有標準相容性與測試缺口。**

**統一目標設計：非漫遊 Create response 不攜帶供 AMF 下發的 URSP；PCF 使用 Namf Communication，AMF 透明轉送，PCF 負責 Policy Transaction、Complete／Reject、T3501 與重送。** 第 5.1 節與 AMF 報告使用相同契約；第 8 節已據此修正最小方案，不能沿用 ClaudIA 的 Create response 下發路徑。

針對 ClaudIA AMF config 提到的 length-field 問題，**不能認定「已有修復」**：公開歷史中的 Encoder 與測試完全未變；不過現行兩個長度算法與 Wireshark 解碼邏輯一致，本次未重現註解所稱的錯誤。實機 OTA 根因仍未確認。

下文「已確認」指已有程式碼、歷史或測試證據；「尚未確認」不代表已證明功能不存在。對 free5GC 的「尚未實作」判定以實際 Handler 與呼叫流程為依據，不以 API 宣告推定功能已完成。

## 2. 分析版本與規範

所有程式碼連結均固定至下列 commit，行號以該版本為準。

| 代號 | Repo | Commit |
|---|---|---|
| C | `francurieses/claudia-5gc` | `447c6e4e7ab7c11ccd36881f988ef9f19cc06727` |
| F | `free5gc/pcf` | `e953c7a7f5f46d9c51999c9f59a5f77867729d25` |
| O | `free5gc/openapi`，採 F 的實際依賴版本 | `7fad8e868cb23863719dea4b43db36ef54655454` |

規範採用：

- [TS 29.525 V17.9.0 — UE Policy Control Service](https://www.etsi.org/deliver/etsi_ts/129500_129599/129525/17.09.00_60/ts_129525v170900p.pdf)：與 free5GC 使用的 UE Policy Models 來源版本一致。
- [TS 24.526 V17.8.0 — UE policies for 5GS](https://www.etsi.org/deliver/etsi_ts/124500_124599/124526/17.08.00_60/ts_124526v170800p.pdf)。
- [TS 24.501 V17.15.0 — NAS protocol，Annex D](https://www.etsi.org/deliver/etsi_ts/124500_124599/124501/17.15.00_60/ts_124501v171500p.pdf)。
- [TS 23.502 V17.11.0](https://www.etsi.org/deliver/etsi_ts/123500_123599/123502/17.11.00_60/ts_123502v171100p.pdf)：§4.2.4.3、§4.16.11–13，透明下發及 Association 生命週期。
- [TS 29.518 V17.9.0](https://www.etsi.org/deliver/etsi_ts/129500_129599/129518/17.09.00_60/ts_129518v170900p.pdf)：§5.2.2.3、§6.1，Namf SBI 服務與資料格式。

## 3. 實作差異表

| 功能 | ClaudIA | free5GC | 程式證據：檔案、函式／型別、行號 |
|---|---|---|---|
| HTTP 路由 | 已註冊 Create、Delete | 已註冊 Create、Read、Update、Delete | C [`server.go:262–265`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/server/server.go#L262)；F [`getUePolicyRoutes():18–44`](https://github.com/free5gc/pcf/blob/e953c7a7f5f46d9c51999c9f59a5f77867729d25/internal/sbi/api_uepolicy.go#L18) |
| 路由授權 | 本表不評估完整授權能力 | **已有基礎**：UE Policy 路由掛上服務名稱對應的授權檢查 | F [`Server` 路由初始化，`server.go:120–126`](https://github.com/free5gc/pcf/blob/e953c7a7f5f46d9c51999c9f59a5f77867729d25/internal/sbi/server.go#L120) |
| Association Create | **已實作基本流程**：讀規則、編碼、配置 UUID、儲存、回傳 201／Location／`uePolicy` | **尚未實作**：直接 501 | C [`handleCreateUEPolicy():38–134`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/server/n15.go#L38)；F [`HTTPCreateIndividualUEPolicyAssociation():59–60`](https://github.com/free5gc/pcf/blob/e953c7a7f5f46d9c51999c9f59a5f77867729d25/internal/sbi/api_uepolicy.go#L59) |
| Association Delete | **已實作**：存在回 204；不存在或類型不符回 404 | **尚未實作**：直接 501 | C [`handleDeleteUEPolicy():143–168`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/server/n15.go#L143)；F [`HTTPDeleteIndividualUEPolicyAssociation():47–48`](https://github.com/free5gc/pcf/blob/e953c7a7f5f46d9c51999c9f59a5f77867729d25/internal/sbi/api_uepolicy.go#L47) |
| Read／Update | 此 UE Policy 路由區塊未提供 | **尚未實作**：兩者直接 501 | C `server.go:262–265`（同上）；F [`HTTPReadIndividualUEPolicyAssociation()`、`HTTPReportObservedEventTriggersForIndividualUEPolicyAssociation():51–56`](https://github.com/free5gc/pcf/blob/e953c7a7f5f46d9c51999c9f59a5f77867729d25/internal/sbi/api_uepolicy.go#L51) |
| Processor 業務流程 | 業務邏輯直接放於 Server Handler | **沒有接上的 UE Policy Processor 流程**：四個 Handler 均未呼叫 Processor | F [`api_uepolicy.go:47–60`](https://github.com/free5gc/pcf/blob/e953c7a7f5f46d9c51999c9f59a5f77867729d25/internal/sbi/api_uepolicy.go#L47)；[`Processor`／`NewProcessor():13–22`](https://github.com/free5gc/pcf/blob/e953c7a7f5f46d9c51999c9f59a5f77867729d25/internal/sbi/processor/processor.go#L13) |
| Association Context | 共用 policies map，以記錄類型區分 UE／SM Policy，另存 feature negotiation 結果 | **沒有專用 UE Policy Association 儲存**；現有 `AMPolicyData` 是 AM Policy | C [`handleCreateUEPolicy():99–107`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/server/n15.go#L99)；F [`UeContext:17–45`](https://github.com/free5gc/pcf/blob/e953c7a7f5f46d9c51999c9f59a5f77867729d25/internal/context/ue.go#L17)、[`PCFContext:24–49`](https://github.com/free5gc/pcf/blob/e953c7a7f5f46d9c51999c9f59a5f77867729d25/internal/context/context.go#L24) |
| URSP 規則結構 | 自訂 `URSPRule`、`TrafficDescriptor`、`RouteSelectionDescriptor` | PCF 未見對應可供 Encoder 使用的規則結構 | C [`shared/types/policy.go:10–55`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/shared/types/policy.go#L10)；F Create Handler 仍止於 501 |
| 固定 YAML | 已有 `default_ursp.rules`，轉換成內部規則 | **尚未實作**：`Configuration` 沒有 URSP 欄位 | C [`URSPRuleConfig:35–56`、`Config:89–91`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/config/config.go#L35)、[`defaultURSPRules():193–223`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/server/n15.go#L193)；F [`Configuration:72–91`](https://github.com/free5gc/pcf/blob/e953c7a7f5f46d9c51999c9f59a5f77867729d25/pkg/factory/config.go#L72) |
| URSP Encoder | **已實作部分元件**及完整 MANAGE UE POLICY COMMAND 外框 | **尚未接入／實作 PCF URSP 編碼流程** | C [`EncodeURSPRules():97–132`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/policy/ursp.go#L97)；F Create Handler 無編碼流程 |
| UDR 規則來源 | 有 UDR 查詢；無規則或查詢失敗時使用設定預設值 | PCF 未見 UE Policy Set 查詢接入業務流程 | C [`resolveURSPRules():174–189`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/server/n15.go#L174)、[`GetPolicySubscription():58–89`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/server/udr_client.go#L58) |
| OpenAPI Models／Client | 使用自行定義的 request struct／response map | **已有介面資料模型與 Client**，但不構成 Server 功能完成 | O [`PolicyAssociation:17–45`](https://github.com/free5gc/openapi/blob/7fad8e868cb23863719dea4b43db36ef54655454/models/model_pcf_ue_pol_ctrl_policy_association.go#L17)、[`CreateIndividualUEPolicyAssociation():62–155`](https://github.com/free5gc/openapi/blob/7fad8e868cb23863719dea4b43db36ef54655454/pcf/UEPolCtrl/api_ue_policy_associations_collection.go#L62) |

free5GC 可歸類為：**路由、授權掛接及 Models／Client 已存在；整體服務只有介面基礎，Association 業務與 URSP 產生尚未完成。**

## 4. ClaudIA URSP 資料結構與 Encoder

已確認的規則結構位於 C [`shared/types/policy.go:10–55`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/shared/types/policy.go#L10)：

| 型別 | 欄位摘要 |
|---|---|
| `URSPRule` | precedence、traffic descriptor、route selection descriptor list |
| `TrafficDescriptor` | match-all、DNNs、FQDNs、IPv4、protocol IDs、port ranges、connection capabilities |
| `RouteSelectionDescriptor` | precedence、SSC mode、S-NSSAI、DNN、PDU session type |
| `SNSSAI` | SST、可選 SD |
| `PolicySubscription` | ID、SUPI、precedence、規則集合 |

Encoder 支援的是一組 URSP 子集：

- `encodeTrafficDescriptor()` 實際處理 match-all、FQDN、IPv4、protocol、port range、connection capabilities。
- `encodeRSDContents()` 實際處理 SSC mode、S-NSSAI、DNN、PDU session type。
- 宣告的 IPv6、single remote port、preferred access 常數不能算作已完成能力。

證據：C [`ursp.go:71–88`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/policy/ursp.go#L71)、[`encodeTrafficDescriptor():166–210`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/policy/ursp.go#L166)、[`encodeRSDContents():232–257`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/policy/ursp.go#L232)。

固定設定範例位於 C [`nf/pcf/config/dev.yaml:46–61`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/config/dev.yaml#L46)。設定與共享型別的能力亦不完全相同：YAML 的 `URSPRuleConfig` 沒有共享 TD 型別中的 port ranges、connection capabilities 欄位；因此共享型別能表示，不代表能透過該 YAML 設定。

## 5. Release 17 標準對照與限制

| 項目 | 已確認的差異／限制 | 依據 |
|---|---|---|
| Create request | ClaudIA 將 `servingPlmn` 定義成字串；標準是 `PlmnIdNid` 物件。標準格式的物件無法直接解入該字串欄位。另使用 `servingAmfId`，而標準欄位為 `servingNfId` | C [`handleCreateUEPolicy():44–49`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/server/n15.go#L44)；O [`PolicyAssociationRequest:67–86`](https://github.com/free5gc/openapi/blob/7fad8e868cb23863719dea4b43db36ef54655454/models/model_pcf_ue_pol_ctrl_policy_association_request.go#L67)；TS 29.525 §5.6.2.3 |
| 必填欄位 | ClaudIA 只明確要求 SUPI，未要求 `notificationUri`，並允許省略 `suppFeat`。Release 17 的 request 要求這三項；response 的 `suppFeat` 亦為必填 | C [`handleCreateUEPolicy():44–77`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/server/n15.go#L44)、[`112–119`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/server/n15.go#L112)；TS 29.525 §4.2.2.2、§5.6.2.2–3 |
| Response 格式與適用角色 | Location 識別 Association 的方向正確，但 **對 AMF 回傳 `uePolicy` 作下發來源不符合標準分工**。`uePolicy` 是 H-PCF → V-PCF 的欄位；其 string／base64 表示不能證明使用情境合規。非漫遊 PCF → AMF 改走 Namf binary N1 part | C [`handleCreateUEPolicy():109–133`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/server/n15.go#L109)；O [`PolicyAssociation:20–21`](https://github.com/free5gc/openapi/blob/7fad8e868cb23863719dea4b43db36ef54655454/models/model_pcf_ue_pol_ctrl_policy_association.go#L20)；TS 29.525 §4.2.2.1、§4.2.2.2.1.0 |
| DNN Traffic Descriptor | ClaudIA 的註解稱 DNN 不是 TD 元件，並將 DNN-only／空 TD 退成 match-all。**這不符合標準的完整能力**：TD 有 DNN type `0x88`；靜默擴大為 match-all 亦改變規則語意 | C [`encodeTrafficDescriptor():161–169`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/policy/ursp.go#L161)、[`206–209`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/policy/ursp.go#L206)；TS 24.526 §5.2、Table 5.2.1 |
| RSD 規範章節 | ClaudIA 多處引用 §5.3 作為 RSD 編碼依據；本次核對的 Release 17 版本中，RSD 在 **§5.2**，§5.3 是 ANDSP | C [`ursp.go:34–43`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/policy/ursp.go#L34)、[`82–88`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/policy/ursp.go#L82)；TS 24.526 §5.2 |
| UDR 資料格式 | ClaudIA 在標準形狀的 URL 下解碼自訂 `PolicySubscription.rules`；不能直接當成 free5GC `UePolicySet` 的通用實作 | C [`GetPolicySubscription():58–89`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/server/udr_client.go#L58)；O [`Udr_DR_UePolicySet:16–31`](https://github.com/free5gc/openapi/blob/7fad8e868cb23863719dea4b43db36ef54655454/models/model_udr_dr_ue_policy_set.go#L16) |
| 完整 delivery lifecycle | ClaudIA 固定 PTI `0x80`、UPSC `1`；AMF 收到 COMPLETE／REJECT 主要是記錄 log。這不足以證明具備 PCF 交易關聯、T3501、重送及結果處理 | C [`ursp.go:60–65`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/policy/ursp.go#L60)；[`handleUEPolicyDelivery():3216–3256`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/amf/internal/nas/nas.go#L3216)；TS 24.501 Annex D.1.2、D.2.1 |

以上標準結論請對照第 2 節所列固定版本。OpenAPI Go 欄位上的 `omitempty` 不應被當成標準必填欄位的驗證機制；實作 Handler／Processor 時仍需依標準做明確驗證。

### 5.1 PCF／AMF 共用的 R17 Delivery 契約

<!-- BEGIN SHARED R17 UE POLICY DELIVERY CONTRACT -->
**共同設計基準（2026-11-24）：非漫遊 PLMN、正常註冊 UE、3GPP access。** 此範圍是首版實作與驗收限制；不代表已完成漫遊、SNPN、雙 access 或全部 Release 17 功能。下表是兩份報告共用的目標契約，並非現有程式已實作的宣告。

| 階段 | PCF 責任 | AMF 責任／介面約束 |
|---|---|---|
| Association | 建立獨立 UE Policy context；Create 回 `201 Created`、`Location`、`PolicyAssociation.suppFeat`，及適用的其他欄位 | 發起 `Npcf_UEPolicyControl_Create`；即使沒有待下發政策仍保存 Association URI；與 AM Policy 分開 |
| Create response | **對 AMF 不放入供其下發的 `uePolicy`／URSP**；該 response 欄位用於 H-PCF → V-PCF | 不從 Create response 取容器排入 NAS；不新增此路徑的 fallback |
| 回覆訂閱 | 呼叫 `Namf_Communication_N1N2MessageSubscribe`，訂閱 `UPDP` 並提供 PCF 的 N1 notification URI；保存訂閱資源 | 建立每個 UE Policy association 對應的訂閱，後續依訂閱轉送 UE 回覆 |
| 政策下發 | 產生 URSP 與 Annex D Command，經 `Namf_Communication_N1N2MessageTransfer` 下發 | 將 UPDP binary 原樣裝入受 NAS 安全保護的 `DL NAS TRANSPORT`，payload container type 為 UE policy；處理可達性、paging 與傳輸失敗 |
| 政策回覆 | 接收 `Namf_Communication_N1MessageNotify`，解碼 Complete／Reject，關聯 PTI 及各 UPSI 結果 | 從受保護的 `UL NAS TRANSPORT` 取出 UPDP，原樣以 multipart Notify 回 PCF；不終結 UE Policy 協定 |
| Policy Transaction | 管理 PTI、命令快照、T3501、重送次數、結果與過期交易；政策更新沿用 Association | 不配置政策 PTI、不跑 T3501、不決定政策重送；重新傳送時以當時 NAS context 產生受保護外框 |
| 生命週期 | Delete／termination 時停止交易與 timer，清理 Namf 訂閱及延遲工作 | 依最後 access、abort、移交等情境決定 Delete／Update，避免舊 callback 誤配新 context；Association Delete 不代表 UE 已刪除 URSP |

介面分工依 [TS 29.525 V17.9.0 §4.2.2.1、§4.2.2.2.1.0、§4.2.3.1、§5.6.2](https://www.etsi.org/deliver/etsi_ts/129500_129599/129525/17.09.00_60/ts_129525v170900p.pdf) 與 [TS 23.502 V17.11.0 §4.2.4.3、§4.16.11–13](https://www.etsi.org/deliver/etsi_ts/123500_123599/123502/17.11.00_60/ts_123502v171100p.pdf)。非漫遊 Complete／Reject 走 Namf Notify；**不由 AMF 改裝成 `uePolDelResult` 呼叫 Npcf Update**。後者是漫遊 V-PCF → H-PCF 的結果轉報；不應與 AMF 合法的 Association Update／事件回報混為一談。

**三種 callback 必須分開。** AMF 在 Create request 提供的 `notificationUri` 接收 Npcf 的 association 更新／終止通知；PCF 在 Namf Subscribe 提供的 N1 notification URI 接收 UE UPDP；PCF 在 Transfer 提供的 `n1n2FailureTxfNotifURI` 接收傳輸失敗。URI 可共用主機，但須各自有正確的路由、資料模型與關聯。Namf 使用 multipart JSON metadata＋binary N1 part，以 Content-ID 關聯，不把 Create response 的 base64 字串當成 NAS payload。[TS 29.518 V17.9.0 §5.2.2.3.1–5、§6.1](https://www.etsi.org/deliver/etsi_ts/129500_129599/129518/17.09.00_60/ts_129518v170900p.pdf)

**傳輸與政策結果分開記錄。** Create 的 201、Transfer 的 200／202、Notify 的 204 都不是 UE 套用政策的證明。PCF 須關聯 Transfer 的 Location／`n1n2MsgDataUri` 與政策交易；AMF 的同步錯誤或非同步 failure notification 交由 PCF 決定後續工作。Registration 尚在進行而暫拒 Transfer 時，須有可恢復的排程；不能在 AMF 另建政策重送器。此為依標準分工推導的實作要求。[TS 29.518 V17.9.0 §5.2.2.3](https://www.etsi.org/deliver/etsi_ts/129500_129599/129518/17.09.00_60/ts_129518v170900p.pdf)

**PCF 交易規則與驗收：**

- 發出 Command 時依 Annex D 配置可用 PTI，先建立可接收快速回覆的交易狀態，並啟動該 PTI 的 T3501；同一交易重送保留 PTI 與命令內容，不重新 Create。初版限制每 UE 一筆進行中交易是本地設計選擇，須有佇列／合併決策，不能覆寫既有交易。
- Complete 停止 T3501；Reject 亦停止 T3501，依 management result 中各 UPSI／cause 處理，未列為失敗的指令視為成功。不能把 Reject 一律當成整包失敗，或直接讀第三個 byte 作 cause。
- T3501 第 1–4 次到期重送並重啟 timer；第 5 次到期中止並釋放 PTI。傳輸失敗 response／notification 必須停止受影響交易的監督 timer；恢復可達性後的補送與 T3501 到期重送分開處理。
- Complete／Reject 後不立即重用 PTI；保留至少一個預設 T3501 期間是規範列出的實作方式。未知／保留 PTI 依 Annex D 忽略，重複／遲到回覆不得完成另一筆交易。
- PCF 與 UE 的政策處理成功仍須與實際依 URSP 選擇 PDU Session 分層驗證；AMF 不保存「URSP 已套用版本」作為成功判準。

以上依 [TS 24.501 V17.15.0 Annex D.1.2、D.2.1.2–5、D.5–D.6、D.8.3](https://www.etsi.org/deliver/etsi_ts/124500_124599/124501/17.15.00_60/ts_124501v171500p.pdf)。URSP policy part 依 TS 24.526 §5.2；NAS transport 依 TS 24.501 §5.4.5。

**時序限制：** Subscribe 必須在可能收到 UE 回覆前就緒；PCF／AMF 須能承受 Create response 前後的反向 SBI 呼叫，不持有 UE lock 等待 callback。圖示順序不構成規範要求的全序；Registration Complete 也不是所有 registration update 的唯一觸發點。可先驗證非漫遊閉環，漫遊另設 H-PCF／V-PCF 及 PTI mapping 測試，不沿用非漫遊假設。
<!-- END SHARED R17 UE POLICY DELIVERY CONTRACT -->

對應 AMF 整合及標準時序圖見[AMF 分析第 3、8–10 節](claudia-free5gc-amf-ue-policy-analysis.md)。

## 6. AMF config 的 length-field 問題

### 6.1 設定與公開歷史

AMF 設定仍寫明 Encoder 第 109、115 行有兩項錯誤，且 `ursp_enabled: false`。這是已確認的設定狀態；其引用的 OTA evidence 檔，在公開 repo／該路徑歷史中未找到。

證據：C [`nf/amf/config/dev.yaml:9–19`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/amf/config/dev.yaml#L9)。

| 核對項目 | 現碼／測試結果 | 判定 |
|---|---|---|
| Policy part length | `1 + len(URSP rules)`，涵蓋 type octet | 與獨立解碼邏輯一致 |
| Instruction length | `2 + len(policy parts)`，涵蓋 UPSC | 與獨立解碼邏輯一致 |
| 既有 golden vector | 全長 50 bytes；list=46、sublist=44、instruction=39、part=35、rule=32、RSD list=26、RSD=24、contents=21 | 目前向量的長度互相吻合 |
| 公開修復歷史 | Encoder、`ursp_test.go` 從首個公開 commit 起內容完全相同 | **沒有公開的 length-field 修復紀錄** |
| 實機 OTA | 未取得註解引用的封包與證據 | **尚未確認問題根因或實機已恢復正常** |

上述長度與向量來自 C [`EncodeURSPRules():107–130`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/policy/ursp.go#L107)、[`TestEncodeURSPRules_GoldenVector():357–398`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/policy/ursp_test.go#L357)。歷史對照為首版 commit [`d72a7a88135bc48f983572728e5bcc4fff2adfd1`](https://github.com/francurieses/claudia-5gc/blob/d72a7a88135bc48f983572728e5bcc4fff2adfd1/nf/pcf/internal/policy/ursp.go#L107)。

查核時，首版與 C 的 git blob IDs 相同：

| 檔案 | Git blob ID |
|---|---|
| `nf/pcf/internal/policy/ursp.go` | `b0ae6f0f3ea0bd8c5075272c706d2737042706a0` |
| `nf/pcf/internal/policy/ursp_test.go` | `ea01159ea2318e4602b2bd12fb422da343918a22` |

### 6.2 獨立交叉檢查與限制

交叉檢查使用 Wireshark commit `6c46e8cd6d8dcdf560c17773b9765acd3c891663` 的 [`de_nas_5gs_updp_ue_policy_section_mgm_lst()`，12140–12214 行](https://github.com/wireshark/wireshark/blob/6c46e8cd6d8dcdf560c17773b9765acd3c891663/epan/dissectors/packet-nas_5gs.c#L12140)：其解碼分別扣除 UPSC 的 2 bytes 與 policy type 的 1 byte。這是原始碼交叉核對，並非實際跑過 Wireshark 封包解碼。

因此，較準確的結論是：**現行兩個算法有相容的交叉證據；設定註解與現碼不一致，但不能據此宣布 OTA 問題已修復。**也不應只因註解存在，就把目前兩個加法直接刪除。

## 7. ClaudIA 測試與實際驗證

### 7.1 已閱讀的測試資產

| 測試／資產 | 覆蓋內容 | 固定版本證據 |
|---|---|---|
| `parseContainer()`、`parseURSPRules()` | 多層長度與 framing 檢查 | C [`ursp_test.go:30–147`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/policy/ursp_test.go#L30) |
| `TestEncodeURSPRules_*` | 空集合、header、PLMN、match-all、RSD 元件、SST-only、TD 元件、多規則、golden vector | C [`ursp_test.go:149–398`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/policy/ursp_test.go#L149) |
| `TestUEPolicy_CanonicalPath()` | Create、base64、Location、Delete | C [`routes_test.go:83–117`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/server/routes_test.go#L83) |
| `TestUEPolicy_DeleteUnknown()` | 未知 ID、重複刪除、SM／UE association 隔離 | C [`routes_test.go:122–151`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/server/routes_test.go#L122) |
| `TestGolden_ProducerCreates()` | 含 UE Policy Create 的 HTTP response snapshot | C [`features_golden_test.go:74–90`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/internal/server/features_golden_test.go#L74)；已閱讀，本次未用下列篩選條件執行此測試 |

### 7.2 過時的 feature 情境

`ursp.feature` 還要求 Registration Accept／Configuration Update Command 攜帶 URSP；現行 `SendUEPolicyContainer()` 已使用 DL NAS TRANSPORT、payload type `0x05`。該 feature 不能作為目前端到端能力的證據。

證據：C [`ursp.feature:13–34`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/pcf/tests/features/ursp.feature#L13)、[`SendUEPolicyContainer():3977–4015`](https://github.com/francurieses/claudia-5gc/blob/447c6e4e7ab7c11ccd36881f988ef9f19cc06727/nf/amf/internal/nas/nas.go#L3977)。

### 7.3 原分析執行結果（2026-11-23）

在 ClaudIA 的獨立下載副本執行：

```bash
GOWORK=off go test -mod=readonly \
  ./nf/pcf/internal/policy ./nf/pcf/internal/server \
  -run 'TestEncodeURSPRules_|TestUEPolicy_|TestLocationAbsoluteAllCreates' \
  -count=1
```

兩個 package 的選定測試均通過。

這不等同完整測試套件、3GPP conformance、實機 OTA 或 free5GC 端到端驗證。free5GC 的 501 判定來自直接閱讀 Handler；本次未另外對其執行 HTTP runtime probe。

## 8. 最小修改方案與建議實作順序

第一個交付里程碑可先完成「固定設定 → 編碼 → Association Create/Delete」，但這只是內部前置能力。**可驗收的最小 UE Policy Delivery 必須包含 Namf 下發、UE 回覆與 PCF 交易閉環。** 以下為目標方案，尚未實作；不以 Create response 回傳 URSP 當過渡路徑。

| 順序 | 建議修改 | 驗收條件 |
|---|---|---|
| 1 | **固定 YAML 與驗證**：於 F `pkg/factory/config.go` 的 `Configuration` 增加可選 URSP 設定；第一版只接受 match-all、規則／RSD precedence、SSC mode、S-NSSAI、DNN、PDU session type | 拒絕重複 precedence、空 RSD、不合法 SD／PLMN／DNN、超長內容及不支援元件；不靜默轉成 match-all |
| 2 | **獨立撰寫 Encoder 與測試**：新增 URSP policy part encoder 與 Annex D message codec；PTI、UPSC、PLMN 由 PCF 呼叫端提供 | 依 TS 24.526 §5.2、TS 24.501 D.5／D.6 自行推導測試向量；各層長度、兩／三位 MNC、SST-only／含 SD、多規則及錯誤輸入；不固定重用 PTI `0x80` |
| 3 | **專用 Association／Transaction Context**：獨立 store、同步與生命週期；交易保存 SUPI／association generation、PTI、UPSI 清單、命令快照、timer、計數及結果，另關聯 Namf subscription／transfer URI | 不混用 AM／SM context；同 UE 交易不覆寫，舊回覆不影響新交易；跨 UE、Create/Delete／timer 競態可控 |
| 4 | **Create／Read／Update／Delete Handler → Processor**：替換 F `api_uepolicy.go:47–60` stub；依 R17 必填及條件欄位驗證 | 非漫遊 Create：201、可查得的 Location、必填 suppFeat，**不含下發用 uePolicy**；無新政策仍建 Association。Delete：204，未知 ID：404；Update 回報事件／更新 association，不承接 AMF 改裝的 Complete／Reject |
| 5 | **Namf consumer 與 PCF callbacks**：新增 UPDP Subscribe／UnSubscribe、N1N2MessageTransfer client，以及 N1MessageNotify／failure notification endpoints | 訂閱就緒後才能進入可收回覆的下發流程；multipart JSON／binary、Content-ID 正確；保存 202 Location 並關聯失敗回報；callback 與 Create response 可競態 |
| 6 | **PCF 交易引擎**：Command、Complete／Reject decoder、PTI 關聯、T3501、重送、各 UPSI 結果及清理 | 依第 5.1 節：第 1–4 次 expiry 重送、第 5 次中止；Complete／Reject／傳輸失敗停 timer；部分成功、未知／重複／遲到 PTI、Delete 取消工作均有測試 |
| 7 | **AMF 整合與閉環驗證**：對接 AMF 報告第 8–10 節；將初始原始 uePolReq 交 PCF，重用 UPDP 雙向透明轉送 | Create → Subscribe → Transfer → protected DL NAS → UE Complete／Reject → protected UL NAS → Notify → PCF 結果 → Delete；不依賴 Create response 容器；200／202／204 不當作套用成功 |
| 8 | **恢復與擴充**：registration ongoing 暫拒、CM-IDLE／paging、timeout、重啟／移交，再接 UDR 與動態政策 | 同步／非同步傳輸失敗與交易關聯、有限恢復策略、無明文 fallback／無舊加密 NAS 重播；動態更新沿用 Association；漫遊另驗 H-PCF／V-PCF 分工 |

第 4 步可獨立驗證 Association API，但不得單獨宣稱 UE Policy Delivery 已完成。第 5–7 步是最小功能閉環的必要部分。

上述現有程式入口：

- F [`Configuration:72–91`](https://github.com/free5gc/pcf/blob/e953c7a7f5f46d9c51999c9f59a5f77867729d25/pkg/factory/config.go#L72)。
- F [`UeContext:17–45`](https://github.com/free5gc/pcf/blob/e953c7a7f5f46d9c51999c9f59a5f77867729d25/internal/context/ue.go#L17)、[`PCFContext:24–49`](https://github.com/free5gc/pcf/blob/e953c7a7f5f46d9c51999c9f59a5f77867729d25/internal/context/context.go#L24)。
- F [`UE Policy Handlers:47–60`](https://github.com/free5gc/pcf/blob/e953c7a7f5f46d9c51999c9f59a5f77867729d25/internal/sbi/api_uepolicy.go#L47)。
- O [`PolicyAssociationUpdateRequest:39–45`](https://github.com/free5gc/openapi/blob/7fad8e868cb23863719dea4b43db36ef54655454/models/model_pcf_ue_pol_ctrl_policy_association_update_request.go#L39)。`uePolReq` 可轉交適用的原始 UE protocol request；`uePolDelResult`／`uePolTransFailNotif` 按 H-PCF／V-PCF 角色使用，欄位存在不表示非漫遊 AMF 應拿它們回報政策交易結果。

第一版可用固定 UPSC 識別固定政策內容，但內容更新、UE state 與 section 管理仍由 PCF 決定；PTI 必須由交易狀態管理。

**Association Delete 的 204 只代表 PCF association 已刪除，不能當成 UE 已移除本地 URSP 的證明。** 若要刪除 UE 儲存的 section，須由 PCF 發送 Annex D 的刪除指令並處理 UE 回覆。

## 9. 不複製 GPL 程式碼的實作原則

後續實作應保留標準條文、欄位長度推導及自行建立的測試向量作為設計依據，**不移植 ClaudIA 的函式、decoder 或 golden test 程式碼**。

本次已閱讀 GPL 原始碼，因此也不應將後續工作描述為已完成正式的 clean-room 隔離。本報告只參考設計及比對行為，沒有將 ClaudIA 程式碼加入 free5GC。

## 10. 尚未確認事項

- ClaudIA AMF config 引用的 OTA evidence 原始內容及封包未取得，無法判斷當時執行的 binary／分支是否與公開程式碼一致。
- 未執行真實 UE／Pixel 6a 的 OTA 重測，不能宣布 config 所述實機問題已解決。
- 尚未完成完整 UE policy transaction、通知、T3501、重送與 UE State Indication 的跨 NF 動態驗證。
- 原 PCF 分析未涵蓋 AMF 完整路徑；本次已與 AMF 報告統一目標設計，但未新增實作或端到端測試，仍不能宣稱 free5GC 已具完整 URSP 支援。
