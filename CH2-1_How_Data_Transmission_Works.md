# 資料傳輸機制 (How Data Transmission Works)

當開發者選用相機、光達、控制器或零組件時，原廠通常會提供其支援的原生介面，主要包含**硬體介面**與**通訊協定**兩個層面，簡單來說：
- 插什麼線？ → 硬體介面 Hardware Interface
- 資料如何交換？ → 通訊協定 Communication Protocol

因此，了解兩者差異，以及背後對應的介面規格和通訊方式，將有助於開發者選擇合適的零組件，並有效率地建立穩定且具擴充性的系統架構。

【待畫圖】架構示意如下
<!-- ![介面與協定](./images/data_transmission_architecture.jpg) -->
<p align="center"><img src="./images/data_transmission_architecture.jpg" style="width:600px" ></p>

## 1. 硬體介面（Hardware Interfaces）

硬體介面是不同零組件間進行實體連接與資料傳輸的方式，提供裝置間電氣訊號與實體連接的基礎，不同介面具有不同的傳輸速度、資料頻寬、傳輸距離及抗干擾能力，在機器人開發實務中，會依據機構設計需求、傳輸資料量、資料類型等因素，選擇或要求供應商提供特定的硬體介面。

而目前硬體介面可依介接層級和使用方式不同，概分為板級介面與系統級介面兩種。前者是指系統單晶片（SoC）、載板（Carrier Board）與周邊模組間的短距離連接，如 MIPI、UART、I²C 與 SPI，通常存在於邊緣運算平台內部，大多無對外實體接頭，僅部分會透過擴充接頭對外提供；後者則是邊緣運算平台與外部裝置間的連接，如 GMSL2、Ethernet、CAN、RS-485 與 USB，通常具有對外實體接頭，以因應系統整合需求。故以下依此分類順序，整理機器人開發上常見的硬體介面項目：

| 介面名稱 | 主要傳輸資料類型 | 主要特點 | 典型對接硬體 |
| :--- | :--- | :--- | :--- |
| **MIPI**<br>*(Mobile Industry Processor Interface)* | 未壓縮影像、原始（RAW）影像 | 影像感測器（CIS）與系統單晶片直接連接，不需額外介面轉換，具高頻寬、極低延遲、低功耗等優勢；惟連接器腳位配置（Pinout）尚未被統一，實務上需確認腳位的數量和定義 | • 相機模組（Camera Module，非完整相機產品） |
| **UART**<br>*(Universal Asynchronous Receiver/Transmitter)* | ASCII 字串、二進位序列資料 | 最常見的非同步序列通訊介面，採用點對點傳輸方式，通常只需設定鮑率（Baud Rate）即可進行資料交換，具架構簡單、成本低、易於嵌入式系統整合等優勢 | • GNSS / RTK<br>• IMU<br>• 2D 光達 |
| **I²C**<br>*(Inter-Integrated Circuit)* | 控制訊號、感測器資料、暫存器讀寫 | 使用兩條訊號線（SDA、SCL）即可連接多個週邊裝置，是載板內短距離的低速通訊方式 | • IMU<br>• 電子皮膚<br>• 溫度感測器 |
| **SPI**<br>*(Serial Peripheral Interface)* | 感測器資料、暫存器讀寫、Flash 記憶體資料 | 使用四條訊號線（MOSI、MISO、SCLK、CS）進行資料傳輸，傳輸速度快，是載板內短距離的高速通訊方式 | • IMU<br>• 電子皮膚<br>• 編碼器 |
| **GMSL1 / GMSL2**<br>*(Gigabit Multimedia Serial Link)* | 高解析度影像、感測器資料、控制訊號 | ADI 高速傳輸技術，需搭配其對應晶片，就可透過同軸纜線傳輸資料並支援 PoC 供電，具長距離、高頻寬及抗干擾等優勢；其中 GMSL1 約 3 Gbps，仍常見於 2MP 等級影像的工業機器人應用，GMSL2 則有更高傳輸能力，廣泛應用於車載及高階視覺系統 | • RGB 相機（車用等級）<br>• 全景相機 |
| **Ethernet** | 影像串流、點雲、控制封包、IP 網路資料 | 目前最普遍的有線乙太網路介面，不僅可作為一般網路通訊，也能直接連接高速感測器，具高頻寬、長距離、標準化網路架構等優勢 | • 3D 光達<br>• RGB 相機<br>• 馬達 |
| **USB** | 影像、點雲、數位音訊、批次資料 | 最常見的高速介面，即插即用（Plug & Play），支援熱插拔，並可同時提供資料傳輸與裝置供電 | • 深度相機<br>• RGB 相機<br>• 語音模組（USB Audio Class，UAC；內含 MEMS 麥克風） |
| **RS-232** | ASCII 字串、控制命令、序列資料 | 傳統短距離點對點傳輸，架構簡單，但傳輸距離增加時訊號易衰減，不適合影像等大量資料傳輸 | • 2D 光達<br>• 控制器 (舊型) |
| **RS-422 / RS-485** | 控制命令、感測器資料、序列資料 | 差動訊號傳輸，具長距離、抗干擾等優勢，同樣常見於工業設備 | • 控制器<br>• 力 / 力矩感測器<br>• 末端執行器 |
| **GPIO**<br>*(General-Purpose Input/Output)* | 數位狀態、控制訊號 | 通用型數位訊號介面，可用來讀取裝置狀態或控制外部裝置，具簡單、彈性高等特點 | • 編碼器<br>• 按鈕/開關 |
| **CAN / CAN FD**<br>*(Controller Area Network)* | 控制命令、感測器資料、裝置狀態 | 差動訊號傳輸，可讓多個裝置共用同一條匯流排（CAN Bus），具抗干擾及可靠度高等優勢；CAN FD 則為 CAN 的擴充版本，將每個資料框的資料長度由最高 8 Bytes 提升至 64 Bytes，並支援更高的資料階段傳輸速率 | • 馬達<br>• 控制器<br>• 編碼器 |

若尚不熟悉機器人開發上會使用到的硬體介面有哪些，從市面上針對機器人應用設計的專用型 Robot Controller 著手，或許是個不錯的開始，此類產品通常已整合開發與系統建置過程中的主要介接需求，因此本文透過盤點市售產品的介面配置（I/O Layout），以此歸納出目前業界較為共通的硬體介面配置，供開發者作為初期規格規劃與運算硬體選型時的參考。

【待畫圖】會畫出每個插槽的樣子

- **GMSL2**：8 channels（via 2 ports）→ 用於接多相機以確保360度環境感知，但並非標配，若無提供可依相機規格改採用 USB 或 MIPI
- **Ethernet**：1 - 5 ports → 依傳輸頻寬與應用需求，可概分為一般網路、高速網路及 EtherCAT 用途
  - 一般網路：1GbE（或 2.5GbE），用於接光達等高資料量感測器及其他網路裝置
  - 高速網路：例如 QSFP28，可做到大量資料交換與高速網路傳輸
  - EtherCAT 用途：100 Mbps 傳輸速率，因應像是馬達的低延遲、高同步之控制需求
- **USB**：2 - 4 ports → 常用 USB 3.2 Gen 1 、USB 3.2 Gen 2 或 USB2.0 版本，並採用 Type-C 或 Type-A 接頭標準
- **Serial / COM**：4 channels (via 1 port) → 有單一 (4 x RS485)或混合 (2 x RS232 + 2 x RS-232/485/422)配置
- **GPIO**：1 port → 內含 7 – 8 × GPI 與 7 – 8 × GPO
- **CAN / CAN FD**：2 - 6 channels（via 1 or 2 ports） → 輪型基本用 2 channels，人型與狗型因有更多關節馬達控制需求，會來到 4 - 6 channels，當需求更高時則搭配使用 EtherCAT 
- **Audio**：1 x 3.5mm Jack → Mic-in 麥克風輸入與 Line-out 音訊輸出
- **HDMI**：1 port → 顯示輸出

> **註**：上述盤點僅聚焦於 I/O 介面，不納入 PCIe、M.2 等擴充介面（Expansion Interface）

## 2. 通訊協定（Communication Protocol）

通訊協定是不同軟硬體間交換資料所應共同遵循的規範，建立於硬體介面之上並定義封包結構，通常包含資料格式、傳輸流程、錯誤處理及連線方式等；而在機器人開發實務中，會依零組件類型與應用情境會採用不同的協定，以下從開發者視角，依應用需求整理常用通訊協定：

- **視覺與影像裝置（Vision & Imaging Devices）**：當機器人需要管理外部網路攝影機（IP Camera）並取得即時影像，或是進行影音串流傳輸時，就會使用這類通訊協定。
- **雲端與系統整合（Cloud & System Integration）**：當機器人要連接雲端平台、車隊管理系統（Fleet Management）或 Web 系統時，會需要透過這類通訊協定交換設備狀態、任務資訊及系統資料。
- **網路傳輸（Network Communication）**：涵蓋網路層與傳輸層的基礎通訊協定，負責裝置間的資料傳輸、定址及連線管理，許多應用層通訊協定（如 MQTT、REST API、RTSP 等）都是建立在此之上。
- **定位裝置（Positioning Devices）**：當機器人在戶外環境，需要透過 GNSS 接收器取得衛星定位資訊時，通常會使用這類通訊協定，主要傳輸位置、時間、速度及航向等資料。
- **控制器或工業設備（Controller & Industrial Devices）**：機器人本體內的控制器、馬達驅動器等零組件需要交換控制命令與設備狀態時，皆使用此類通訊協定；此外，若機器人要與外部工業設備進行資料交換，例如與產線輸送帶連動，也是使用這類通訊協定。
- **車用系統（Automotive Systems）**：主要應用於智慧車輛與自駕系統，若機器人在設計上採用車載電子架構，例如車廠開發高階人型機器人時，沿用既有的車用技術與系統架構，便可能使用這類通訊協定，讓各個控制單元能交換服務與資料。

| 協定名稱 | 應用分類標籤 | 通訊方式 | 主要特點 | 典型應用場景 |
| :--- | :--- | :--- | :--- | :--- |
| **ONVIF**<br>*(Open Network Video Interface Forum)* | `Vision & Imaging Devices` | HTTP / IP Network <br> (HTTP / SOAP Web Services) | 提供不同品牌網路攝影機的互通標準，著重設備搜尋、設定、控制、影音串流與事件管理 | • 串接網路攝影機 |
| **RTSP**<br>*(Real-Time Streaming Protocol)* | `Vision & Imaging Devices` | IP Network <br> (RTSP Control / RTP Streaming) | 建立、控制與管理即時影音串流，可控制播放、暫停及停止等操作 | • 即時取得網路攝影機的影音 |
| **RTMP**<br>*(Real-Time Messaging Protocol)* | `Vision & Imaging Devices`<br>`Cloud & System Integration` | TCP/IP <br> (Streaming / Push) | 用於將影音推送至直播平台或雲端伺服器，提供持續且即時的影音串流 | • 將機器人影音即時推送至雲端 |
| **MQTT**<br>*(Message Queuing Telemetry Transport)* | `Cloud & System Integration`<br>`Network Communication` | TCP/IP <br>（Publish / Subscribe via Broker） | 輕量級訊息傳輸協定，透過 Broker 進行訊息交換，使裝置間無需建立點對點的連線，具有封包小、頻寬需求低等特性 | • 機器人與雲端平台資料交換<br>• IoT 感測器資料蒐集 |
| **REST API**<br>*(Representational State Transfer API)* | `Cloud & System Integration`<br>`Network Communication` | HTTP / HTTPS <br> (Request / Response) | 基於 HTTP/HTTPS 的 Web API 設計風格，可存取與管理系統資源，支援度高且易於跨平台系統整合 | • 機器人與雲端服務整合<br>• Web 系統資料交換 |
| **TCP/IP**<br>*(Transmission Control Protocol / Internet Protocol)* | `Network Communication` | TCP over IP <br> (Connection-Oriented / Reliable Transport) | IP 負責封包定址與路由，TCP 提供可靠的資料傳輸機制，包括封包重傳、流量控制及錯誤檢查 | • 機器人需要雲端服務或與網路設備進行通訊 |
| **UDP**<br>*(User Datagram Protocol)* | `Network Communication` | IP <br> (Connectionless / Datagram) | 提供無連線、低延遲的資料傳輸方式，不保證封包送達或傳輸順序，因此傳輸效率高 | • 即時性要求高且允許少量丟包的應用，如 3D 光達的點雲資料傳輸 |
| **NMEA 0183**<br>*(National Marine Electronics Association 0183)* | `Positioning Devices` | Serial Communication <br> (Talker / Listener) | GNSS 與導航設備的資料交換標準，以 ASCII 文字串列傳輸經緯度、速度、航向、時間等定位資訊 | • 讀取 GNSS / RTK 定位資訊 |
| **EtherCAT**<br>*(Ethernet for Control Automation Technology)* | `Controller & Industrial Devices` | Ethernet <br> (Master / Slave) | 建立於 Ethernet 的工業乙太網路協定，著重即時控制通訊與高精度時間同步，具低延遲、高同步的優勢 | • 機器人本體內多軸馬達控制 |
| **Modbus TCP** | `Controller & Industrial Devices` | TCP/IP <br> (Client / Server) | 建立於 Ethernet 的工業通訊協定，可讀寫裝置的暫存器資料 | • 機器人與工業設備資料交換 |
| **CANopen** | `Controller & Industrial Devices` | CAN Bus <br> (Producer / Consumer) | 建立於 CAN Bus 的高層通訊協定，沿用 CAN 的訊息仲裁、錯誤偵測、自動重傳等機制，加上標準化的物件字典（Object Dictionary）與通訊物件，實現裝置參數設定、即時資料交換與網路管理 | • 機器人本體內控制器與馬達驅動器通訊<br>• 廣泛應用於工業設備的即時通訊 |
| **SOME/IP**<br>*(Scalable service-Oriented Middleware over IP)* | `Automotive Systems` | UDP / TCP over IP <br> (Service-Oriented / RPC) | 建立於 Ethernet 的車載服務導向通訊協定，以 Service 為中心進行通訊，並支援服務搜尋、遠端程序呼叫及事件通知 | • 採用車載電子架構<br>• 車用電子間通訊 |