# 感測器融合的 ROS 2 工程標準 (ROS 2 engineering standards for sensor fusion)

在機器人開發中，特別是當需要將多個感測器（如光達、深度相機、IMU）的數據融合成統一的定位與地圖時，**工程標準化**是確保系統穩定與不同演算法（如 Nav2、Cartographer、robot_localization）順利串接的關鍵。本章將從 **REP 規範**與**國際主流設備節點設計（從 Launch 回推）**兩個面向，為開發人員提供寫程式時的具體標準參考。

---

## 一、 第一面向：ROS Enhancement Proposal (REP) 開發規範

我們將按照實際的硬體與軟體開發順序（確認開發平台版本 $\rightarrow$ 挑設備看規格 $\rightarrow$ 依應用需求寫設定值）來解析核心的 REP 標準，並歸納出與感知系統密切相關的規範。

### 1. 確認開發平台版本：REP-2000 (ROS 2 Releases)
- **開發順序的第一步**是確認專案所採用的 ROS 2 發行版及其支援的基礎作業系統與編譯器版本。
- **REP-2000** 定義了各個 ROS 2 版本的生命週期與軟硬體平台依賴：
  - **Humble Hawksbill (LTS)**：對應 Ubuntu 22.04 LTS，預設編譯器為 GCC 11.2 (支援 C++17)。
  - **Jazzy Jalisco (LTS)**：對應 Ubuntu 24.04 LTS，預設支援 C++20。
- **工程參考**：撰寫 C++ 融合演算法或自訂感測器驅動時，必須確保你的 `CMakeLists.txt` 中指定的 C++ 標準（例如 `set(CMAKE_CXX_STANDARD 17)`）與 REP-2000 規定的目標發行版完全一致，避免編譯期或執行期的 ABI 相容性問題。

### 2. 挑設備看規格（資料格式）：REP-117 與 REP-118
當你挑選光達或相機時，除了硬體物理指標，還必須確保軟體層面輸出的 Topic 資料格式符合 ROS 2 官方標準。

- **REP-117: Informative Range Limits for Laser Scan Data (光達雷射掃描數據限值規範)**
  - 傳統開發常以 `0.0` 或 `-1.0` 來表示雷射測距超出範圍（太近或太遠）。這會導致 EKF 融合演算法誤判為距離障礙物 0 公尺，造成機器人緊急煞車或路徑規劃出錯。
  - **標準規範**：
    - 當測量值**低於感測器最小偵測距離**（太近）時，數據應設為 **`NaN`**（Not a Number）。
    - 當測量值**高於感測器最大偵測距離**（無反射訊號或太遠）時，數據應設為 **`+Inf`**（Positive Infinity）。
  - **寫程式參考**：在撰寫光達驅動或資料預處理節點時，必須嚴格遵循 REP-117 的數值設定方式。
- **REP-118: Depth Images (深度圖規範)**
  - 規範了 3D 相機輸出的深度圖像 (Depth Image) 格式。
  - **標準規範**：深度圖應使用 `sensor_msgs/msg/Image` 格式傳輸，其中畫素格式應為 `32FC1` (Float32，單位為**公尺**)。若使用 `16UC1` (UInt16，單位為**毫米**)，則必須在節點參數或說明文件中明確聲明並進行轉換，以便與其他 3D 視覺演算法對接。

### 3. 依應用需求寫設定值（座標與單位）：REP-103 與 REP-105
在寫感測器融合演算法時，數據的空間與時間對齊是核心。這需要所有感測器的物理單位與座標定義完全統一。

- **REP-103: Standard Units of Measure and Coordinate Conventions (度量單位與座標 Conventions)**
  - **度量單位**：全面使用國際單位制 (SI)。
    - 長度：公尺 (m)
    - 角度：弧度 (rad)，而非度數 (degree)。
    - 速度：m/s 與 rad/s。
  - **座標系（右手座標系）**：ROS 2 嚴格採用**右手座標系 (Right-Handed Coordinate System)**。
    - **X 軸**：Forward（指向機器人前方）
    - **Y 軸**：Left（指向機器人左方）
    - **Z 軸**：Up（垂直向上）
  - **旋轉方向**：逆時針方向為正。

- **REP-105: Coordinate Frames for Mobile Robots (移動機器人座標系規範)**
  - 定義了機器人系統中最核心的四個座標系層級：
    1. **`map` (地圖座標系)**：全球座標系，長期穩定但可能有突變（如 Loop Closure 修正時）。通常由 SLAM 演算法發布。
    2. **`odom` (里程計座標系)**：局部座標系，短期內連續且平滑，沒有突變，但長期會產生累積漂移。由里程計 (Wheel Odometry/Visual Odometry) 融合發布。
    3. **`base_link` (機器人本體座標系)**：固定在機器人底盤剛體中心（通常是旋轉中心）。
    4. **`sensor_frame` (感測器座標系)**：例如 `laser_frame`、`camera_link`，通過 `static_transform_publisher` 發布與 `base_link` 的相對外參。
  - **感測器融合寫法參考**：
    - `robot_localization` 節點通常會啟動兩個 EKF 實例：
      - 一個輸出 `odom` $\rightarrow$ `base_link` 的轉換（融合輪速計與 IMU，保證軌跡連續性）。
      - 另一個輸出 `map` $\rightarrow$ `odom` 的轉換（加入 GPS 或 AMCL 的絕對定位資訊，修正漂移）。
    - 感測器融合節點在讀取各感測器 Topic 時，必須利用 `tf_buffer` 查詢感測器相對於 `base_link` 的座標轉換關係，將所有數據投影至同一個座標系下再行卡爾曼濾波。

---

## 二、 第二面向：從國際主流產品 ROS package 彙整 Node 類別 (從 Launch 回推)

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
