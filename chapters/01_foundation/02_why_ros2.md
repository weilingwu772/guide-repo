# 第一章 基礎 Foundation

## 2. 主流開發框架 Why use ROS 2 for robot development?

機器人開發是一個極度跨領域的工程，涉及機械、電機、通訊、演算法與軟體。為了避免每開發一款機器人就必須「重複造輪子」，採用標準化的軟體開發框架已成為產業共識。

### 2.1 機器人軟體世界關係與系統架構

機器人開發世界是由多個層級與不同角色相互協作而成的系統。下圖展示了從底層感測硬體、驅動軟體、ROS 2 通訊中間件，到最上層感知與導引演算法的階層關係：

```mermaid
%% 機器人系統開發關係與架構圖
graph TD
    %% 架構層級
    subgraph Upper["演算法與應用層 (High-level Applications)"]
        SLAM["SLAM 建圖與定位 (e.g., FAST-LIO, Cartographer)"]
        Nav["路徑規劃與自主避障 (Nav2)"]
    end

    subgraph Middleware["通訊與核心框架 (Middleware)"]
        ROS2["ROS 2 節點通訊 / DDS 傳輸"]
        SensorFusion["多感測器融合 (Sensor Fusion) 模組"]
    end

    subgraph Driver["硬體抽象與驅動層 (Drivers)"]
        LidarDriver["光達 ROS 2 Driver Node"]
        CamDriver["深度相機 ROS 2 Driver Node (realsense_ros)"]
        IMUDriver["IMU Driver Node"]
    end

    subgraph Hardware["物理硬體層 (Physical Sensors)"]
        3DLidar["3D LiDAR / 2D LiDAR"]
        DepthCam["Depth Camera / RGB Camera"]
        IMU["IMU / Encoder"]
    end

    %% 資料流與控制關係
    3DLidar -->|原始點雲| LidarDriver
    DepthCam -->|原始影像與深度| CamDriver
    IMU -->|角速度與加速度| IMUDriver

    LidarDriver -->|sensor_msgs/PointCloud2| ROS2
    CamDriver -->|sensor_msgs/Image| ROS2
    IMUDriver -->|sensor_msgs/Imu| ROS2

    ROS2 -->|時間同步與校準數據| SensorFusion
    SensorFusion -->|融合後的環境狀態| SLAM
    SLAM -->|機器人位姿 (Pose)| Nav
```

---

### 2.2 核心關鍵概念解析

#### 2.2.1 機器人感知系統 (Robot Perception System)
機器人感知系統是從標準機器人開發標準流程（SOP）中獨立拉出來的一個專門子系統。其核心目的在於「讓機器人看得到並理解周遭環境」。
在實務開發上，感知系統通常會負責環境點雲過濾、影像去噪、特徵提取等前處理工作。有了穩固的感知系統，後續的定位與避障演算法才能獲得乾淨、高頻率、一致性的觀測數據（Observations），而不會因為單一感測器的故障或雜訊導致機器人行為失控。

#### 2.2.2 主流框架 ROS 2 與多感測器融合 (Sensor Fusion)
目前全球學術界與機器人產業，皆已將 **ROS 2** 作為開發的首選框架。
ROS 2 引入了去中心化的 DDS (Data Distribution Service) 技術，具備高即時性、服務品質（QoS）設定與多平台支援等特性。在感知領域，多感測器融合（如將 3D 光達與深度相機、IMU 的數據融合）演算法是實現高可靠性自主導航的核心。ROS 2 的訂閱發布（Pub/Sub）機制，能輕鬆讓一個融合節點（Fusion Node）訂閱來自不同硬體 driver 的話題，在記憶體中進行時間對齊與座標變換，最後將融合數據打包發布。

#### 2.2.3 整機廠商 (Robot OEM)：秉持模組化與 ISO 22166 精神
對於開發機器人整機的廠商而言，極力建議在軟硬體設計上秉持**模組化精神**。
這不僅有利於產品的快速迭代（例如：在不修改定位演算法的情況下，直接更換不同廠牌的光達），更有助於後續的客戶維運與售後服務。
國際標準 **ISO 22166** (Robotics - Modularity for service robots) 明確規範了服務機器人在軟硬體介面上的模組化設計指南。透過 ROS 2 的標準介面格式（如將雷達格式標準化為 `sensor_msgs/msg/LaserScan`），整機廠商能符合此國際標準，進而打入國際頂尖的機器人生態系。

#### 2.2.4 零組件商 (Component Vendor)：Intel RealSense 與 ROS 的小故事
對於感測器、馬達等零組件供應商，若想打入機器人市場，**提供對應的官方 ROS/ROS 2 driver 是極為關鍵的商業決策**。
這背後有一個廣為人知的經典故事：早期 Intel 推出了 RealSense 深度相機，其優秀的深度探測能力吸引了大量機器人開發者。然而，當時官方並未提供完善的 ROS 整合，社群開發者必須自行包裝 API，導致維護困難、Bug 頻傳。
Intel 隨後意識到機器人社群的巨大市場潛力，正式投入資源成立官方團隊來開發與維護 `realsense-ros` 套件。如今，開發者只需一行 `apt install ros-humble-realsense2-camera` 即可在 ROS 2 中完美讀取影像。這極大降低了開發門檻，也讓 Intel RealSense 一躍成為全球機器人感知領域的統治級硬體。
這段歷史告訴我們：**零組件若沒有官方維護的 ROS 2 driver，即便硬體規格再好，也極難在現代機器人市場中取得商業成功**。

#### 2.2.5 兩大主流運算硬體系統：NVIDIA 與 Intel
在大腦（高階邊緣運算平台）的選擇上，市場目前分為兩大技術陣營：
1. **NVIDIA (Jetson / Arm 體系)**：
   - **優勢**：具備強大的 GPU 與 Tensor 核心，極度適合執行基於卷積神經網路（CNN）或 Transformer 的 AI 演算法（如 YOLO 障礙物偵測、BevFusion 多相機鳥瞰圖融合）。
   - **定位**：高階、強調 AI 感知、多相機融合的移動機器人與人型機器人主控。
2. **Intel (x86 體系)**：
   - **優勢**：單核心 CPU 效能強勁，極為適合需要密集邏輯運算、矩陣分解的傳統 3D SLAM（如非線性優化 g2o、Ceres-Solver）與路徑規劃演算法。其生態系相容性極高，編譯 C++ 程式速度極快。
   - **定位**：強調高精度雷達定位、講求穩定邏輯運算與快速開發的工業級 AGV/AMR。
