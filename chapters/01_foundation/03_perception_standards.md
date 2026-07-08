# 第一章 基礎 Foundation

## 3. 感知工程規範 ROS 2 engineering standards for sensor fusion

在機器人感知系統的實務開發中，最令人頭痛的往往不是演算法本身，而是「資料格式不對齊」與「座標系混亂」。
為了確保多個感測器與後續演算法（如建圖、定位、避障）能夠順暢對接，開發團隊必須遵循一套標準的工程規範。本節將從兩個核心面向提供寫程式上的架構性參考。

---

### 3.1 面向一：遵循 ROS 提案標準 (ROS Enhancement Proposal, REP)

REP 是 ROS 社群共同制定的標準協議，旨在規範開發者該如何撰寫代碼、定義單位與座標系。在開發感知系統時，建議遵循以下開發順序，逐一對照對應的 REP 規範：

```
確認開發平台版本 (REP-2000) ──> 挑選設備並研讀規格 ──> 依應用需求撰寫配置並實作代碼 (REP-117, REP-118, REP-103, REP-105)
```

#### 3.1.1 步驟一：確認開發平台版本 (REP-2000)
*   **對應規範**：**REP-2000 (ROS 2 Releases and Target Platforms)**
*   **規範重點**：REP-2000 定義了每一個 ROS 2 版本所支援的作業系統（如 Ubuntu 22.04 LTS）、編譯器版本（GCC/Clang）、Python 版本以及依賴的 C++ 標準（如 C++ 17）。
*   **感知實務**：在動工前，必須與整機系統統一版本（例如統一使用 **ROS 2 Humble Hawksbill** 搭配 Ubuntu 22.04），以確保所有感測器廠商提供的二進位 Driver SDK 或 ROS 2 Node 能無縫編譯，避免因底層 C++ 標準庫或 Python 版本衝突導致編譯失敗。

#### 3.1.2 步驟二：處理無效觀測值 (REP-117 & REP-118)
當感測器（如光達、深度相機）探測的距離過近、過遠或發生鏡面反射時，數據會產生「無效值」。
*   **REP-117 (LaserScan Range Values)**：
    *   規範雷達在無效探測時的數值填充規則：
        *   若距離低於感測器最小量程（Too close），數值應設為 `NaN` 或小於 `range_min`。
        *   若距離超出最大量程（Out of range / Too far），數值應填充為正無限大 `+Inf`。
*   **REP-118 (Depth Images)**：
    *   規範深度相機生成的深度圖像素值（通常為 16-bit 單位毫米，或 32-bit 浮點數單位米）。
    *   無效深度（無法解析深度或噪訊）必須設為 `0` (對於 16-bit 整數) 或 `NaN` / `0.0` (對於 32-bit 浮點數)。
*   **感知實務**：在撰寫感測器 Node 時，切勿自行發明數值（例如將超出距離填為 `-1` 或 `999.0`），這會直接導致避障演算法（如 Nav2 Costmap）將其判定為障礙物，進而造成機器人異常急停。

#### 3.1.3 步驟三：統一計量單位與座標方向 (REP-103)
*   **對應規範**：**REP-103 (Standard Units of Measure and Coordinate Conventions)**
*   **規範重點**：
    *   **單位標準**：長度使用米（meters, m）、角度使用弧度（radians, rad）、時間使用秒（seconds, s）、質量使用公斤（kilograms, kg）。
    *   **座標系標準**：一律採用**右手座標系 (Right-Handed Coordinate System)**。
        *   **X 軸**：指向正前方 (Forward)
        *   **Y 軸**：指向左方 (Left)
        *   **Z 軸**：指向正上方 (Up)
        *   **旋轉方向**：依據右手大拇指指向軸向，其餘四指彎曲方向為旋轉正方向（逆時針為正）。
*   **感知實務**：IMU 輸出的角速度（radians/s）和加速度（m/s²）必須嚴格符合 REP-103。若購買的 IMU 硬體預設輸出是角度（degrees）或左手座標系，驅動 Node 必須在發布 `sensor_msgs/msg/Imu` 前在內部完成單位與方向的轉換。

#### 3.1.4 步驟四：定義階層式座標系 (REP-105)
*   **對應規範**：**REP-105 (Coordinate Frames for Mobile Platforms)**
*   **規範重點**：定義了移動機器人中幾個核心座標系（Frames）的拓撲與物理意義：
    *   `map`（地圖座標系）：全域座標系，原點通常固定在建圖起始點，不隨時間漂移，但座標更新可能是不連續的（因 SLAM 回環檢測修正）。
    *   `odom`（里程計座標系）：局部座標系，原點在機器人啟動點，其位置由里程計、IMU 等推算，隨時間累積會產生漂移，但其更新必須是平滑且連續的。
    *   `base_link`（機器人基座座標系）：固定在機器人底盤物理中心的座標系，原點通常位於旋轉中心。
    *   `sensor_frame`（感測器座標系）：各感測器（如 `laser_link`, `camera_link`）自身的座標系。
*   **感知實務**：感測器相對於 `base_link` 的相對位置（外參）必須透過靜態變換（Static TF）進行發布，確保演算法能將雷達探測到的點雲點轉換到 `base_link` 甚至 `odom` 座標系中。

---

### 3.2 面向二：從主流產品 ROS Package 的 Launch 設計回推 Node 分類

在實務上，我們可以透過觀察國際一流感測器廠商（如 Ouster, Velodyne, Intel RealSense）官方 ROS Package 的 `launch` 檔案結構，回推出標準的**節點類別設計**。
這能幫助我們在撰寫自研感測器驅動或整合系統時，規劃出結構清晰、便於串接的 Node 階層：

#### 3.2.1 節點類別 1：硬體驅動與原始資料發布節點 (Driver Node)
*   **核心職責**：透過物理通訊介面（PCIe, GMSL2, Ethernet, USB）與硬體通訊，讀取原始暫存區（Buffer）數據。
*   **輸出話題**：發布最原始、未經大量過濾的標準資料。
    *   相機發布 `sensor_msgs/msg/Image`
    *   光達發布 `sensor_msgs/msg/PointCloud2`
*   **Launch 範例回推**：通常對應 `driver_node`，此節點通常被寫成 Lifecycle Node，支援在運行中動態重啟與設定參數。

#### 3.2.2 節點類別 2：前處理與過濾節點 (Preprocessing / Filtering Node)
*   **核心職責**：將原始高畫質數據進行降採樣（Downsampling）、噪訊過濾（Filtering）、或局部裁剪，以減輕大腦的運算負擔。
*   **常見操作**：
    *   點雲：使用 Voxel Grid 濾波器降低點雲密度、使用 Passthrough 濾波器裁切掉機器人本體（遮擋區）的點雲。
    *   影像：進行畸變校正（Rectification）或縮放。
*   **Launch 範例回推**：在 Launch 檔中，通常會與 Driver Node 放在同一個 **Component Container** 中，利用 ROS 2 的 **Intra-Process Communication (IPC)** 技術，在同一個進程記憶體中傳遞指針，達成「零拷貝 (Zero-Copy)」，大幅減少 CPU 與記憶體頻寬消耗。

#### 3.2.3 節點類別 3：多感測器融合與特徵提取節點 (Processing / Fusion Node)
*   **核心職責**：執行核心演算法。
*   **常見操作**：
    *   將點雲與影像對齊，為點雲點上色（RGB-D Point Cloud）。
    *   結合 IMU 與光達點雲進行運動去畸變（Motion Undistortion）。
*   **輸出話題**：發布高級特徵或已融合的感知結果，供 SLAM 或導航模組直接訂閱。
