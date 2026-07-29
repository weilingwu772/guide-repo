# 資料傳輸機制 (How Data Transmission Works)

在機器人、智慧製造、AIoT 與自動化系統中，不同設備之間必須透過各種通訊技術完成資料交換。這些技術依用途大致可分為通訊協定（Communication Protocols）、**硬體介面（Hardware Interfaces）以及網路傳輸（Network Communication）**三個層面。通訊協定定義資料如何交換，硬體介面決定設備如何實體連接，而網路傳輸則提供資料在網路上的傳送機制。了解各項技術的設計目的、適用場景與限制，有助於開發者快速建立穩定且具擴充性的系統架構。

一、通訊協定（Communication Protocols）

通訊協定（Communication Protocol）是不同設備或軟體之間交換資料所共同遵循的規範，包含資料格式、傳輸流程、錯誤處理及連線方式等。不同應用領域會採用不同的通訊協定，以兼顧效率、可靠性與相容性。

| 協定名稱 | 傳輸架構 / 運作模式 | 主要特點 | 典型對接感測器與應用 |
| :--- | :--- | :--- | :--- |
| **MQTT**<br>*(Message Queuing Telemetry Transport)* | 發布／訂閱 (Pub/Sub) <br> (Broker 訊息中介) | 專為 IoT/AIoT 設計的輕量級協定，封包小、頻寬需求低，無需設備間直接連線 | • IoT 感測器 |
| **REST API**<br>*(Representational State Transfer)* | HTTP / HTTPS <br> (Client / Server 資源請求) | 透過 GET/POST 等方法存取資源，支援度高且易於跨平台整合 | • Web 系統整合<br>• 邊緣運算平台 (Cloud Service、AI 推論服務) |
| **RTSP**<br>*(Real-Time Streaming Protocol)* | 影音串流控制 <br> (搭配 RTP 傳輸資料) | 可控制影音播放、暫停與停止，適合低延遲視訊傳送 | • 遠端監控<br>• RGB 相機<br>• 全景相機<br>• 即時串流 |
| **RTMP**<br>*(Real-Time Messaging Protocol)* | TCP / 串流推流 | 低延遲影音推流協定，適合將影像推送至媒體伺服器或直播平台 | <br>• RGB 相機<br>• 直播、遠端展示、雲端影音推流 |
| **ONVIF**<br>*(Open Network Video Interface Forum)* | IP 網路國際標準 <br> (Web Services) | 統一不同品牌 IP Camera 控制方式（包含影像串流、PTZ 雲台與設備管理） | <br>• IP Camera |
| **Modbus TCP** | Ethernet TCP/IP <br> (Master / Slave) | 工業自動化最普遍協定，建立於 Ethernet 之上，可讀寫暫存器資料 | <br>• 控制器(PLC) |
| **CAN Bus**<br>*(Controller Area Network)* | 匯流排架構 <br> (Multi-master 即時廣播) | 多主機即時通訊，具備高可靠性、抗干擾能力強及硬體優先權仲裁 | `控制器` (MCU)、伺服馬達驅動器、移動機器人底盤 |
| **SOME/IP**<br>*(Scalable service-Oriented Middleware over IP)* | Ethernet UDP/TCP <br> (服務導向 / RPC) | 汽車產業服務導向通訊協定，支援服務發現（Service Discovery）與遠端呼叫 | 車用 ADAS、`控制器` (自駕車系統) |
| **NMEA 0183**<br>*(National Marine Electronics Association)* | UART / RS-232 <br> (ASCII 文字串列) | GPS 與導航設備資料標準，以 ASCII 字串傳輸經緯度、速度與時間 | `GNSS / RTK` 衛星導航、戶外自主移動機器人 (AMR) |

二、硬體介面（Hardware Interfaces）

硬體介面是設備之間進行實體連接的方式，不同介面具有不同的傳輸速度、距離及抗干擾能力，因此需依據感測器特性與資料量選擇適當的介面。硬體介面決定感測器與處理晶片（SoC / MCU / 工業電腦）之間的實體物理連接方式，直接影響數據傳送的頻寬與抗干擾能力。


| 介面名稱 | 傳輸速率 / 距離 | 主要特點 | 典型對接感測器與應用 |
| :--- | :--- | :--- | :--- |
| **MIPI**<br>*(Mobile Industry Processor Interface)* | 高頻寬 (Gbps 級) <br> < 30 cm | 晶片級直接連線，極低延遲，無額外轉換開銷 | • RGB 相機<br>• 深度相機 (RGB-D)<br>• 全景相機 |
| **GMSL2**<br>*(Gigabit Multimedia Serial Link 2)* | 高頻寬 (最高 6-8 Gbps) <br> 最長 15m | 透過單一同軸電纜同時傳輸影像、控制訊號及 POD 供電，具有長距離、高頻寬及抗干擾能力佳等優點 | •RGB 相機（車載/戶外遠距）<br>• 全景相機（環視視覺）<br>• 3D 光達 |
| **Ethernet** | 1 Gbps - 10 Gbps <br> > 100m | 目前最普遍的有線網路介面，不僅可作為一般網路通訊，也能直接連接各種感測器，長距離、高頻寬且具標準化網路架構，支援 PoE 供電 | • 2D / 3D 光達<br>• 邊緣運算平台 (IPC/SBC) |
| **USB 3.0 / 3.2** | 5 Gbps - 10 Gbps <br> < 3m | 目前最普遍的通用介面，開箱即用（Plug & Play），熱插拔，同時提供 5V 供電，3.0 提供最高 5 Gbps 的傳輸速度，可支援高解析相機、深度相機及高速儲存設備，是機器人開發中最常見的高速介面之一 | • 深度相機 <br>• 2D 光達<br>• RGB 相機 |
| **CAN Bus / CAN-FD** | 1 Mbps (CAN) / 8 Mbps (FD) <br> 數十至數百公尺 | 差動訊號抗干擾強、多主機（Multi-master）具硬體優先權仲裁 | • 控制器 (MCU/DSP)<br>• 馬達/驅動器 (無框力矩馬達/BLDC)<br>• 三維/六維力感測器<br>• 編碼器 |
| **UART**<br>*(Universal Asynchronous Receiver/Transmitter)* | 9600 bps - 921600 bps <br> < 2m | 最常見的序列通訊介面，邏輯電平點對點傳輸，架構極簡、成本低 | • 慣性測量單元 (IMU)<br>• GNSS / RTK<br>• 超音波 / 紅外線<br>• 控制器 (MCU) |
| **RS-232** | 最高 115.2 kbps <br> 最長 15m | 歷史悠久的串列通訊標準，適合短距離點對點傳輸 | • 一維力感測器<br>• 控制器 (舊型/調試介面) |
| **RS-485** | 最高 10 Mbps <br> 最長 1200m | 支援長距離、多節點及差動訊號傳輸，具有良好的抗干擾能力 | • 超音波 / 紅外線<br>• 電子皮膚<br>• 三維/六維力感測器<br>• 控制器 |

三、網路傳輸（Network Communication）

網路傳輸技術負責設備之間資料封包的實際傳送，是所有網路通訊協定的基礎。

TCP/IP（Transmission Control Protocol / Internet Protocol）

TCP/IP 是目前網際網路最核心的通訊架構，其中 IP 負責封包定址與路由，TCP 則提供可靠的資料傳輸機制，包含封包重傳、流量控制及錯誤檢查，因此適合需要資料完整性的應用。

適用情境： REST API、MQTT、雲端服務、檔案傳輸。

UDP（User Datagram Protocol）

UDP 提供無連線、低延遲的資料傳輸方式，不保證封包一定送達，因此傳輸效率較高，適合即時影音及機器人控制等對延遲敏感的應用。

適用情境： 即時影像、LiDAR 資料串流、ROS 2 DDS、影音串流。