# 融合前處理 (Sensor Fusion Preprocessing)

•	空間校準 Extrinsic Calibration：機器人於設計時會有很多感測器（如光學雷達、深度相機、慣性測量單元、輪軸編碼器等），每個感測器蒐集到的資料都是「以自己為中心（自己的座標系）」發布的。如果沒有對各感測器進行精確的空間校準，雷達看到的牆壁、相機看到的障礙物、IMU 量測到的加速度就會「對不齊」，導致建圖模糊、避障失效或定位飄移。故，空間校準的主要的目的為讓機器人知道各個感測器與機構組件，在三維空間中「相對的精確位置與姿態」。
o	空間校準的核心目的，是估算各感測器與機器人本體之間的相對位姿，並建立正確的座標轉換關係（TF Tree），在 ROS 2 中，空間校準的核心成果最終會落實為 tf2 座標變換系（Transform Library） 中的靜態變換（static_transform_publisher）或 URDF 機器人模型檔中的 <joint> 參數。
o	常見作法：空間校準依據感測器的種類不同，使用的數學原理與校準工具也有所差異，較常見的情境與做法如下說明：
	多雷達/雷達對車體（LiDAR-to-Body / Multi-LiDAR Calibration）：為了建立 LiDAR 與機器人本體座標系（base_link）之間的相對位姿，或建立多顆 LiDAR 之間的相對位姿，以融合其量測資料並消除單顆 LiDAR 的視野死角，需進行 LiDAR 外部參數校正（Extrinsic Calibration）。
•	機構安裝校正：依據 CAD 設計尺寸，搭配安裝治具或高精度雷射水平儀，手動或半自動調整 LiDAR 的安裝位置（X, Y, Z）及姿態（Roll、Pitch、Yaw），使其符合設計規格。
•	標靶式校正（Target-based Calibration）：利用已知幾何位置的校準治具（Calibration Fixture）或人工標靶（如 V Landmark），建立 Ground Truth，透過感測器量測結果與標靶真實位置的差異，估算 LiDAR 與機器人本體之間的外部參數。此方法不依賴機器人運動或里程計精度，適合現場快速校正。
•	環境特徵匹配（ICP 法）：利用不同 LiDAR 對相同環境所掃描的重疊點雲，透過迭代最近點演算法（ICP, Iterative Closest Point）進行點雲配準（Registration），反覆估算兩組點雲之間的最佳剛體變換（Rigid Transformation），以求得多顆 LiDAR 之間的相對外部參數（Extrinsic Parameters）。
	雷達與 IMU（LiDAR-IMU Calibration）：估算 LiDAR 與 IMU 之間的外部參數（平移與旋轉）及時間同步偏移，使兩種感測器的量測能對齊至相同座標系與時間軸，以提升 LiDAR-Inertial Odometry、SLAM 與定位的精度。
•	手持/移動運動校準：讓機器人或手持設備執行多方向的旋轉與平移運動，收集 LiDAR（或相機）與 IMU 的同步量測資料，透過聯合最佳化估算兩者之間的外部參數及時間同步偏移。
校準類型	常見開源工具 / 套件	說明
LiDAR 到 IMU	LI-Calib / FAST-LIO-CALIB	自動化估算 LiDAR 與 IMU 之間的高精度旋轉矩陣與平移量。
多LiDAR校準	multi_lidar_calibration	車用/大型機器人多雷達點雲拼接校準。

•	時間校準 Time Synchronization：機器人身上的各個感測器在蒐集資料時，各自都有獨立的採樣頻率與時鐘系統。如果沒有對各感測器進行精確的時間校準，當機器人處於移動狀態時，就會因為「時間對不上」而產生數據時間差，會導致感測器融合時發生嚴重的視覺殘影、空間錯位、建圖模糊或動態定位失效。故，時間校準的主要目的為讓機器人各感測器所產生的數據，在時間軸上擁有「高精度的統一基準與同步時戳」，確保所有資訊能在同一時間點被準確對齊與融合。
o	時間同步通常分為「時間基準同步（Clock Synchronization）」與「資料時間對齊（Data Association）」：
	時間基準同步（Clock Synchronization）
•	PTP (Precision Time Protocol / IEEE 1588)：
o	原理：透過乙太網路傳輸 IEEE 1588 時間同步封包，由主時鐘（Grandmaster Clock，通常為 IPC、工業交換器或 GNSS 時間伺服器）持續校正各裝置的硬體時鐘，使所有設備維持相同的時間基準。
•	PPS 脈衝同步 (Pulse Per Second) + NMEA / Serial：
o	原理：通常由 GNSS（GPS）接收器或時間基準設備 每秒輸出一個高精度 PPS（Pulse Per Second）脈衝，所有感測器或主機收到脈衝後，將其作為共同的時間基準來校正自身時鐘；同時透過 NMEA 或 Serial 傳送絕對時間（如 UTC 時間），讓系統知道每個 PPS 脈衝所對應的實際時間。
•	外觸發訊號 (External Trigger / Strobe)：
o	原理：由主控板（如 MCU 或 FPGA）輸出硬體觸發訊號（Trigger Signal），同步觸發相機曝光或 LiDAR 開始掃描，使多個感測器在同一物理時刻產生資料。
	資料時間對齊（Data Association）：當硬體不支援 PTP 或外觸發時，可利用 ROS 2 的時間戳（Timestamp）在接收資料後，依據時間戳將不同來源的資料配對。此方式不會校正感測器時鐘，而是依據時間戳將不同來源的資料對齊，用於後續感測器融合。
•	ExactTime (精確時間匹配)：
o	只有當多個 Topic 的 header.stamp（時間戳）完全一模一樣時才觸發 Callback。
o	適用：已經做過硬體同步的多相機系統。
•	ApproximateTime (近似時間匹配)：
o	允許設定一個時間容許值（Slop，例如 ±10ms）。演算法會在佇列（Queue）中尋找時間最接近的一組資料進行配對。
o	適用：沒有硬體同步、各感測器採樣頻率不一致的情境（如 10Hz的LiDAR 搭配 30Hz 的 Camera）。
•	即使感測器已完成時間同步並具有時間戳，仍可能因為曝光延遲、傳輸延遲或硬體處理延遲，使時間戳與實際資料產生時刻存在固定偏差（Time Offset），因此仍需進行時間外參校準（Temporal Calibration）：
o	最佳化校準（Optimization-based Calibration）：
	可透過 Kalibr（Camera–IMU）、LI-Calib（LiDAR–IMU）、FAST-LIO-CALIB（LiDAR–IMU） 等校準工具，將固定時間偏移（Time Offset）視為待估參數，並與空間外參共同進行最佳化求解，以提升感測器融合精度。
o	動態軌跡對齊：
	讓機器人執行具有足夠角速度與加速度變化的運動，分別計算相機或 LiDAR 推估的角速度，以及 IMU 量測的角速度，再利用互相關（Cross-Correlation）或非線性最佳化，估計兩組訊號之間的固定時間偏移量。
