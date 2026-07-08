# 感測器融合的 ROS 2 工程標準 (ROS 2 engineering standards for sensor fusion)

在機器人開發中，**工程標準**是確保系統穩定與不同演算法（如 Nav2、Cartographer、robot_localization）順利串接的關鍵。本章將從 **REP d開發規範**與**國際主流設備節點設計**兩個面向，為開發人員提供寫程式時的具體參考。

---

## ROS Enhancement Proposal (REP) 開發規範

以下依開發機器人時的工作順序（確認開發版本 $\rightarrow$ 統一影像資料格式 $\rightarrow$ 統一物理量標準）來介紹常用的 REP 開發規範文件，並整理出與機器人感知系統較為相關的內容，供開發者參考，依循此規範將有助於後續使用 ROS 2 進行多感測器融合 (sensor fusion) 演算法。

### 1. 確認開發版本

開發的首要工作，是先決定這個機器人開發專案要使用哪個版本的 ROS 2，或依據既有的作業系統選擇相容的 ROS 2 版本，這是因為不同的 ROS 2 版本，其支援的作業系統、編譯工具及套件皆有所不同，因此就需要透過 REP-2000 查閱，以確認各版本的支援資訊。

* **REP-2000 ROS 2 Releases and Target Platforms**

    REP-2000 除了說明 ROS 2 的發行策略（一年一版，並分為長期支援 LTS 版本與非長期支援 Non-LTS 版本），也定義每個 ROS 2 版本對應支援的作業系統有哪些，同時將各個作業系統被支援的程度分為三個等級，Tier 1 代表完全支援，Tier 2 次之，Tier 3 則是最少支援，僅提供基本相容性。
    
    因此，為避免因版本問題而收到錯誤訊息，導致無法順利啟動 ROS 2，建議先使用這份文件做版本查詢，檢索方法如下：
    
    假設已決定要使用 ROS 2 的 Humble Hawksbill 作為開發版本，先在 REP-2000 找到該版本名稱，並可從中看到對應支援的作業系統（Target Platform）表格，而此版本Tier 1（也就是完全支援）作業系統有：
    * Ubuntu Jammy (22.04) 的 64 位元版本
    * Windows 10 (VS2019)
    
    接著查看 Ubuntu 官方的 Python 版本對照表 (Available Python versions)，就能得知 Ubuntu 22.04 對應 Python 3.10 版本，也就是說，要用 Python 3.10 才能完整安裝及執行 ROS 2 Humble Hawksbill。

### 2. 統一影像資料格式

不同感測器隨著廠牌和類型，會有不同形式的距離或影像資料，若缺乏一致的資料格式，容易造成後續演算法的相容性問題；因此就需要參考 REP-117 與 REP-118 來確保軟體層面輸出的資料符合 ROS 2 官方標準。

* **REP-117 Informational Distance Measurements**

    REP-117 規範 ROS 與 PCL (Point Cloud Library，專門處理 3D 點雲資料的開源 C++ 函式庫)中，物理距離(Physical Distance)測量值的表示方式，此規範適用於各種距離感測資料，例如 `sensor_msgs/Range.msg`、`sensor_msgs/LaserScan.msg`、`sensor_msgs/PointCloud2.msg` 三種格式，並強調當感測器無法取得有效距離時，應使用特殊數值（`-Inf`、`+Inf`、`NaN`）來表示不同的測量狀態，避免不同廠商或驅動使用不同的表示方式，造成後續演算法誤判。
    
    舉例來說，傳統開發可能出現 `0.0`、`-1.0`、`minimum_range`、`maximum_range` 或其他自訂數值，來表示超出量測範圍或量測異常，資料語意的不同，容易導致演算法搞錯意思，造成機器人緊急煞車或路徑規劃出錯。REP-117 建議遵循以下形式：
    
     - 當測量值**低於感測器最小偵測距離**時，數據應設為 **`-Inf`**（Negative Infinity），表示物體距離過近，已超出感測器可量測範圍最小值，而非量測失敗。
     - 當測量值**高於感測器最大偵測距離**或未偵測到任何物體時，數據應設為 **`+Inf`**（Positive Infinity），表示目標超出感測器的量測範圍，或該方向沒有接收到有效回波。
     - 當感測器發生錯誤、資料遺失或量測結果無效時，數據應設為 **`NaN`**（Not a Number），表示此筆量測資料不可用，請演算法視為無效資料並直接忽略。

* **REP-118 Depth Images**

    REP-118 規範 ROS 中深度影像（Depth Image）的表示方式，包含輸出的資料格式、單位、`Topic`，使不同廠商或不同技術的深度相機（如 Stereo Camera、Structured Light、Time-of-Flight）都能輸出相同格式的深度影像，方便後續演算法直接使用。
    
    舉例來說，感測器應使用 `sensor_msgs/Image` 來表示深度影像，而非 `sensor_msgs/DisparityImage`，其中每個像素沿相機 Z 軸的深度值應為 32-bit float（單位為公尺）格式，若使用 16-bit unsigned integer（單位為毫米）格式，則必須在說明文件中明確聲明並進行轉換，最後則是搭配 `camera_info` Topic 進行發布，演算法對接後就能基於此建立三維點雲。

### 3. 統一物理量標準

在寫多感測器融合演算法時，數據的空間與時間對齊是核心，若不同感測器使用不同的度量單位、坐標系或坐標框架，即使取得的是同一個物體的資訊，也可能因解讀方式不同而產生融合誤差，因此開發前就需參考 REP-103 與 REP-105，確保不同感測器能在 ROS 中正確地交換與整合資料。

* **REP-103 Standard Units of Measure and Coordinate Conventions**

  REP-103 定義出 ROS 中的測量單位、坐標系、旋轉表示方式與協方差矩陣，確保不同軟體元件之間在處理物理量時能保持一致，避免因單位不同導致的計算錯誤。規定如下：

  * **度量單位**：全面使用國際單位制 (International System of Units，簡稱SI)

    | 物理量                    | 單位           |
    | --------------------- | ----------------- |
    | 長度（Length）            | meter          |
    | 質量（Mass）              | kilogram       |
    | 時間（Time）              | second         |
    | 電流（Current）           | ampere         |
    | 角度（Angle）             | radian         |
    | 頻率（Frequency）         | hertz          |
    | 力（Force）               | newton         |
    | 扭矩（power）             | watt           |
    | 電壓（voltage）           | volt           |
    | 溫度（temperature）       | celsius        |
    | 磁場（magnetism）         | tesla          |

  * **坐標系**：採用右手坐標系 (Right-Handed Coordinate System)
    - X 軸：Forward（指向機器人前方）
    - Y 軸：Left（指向機器人左方）
    - Z 軸：Up（垂直向上）
      
  * **大地坐標系**：採用 ENU (East-North-Up) 坐標系
    - X 軸：East（東方）
    - Y 軸：North（北方）
    - Z 軸：Up（垂直向上）
      
  * **具有`_optical`後綴的坐標系**：相機使用的坐標系與機器人本體不同
    - X 軸：Right（右方）
    - Y 軸：Down（下方）
    - Z 軸：Forward（向前）
      
  * **旋轉表示方式**：逆時針方向為正
    1. Quaternion：最建議使用，用四個數值表示旋轉，表示方式相對緊湊且無奇異點(singularities) 。
    2. Rotation Matrix：無奇異點。
    3. Fixed-axis Roll-Pitch-Yaw：依序繞 Y、X、Z 軸的角速度。
    4. Euler Angles：最不建議使用，因其存在 24 種旋轉慣例，容易造成姿態解讀不一致。
      
  * **協方差矩陣**：各種感測器的協方差矩陣必須依照固定順序排序，例如 IMU 的線性加速度協方差矩陣，採用 x、y、z 的 Row-major（以列為主） 順序儲存。

* **REP-105 Coordinate Frames for Mobile Platforms**

    REP-105 建立移動平台（Mobile Platforms）的坐標系命名規範，使驅動、模型、函式庫及應用程式能共享相同的座標框架定義，不需因不同機器人而修改程式；文件中定義了四個主要座標系：**`base_link`機器人本體座標系**、**`odom`里程計座標系**、**`map`地圖座標系** 與 **`earth`地球座標系**，彼此的關係架構如下，其中單一室內機器人通常只會使用 base_link、odom 與 map 三個座標系，earth 則主要應用於戶外應用情境。

```mermaid
 graph LR
     earth --> map
     map --> odom
     odom --> base_link
```

---

## 2. 從國際主流產品 ROS package 彙整 Node 類別 (從 Launch 回推)

為了解國際大廠如何將硬體接入 ROS 2 體系，我們可以從各家官方的 Launch 檔宣告回推，歸納出它們的 Node 類別架構設計。以下彙整了 Intel RealSense、RPLIDAR 以及 Velodyne 等主流產品的工程實踐。

### 1. 典型主流產品的 Node 宣告回推

#### 案例 A：Intel RealSense (D400系列) — 影像與深度感測器
- **官方 Package**：`realsense2_camera`
- **Launch 檔宣告回推**：
  ```python
  from launch_ros.actions import Node
  # 宣告 RealSense 驅動節點
  realsense_node = Node(
      package='realsense2_camera',
      executable='realsense2_camera_node',
      name='camera',
      namespace='camera',
      parameters=[{
          'enable_color': True,
          'enable_depth': True,
          'depth_module.profile': '640x480x30',
          'pointcloud.enable': True
      }]
  )
  ```
- **Node 類別歸納**：
  - **`Driver Node` (硬體驅動/數據提供者)**：主要發布 `sensor_msgs/msg/Image` (RGB & Depth)、`sensor_msgs/msg/PointCloud2` 與 `sensor_msgs/msg/Imu`。
  - **設計特色**：採用 ROS 2 的 **Component / Lifecycle** 架構。驅動節點繼承自 `rclcpp::Node` 或 `rclcpp_lifecycle::LifecycleNode`，將相機硬體的初始化與 ROS 通訊生命週期解耦。

#### 案例 B：RPLIDAR (Slamtec) — 2D 固態/旋轉光達
- **官方 Package**：`rplidar_ros`
- **Launch 檔宣告回推**：
  ```python
  rplidar_node = Node(
      package='rplidar_ros',
      executable='rplidar_node',
      name='rplidar_node',
      parameters=[{
          'channel_type': 'serial',
          'serial_port': '/dev/rplidar',
          'serial_baudrate': 115200,
          'frame_id': 'laser_frame',
          'inverted': False,
          'angle_compensate': True,
      }],
      output='screen'
  )
  ```
- **Node 類別歸納**：
  - **`Sensor Data Publisher Node` (單一數據發布節點)**：主要發布 `sensor_msgs/msg/LaserScan`。
  - **設計特色**：著重於底層串口通訊 (UART to USB) 的斷線重連機制與執行緒管理。內部會開闢一個工作執行緒 (Worker Thread) 持續讀取串口緩衝區，並使用 `std::mutex` 與 ROS 發布主執行緒進行執行緒安全（Thread-Safe）的資料傳遞。

#### 案例 C：Velodyne (3D 光達) — 多線式 3D 光達
- **官方 Package**：`velodyne_driver` 與 `velodyne_pointcloud`
- **Launch 檔宣告回推**：
  在 Launch 檔中，Velodyne 採用了**管線化 (Pipeline) / 組件化 (Composable Nodes)** 設計：
  ```python
  from launch_ros.actions import ComposableNodeContainer
  from launch_ros.descriptions import ComposableNode

  container = ComposableNodeContainer(
      name='velodyne_container',
      namespace='',
      package='rclcpp_components',
      executable='component_container',
      composable_node_descriptions=[
          # 節點 1: 負責接收 UDP 原始封包 (Raw Packets)
          ComposableNode(
              package='velodyne_driver',
              plugin='velodyne_driver::VelodyneDriver',
              name='velodyne_driver_node',
              parameters=[{'device_ip': '192.168.1.201'}]
          ),
          # 節點 2: 負責將 Raw Packets 轉換為 PointCloud2 點雲
          ComposableNode(
              package='velodyne_pointcloud',
              plugin='velodyne_pointcloud::Convert',
              name='velodyne_convert_node',
              parameters=[{'calibration': 'VLP16db.yaml'}]
          )
      ]
  )
  ```
- **Node 類別歸納**：
  - **`Data Ingestion Node` (數據接入節點)**：發布原始封包 (如 `velodyne_msgs/msg/VelodyneScan`)。
  - **`Data Processing/Nodelet Node` (數據處理節點)**：訂閱原始封包，發布標準的 `sensor_msgs/msg/PointCloud2`。
  - **設計特色 (Zero-Copy 共享記憶體)**：使用 Composable Nodes (組件化節點)。這類 Node 不直接繼承獨立的 Process，而是編譯為動態連結檔 (Plugin)。當兩者被加載至同一個 `component_container` 時，ROS 2 內部的 `intra-process communication` 機制會以**指標傳遞**的方式在節點間傳輸大容量點雲，實現 **Zero-Copy (零拷貝)**，大幅節省 CPU 資源。

### 2. 彙整：感測器驅動 Node 架構設計建議

綜合上述主流產品的 Launch 與原始碼架構，當你為新硬體編寫用於融合的 ROS 2 驅動時，建議參考以下 Node 類別結構：

```
                        ┌────────────────────────┐
                        │   Physical Sensor      │
                        └───────────┬────────────┘
                                    │ 底層通訊: Serial/UDP/I2C
                                    ▼
                        ┌────────────────────────┐
                        │   Hardware SDK Layer   │
                        │ (分離 ROS 依賴的純 C++ 類)│
                        └───────────┬────────────┘
                                    │
                       (std::thread / Worker Loop)
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ ROS 2 Wrapper (LifecycleNode / ComposableNode)                  │
│                                                                 │
│   - Topic Publisher: 發布符合 REP-103/117/118 標準的 Topic        │
│   - Parameters Manager: 讀取 ROS 2 Params (Baudrate, frame_id)   │
│   - TF2 Static Broadcaster: 發布 sensor_frame 靜態外參座標轉換     │
│   - Lifecycle Management: on_configure(), on_activate()         │
└─────────────────────────────────────────────────────────────────┘
```

1. **分離硬體 SDK 與 ROS 2 Wrapper**：先用純 C++ 寫好感測器的連線與資料解析類別，再用 ROS 2 Component 或 LifecycleNode 將其包裝。這利於單獨在無 ROS 2 環境下進行單元測試。
2. **遵守時空標準發布**：
   - 驅動節點在發布資料時，Header 中的 `stamp` 必須填入感測器採樣時的精準時間（使用 `this->now()` 或硬體時戳對齊）。
   - `frame_id` 應設為可由 Parameter 動態配置的值（如 `laser_frame`），並在節點內部建立 `tf2_ros::StaticTransformBroadcaster` 發布初始的安裝位置坐標。
