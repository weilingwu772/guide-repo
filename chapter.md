
前言

	- 說明聚焦的方向避免爭議
第一章
基礎 Foundation
 
1. 解構機器人
The robot anatomy
	- 【畫圖】機器人→零組件→供應商
	  
	- 訂出零組件分類，以及對應人型做分層(大腦、小腦...)
	- 進而帶出國際主流產品和國內供應商名單
	- 點出主要談「感知」區塊的智慧硬體
 
2. 主流開發框架
Why use ROS 2 for robot development?
	- 【畫圖】機器人開發世界的關係圖(+系統架構)
	- 深入談以下關鍵字
		- 📸機器人感知系統：從機器人開發SOP拉出來，為了讓機器人看得到而須建立的系統
		- 📂ROS 2：目前主流為ROS 2框架，進而談多感測器融合(sensor fusion)的演算法
		- 🧑整機廠商：開發上建議秉持模組化精神，利於產品迭代，並有助後續維運服務→ISO 22166
		- 🧑零組件商：講Intel RealSense與ROS的小故事，建議若要面向機器人市場，有對應driver會較好
		- 📂兩大系統：NVIDIA(Jetson)、Intel(x86)
 
3.感知工程規範
ROS 2 engineering standards for sensor fusion
	- 從兩個面向提供寫程式上的參考，以利後續與其他演算法串接
	- 一是ROS Enhancement Proposal (REP)，將用開發順序(確認開發平台版本→挑設備看規格→依應用需求寫設定值)依序說明
		- REP-2000  ROS 2 Releases
		- REP-117	LaserScan
		  REP-118	Depth Images
		- REP-103	Units & Coordinate Conventions
		  REP-105	Coordinate Frames
		- 從上述找出跟感知有關的，例如右手座標系(X→forward. Y→left. Z→up)，或座標定義(  map / odom / base_link / sensor frame )
	- 二是從國際主流產品ROS package內，彙整出各家node 類別(從launch 回推)
第二章
對接 Integration
 
1.主流演算法
What Algorithms Do Robots Use?
	- 【畫圖】以技術四象限進行應用分類
	  
	- 依此將演算法分為兩大類
		- 第Ⅱ、Ⅲ象限：用較低成本的方式做到建圖、定位及避障，常見感測器配置：〔2D光達 or RGB相機〕+ IMU + 里程器(通常在馬達裡) + 超音波
		- 第Ⅰ、Ⅳ象限：注重點雲圖品質，強調即時運算與時間同步，常見感測器配置：〔3D光達 or 深度相機〕+ IMU + GNSS RTK + 里程器(通常在馬達裡) + 毫米波
	- 點出若要接演算法，需要哪些message ，並用拓譜圖(Topic Graph)呈現資料流動
 
2.資料傳輸機制
How Data Transmission Works?
	- 【畫圖】感測器傳輸決策樹 decision tree
	- 呈現三個面向
		- 通訊協定 Communication Protocols
			- IoT / AIoT 資料交換→MQTT
			- Web / Cloud 系統整合→REST API
			- 遠端監控(視訊)→RTSP
			- 推流至雲端伺服器(如 YouTube)→RTMP
			- 串 IP 攝影機 (Security Robot)→ONVIF
			- 工控世界→Modbus TCP、CAN Bus
			- 車用世界→SOME/IP
		- 實體介面 Hardware Interfaces
			- 高頻寬需求，例如3D光達或高解析影像→MIPI、GMSL2、Ethernet、USB 3.0
			- 中低頻寬需求，例如2D光達或控制訊號→UART、RS-232、RS-485、USB
		- 網路傳輸 Network Communication
			- 聯網需求→TCP/IP、UDP
 
3.融合前處理
Sensor Fusion Preprocessing
	- ROS 2 上常見的處理方式，主要談兩個面向
		- 空間校準 Extrinsic Calibration
			- TF Tree，描述不同座標系間的轉換關係，可能以 IMU 為建議基準
			- URDF，描述機器人結構與感測器配置，可用於 RViz 顯示與模型建立
		- 時間同步 Time Synchronization
			- 常見加掛硬體(接線)處理，例如時間同步卡
			- ROS有 message_filters 套件可處理ExactTime 與 ApproximateTime；不確定是否有其它在處理器上用軟體同步的方法
			- 網路時間同步的必要協定→NTP(雲端聯網/軟體)、PTP(地端區網/硬體)
		- 除了公版sensor_msgs，可能有針對特定演算法自定義 .msg的需求
第三章
優化 Optimization
 
1.淺談校正實務
Is the Data Really Accurate?
	- 提實務面會遇到的問題：例如設定相機輸出 30 FPS，怎麼知道相機真的輸出 30 FPS，出來的數據是真的嗎？以及如果真的不準(掉幀、延遲或阻塞)，那要怎麼處理？
	- 不確定是否有足夠資料寫→可能談移動上的精準校正
 
2.進入模擬環境
Robotics Simulation
	- 用開發成本(費用&人力)降低來談，在機器人硬體還沒做出來前，建議先進入模擬程式來改軟體或訓練資料
	- 使用 Gazebo 或 NVIDIA Isaac Sim 進行演算法在不同情境下的模擬測試，特別是極端天氣(霧、雨、強光)，共同議題是要轉成3D模型格式
 
3.邊緣運算優化
Edge AI Acceleration
	- 若要執行深度學習模型(例如 YOLO)，談如何在NVIDIA和Intel兩大系統框架下，利用其套件加速推論
		- ROS 2 + NVIDIA Jetson + TensorRT
		- ROS 2 + Intel D435 / x86 + OpenVINO
版權頁
 
 