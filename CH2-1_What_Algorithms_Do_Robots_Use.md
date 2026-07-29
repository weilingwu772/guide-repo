# 主流演算法 (What Algorithms Do Robots Use)

在智慧機器人的開發中，「感測器」負責蒐集環境與自身狀態數據，而「演算法」則是將這些龐雜數據轉換為定位、路徑規劃及控制命令等資訊，機器人自主移動中的每個階段，都依賴不同特性與架構的演算法相互配合。對開發者來說，關鍵往往不在於從頭撰寫演算法，而是精準掌握各演算法的**適用場景與既存限制**，以及對應所需的感測器硬體、輸入 Topic 和輸出 ROS 2 Message，本節將盤點目前機器人感知領域中最主流的演算法，協助開發者能依據實際應用需求，快速選擇合適的解決方案。

## 1. 雷達建圖與定位 (LiDAR SLAM & Localization)

| 演算法 | 特點 / 優缺點 | 對接硬體與輸入 Topic (說明) | 產出 Message |
| :--- | :--- | :--- | :--- |
| **Gmapping** | **說明**：基於粒子濾波器（Particle Filter）架構，是早期經典的 2D 光達建圖演算法，多用於室內小型輪式機器人（如掃地機器人）。<br><br>**優點**：<br>1. 結構簡單，計算資源消耗低<br>2. 在小範圍室內環境中，建圖精度高且具穩定性<br><br>**缺點**：<br>1. 不適合大範圍場景，累積誤差會讓地圖嚴重變形<br>2. 純光達主導，不支援 3D 建圖與 IMU 緊耦合 | • 2D / 3D 光達 ➔ `/scan` (只接收 2D 掃描數據)<br>• 編碼器 ➔ `/odom`、 `base_link`<br>•  `/tf` | • `nav_msgs/OccupancyGrid`<br>• `nav_msgs/MapMetaData`<br>• `tf2_msgs/TFMessage` |
| **Cartographer** | **說明**：基於圖優化（Graph-based）架構，是工業界主流的 2D / 3D 光達建圖演算法，多用於工廠運行的 AGV 或 AMR 。<br><br>**優點**：<br>1. 強大閉環檢測（Loop Closure）能力，當機器人回到初始點會自動校正累積誤差<br>2. 適合大範圍與複雜的室內環境，可直接融合 IMU 數據即時校正機器人的重力方向<br><br>**缺點**：<br>1. 為了能隨時校正，計算資源消耗高<br>2. 遇到低特徵環境，容易產生誤判或建圖錯位 | • 2D 光達 ➔ `/scan` <br>• 3D 光達 ➔ `/points2` <br>• IMU ➔ `/imu`<br>• 編碼器 ➔ `/odom` (選配) | • `cartographer_ros_msgs/SubmapList`<br>• `visualization_msgs/MarkerArray`<br>• `tf2_msgs/TFMessage` |
| **SLAM Toolbox** | **說明**：基於圖優化架構，是 ROS 2 官方預設的 2D 光達建圖演算法，被視為 Cartographer 的輕量化替代方案。<br><br>**優點**：<br>1. 跟Cartographer相比，除了計算資源較省，建圖速度也較快<br>2. 支援地圖序列化（Lifelong Mapping）及非同步建圖（Asynchronous Mapping），前者可進行地圖儲存與載入後的持續更新，後者能降低處理延遲並提升大場景建圖效率<br><br>**缺點**：<br>1. 不支援 3D 建圖<br>2. 不內建處理 IMU，需另用套件融合`/odom`後再輸入 | • 2D / 3D 光達 ➔ `/scan` (只接收 2D 掃描數據)<br>• 編碼器 ➔ `/odom`<br>• `/tf` | • `nav_msgs/msg/OccupancyGrid`<br>• `nav_msgs/msg/MapMetaData`<br>• `visualization_msgs/msg/MarkerArray`<br>• `tf2_msgs/msg/TFMessage` |
| **LIO-SAM** | **說明**：基於光達慣性里程計（LiDAR-Inertial Odometry）架構的 3D 光達建圖演算法，多用於地形不平整的室外環境。<br><br>**優點**：<br>1. 光達與 IMU 緊耦合，在劇烈運動或顛簸狀態下仍能維持高精度定位<br>2. 透過因子圖優化（Factor Graph），建圖速度快<br><br>**缺點**：<br>1. 硬體要求高，感測器品質及參數校正皆直接影響地圖品質<br>2. 遇到狹窄、人多的場景，定位穩定性可能下降 | • 3D 光達 ➔ `/points_raw` (可接收原始 3D 點雲)<br>• IMU ➔ `/imu/data` (必須接收原始高頻率[>200Hz]及高精度[六軸或九軸]的IMU數據)<br>• GPS ➔ `/gps/fix` (選配，消除全域漂移，室內不適用) | • `nav_msgs/Odometry`<br>&nbsp;&nbsp;(`/lio_sam/mapping/odometry`)<br>• `sensor_msgs/PointCloud2`<br>&nbsp;&nbsp;(`/lio_sam/mapping/cloud_registered`)<br>• `sensor_msgs/msg/PointCloud2` |

## 2. 視覺建圖與定位 (Visual SLAM & Localization)

| 演算法 | 特點 / 優缺點 | 對接硬體與輸入 Topic (說明) | 產出 Message |
| :--- | :--- | :--- | :--- |
| **ORB-SLAM3** | **說明**：基於特徵點法（Feature-based Method）的視覺 SLAM 主流演算法。<br><br>**優點**：<br>1. 以視覺為核心，相機與 IMU 緊耦合，可直接利用 IMU 數據即時優化定位<br>2. 在無人機、AR / VR 眼鏡、無 GPS 環境下的輕量化定位表現極佳（公分級）<br><br>**缺點**：<br>1. 遇到低光源、反光或低特徵環境，會直接丟失定位<br>2. 產出的地圖為稀疏點雲（Sparse Point Cloud），僅供定位無法直接導航 | • RGB 相機 ➔ `/camera/rgb/image_raw`<br>• 深度相機 ➔ `/camera/depth/image_raw`、<br>`/camera/left/image_raw`、<br>`/camera/right/image_raw`<br>• IMU ➔ `/imu` (必須接收原始高頻率[>100Hz]的IMU數據) | • `geometry_msgs/PoseStamped`<br>• `sensor_msgs/PointCloud2`<br>• `visualization_msgs/Marker`<br>• `tf2_msgs/msg/TFMessage` |
| **RTAB-Map** | **說明**：基於圖優化架構，結合視覺與光達的 3D 稠密建圖演算法，以記憶體管理機制聞名。<br><br>**優點**：<br>1. 支援多種感測器，包含相機、2D / 3D光達、IMU及GPS<br>2. 可同時建立三維點雲地圖與二維佔據網格地圖，並具備強大的閉環檢測與重定位能力（Kidnapped Robot Problem）<br><br>**缺點**：<br>1. 也因支援多種感測器，導致其設定極其複雜<br>2. 隨地圖規模增加，記憶體與運算資源需求也顯著提升 | • RGB-D 相機 ➔ `/Camera/RGBD`<br>• 2D / 3D 光達 ➔ `/LiDAR` (修正視覺深度誤差)<br>• 編碼器 ➔ `/odom`<br>• IMU ➔ `/imu` | • `nav_msgs/msg/OccupancyGrid`<br>• `sensor_msgs/msg/PointCloud2`<br>• `geometry_msgs/msg/PoseWithCovarianceStamped`<br>• `tf2_msgs/msg/TFMessage` |

## 3. 定位 (Localization)

| 演算法 | 特點 / 優缺點 | 對接硬體與輸入 Topic (說明) | 產出 Message |
| :--- | :--- | :--- | :--- |
| **AMCL**<br>*(Monte Carlo Localization)* | **說明**：基於粒子濾波器架構，ROS 導航內建的經典定位法。<br><br>**優點**：<br>1. 在地圖中灑出大量「可能位置的粒子」，透過光達掃描與地圖的比對，逐步收斂到唯一正確的位置<br>2. 計算資源消耗低，但非常依賴環境特徵<br><br>**缺點**：<br>1. 必須在有現成地圖的情況下才能運作<br>2. 若環境變動太大時（例如預設的地圖是空的，但現實中被貨物堆滿）就容易失準 | • `/map` (可自建也可匯入的靜態黑白地圖)<br>• 2D 光達 ➔ `/scan`<br>• `/tf` | • `geometry_msgs/PoseWithCovarianceStamped`<br>• `geometry_msgs/PoseArray`<br>• `tf2_msgs/TFMessage` |
| **robot_localization** | **說明**：基於卡爾曼濾波（Kalman Filter）架構的狀態估計（State Estimation）套件，負責融合多種感測器資訊並輸出機器人姿態與里程計資訊（`/odom`）。<br><br>**優點**：<br>1. 支援輪速計、IMU、GPS、視覺里程計等多種感測器<br>2. 可降低單一感測器的雜訊與漂移，提高定位穩定性<br><br>**缺點**：<br>1. 僅負責感測器融合與狀態估計，不具備建圖功能<br>2. 融合效果高度依賴感測器品質與參數設定 | • 編碼器 ➔ `/odom/wheel`<br>• IMU ➔ `/imu/data`<br>• GPS ➔ `/gps/fix` (選配，改善全域漂移，室內不適用) | • `geometry_msgs/msg/TransformStamped` |

## 4. 導航與避障 (Navigation & Obstacle Avoidance)

| 演算法 | 特點 / 優缺點 | 對接硬體與輸入 Topic (說明) | 產出 Message |
| :--- | :--- | :--- | :--- |
| **A* (A-Star) / NavFn***<br>*(全域規劃)* | **說明**：ROS 2 (Nav2) 常用的全域規劃器，屬啟發式圖形搜尋（Heuristic Search）方法。<br><br>**優點**：<br>1. 將地圖視為網格，以幾何距離加上預估成本，在毫秒內找出最短路徑<br>2. 邏輯簡單、執行穩定<br><br>**缺點**：<br>1. 網格規劃使路線較生硬（直角多）<br>2. 地圖太大時，搜尋會變慢 | • `/goal_pose`<br>• `/global_costmap/costmap`<br>• `/tf` | • `nav_msgs/msg/Path` |
| **Theta* / Smac Planner**<br>*(全域規劃)* | **說明**：ROS 2 (Nav2) 常用的全域規劃器。<br><br>**優點**：<br>1. 打破網格限制，允許機器人規劃出任意角度的平滑斜線，更符合實際物理車輛的行駛軌跡<br>2. 產出的路徑較為平順，可減少額外平滑處理需求<br><br>**缺點**：<br>1. 運算複雜度比傳統 A* 高，並需要較好的處理器<br>2. 在障礙物密集或高解析度地圖下，規劃時間可能增加 | • `/goal_pose`<br>• `/global_costmap/costmap`<br>• `/tf` | • `nav_msgs/msg/Path` |
| **DWA**<br>*(Dynamic Window Approach)*<br>*(局部控制)* | **說明**：傳統 2D 避障與局部跟隨的速度空間採樣演算法。<br><br>**優點**：<br>1. 在機器人可執行的速度與轉速範圍內進行「模擬試車」，捨棄會撞到障礙物的速度組合，挑選出最安全且最接近全域路徑的速度<br>2. 計算量低且反應快速，適合室內差速驅動機器人<br><br>**缺點**：<br>1. 易陷入局部最優解（例如在狹窄通道卡死）<br>2. 不適合車身很長，並像汽車一樣有迴轉半徑限制的機器人 | • `/local_costmap/costmap`<br>• `/plan`<br>• 編碼器 ➔ `/odom`<br>• `/tf` | • `geometry_msgs/msg/Twist` |
| **RPP**<br>*(Regulated Pure Pursuit)*<br>*(局部控制)* | **說明**：基於幾何追隨的演算法，多用於工業級 AGV、大型搬運車。<br><br>**優點**：<br>1. 極度穩定且計算量極低，在接近轉角或障礙物時，會自動調速不易甩尾<br>2. 適合讓機器人嚴格沿著既定軌道（像是地上畫的線或虛擬軌道）走的場景<br><br>**缺點**：<br>1. 不具備主動繞障能力，高度依賴全域路徑引導<br>2. 若全域路徑品質不佳或環境變化劇烈，追蹤效果容易受到影響 | • `/plan`<br>• 編碼器 ➔ `/odom`<br>• `/tf` | • `geometry_msgs/msg/Twist` |
| **TEB**<br>*(Timed Elastic Band)*<br>*(局部控制)* | **說明**：彈性帶式時空（時間與空間）優化演算法，高性能避障演算法之一。<br><br>**優點**：<br>1. 將路徑視為可伸縮的彈性帶，透過最佳化方式同時調整路徑形狀與移動速度，以產生平順且可行的局部軌跡（Local Trajectory）<br>2. 支援非圓形車體（如長方形 AGV 或 AMR），能實現流暢的前進、後退與繞行<br><br>**缺點**：<br>1. 演算法較複雜，在動態障礙物密集的環境下可能產生較高 CPU 負載<br>2. 可能出現路徑震盪或反覆切換繞行策略的狀況 | • `/plan`<br>• `/local_costmap/costmap`<br>• 編碼器 ➔ `/odom`<br>• `/tf` | • `geometry_msgs/msg/Twist` |
| **MPPI**<br>*(Model Predictive Path Integral)*<br>*(局部控制)* | **說明**：結合模型預測控制（Model Predictive Control）與路徑積分採樣（Path Integral Sampling）方法，現代導航最強演算法，適用於高速無人機、戶外越野無人車、以及極高動態避障需求的機器人。<br><br>**優點**：<br>1. 有效處理高速運動、多障礙物及複雜動態環境，並支援多種移動平台及運動模型<br>2. 模擬路徑多，可大幅降低陷入局部最佳解的機率<br><br>**缺點**：<br>1. 計算量極大，高度依賴多核心 CPU 或 GPU 加速<br>2. 參數較多且調校難度高，並需要根據車體模型與應用場景進行優化 | • `/plan`<br>• `/local_costmap/costmap`<br>• 編碼器 ➔ `/odom`<br>• `/tf` | • `geometry_msgs/msg/Twist` |
