# ROS 2 零組件整合 (Integrating Components with ROS 2)

繼硬體介面與通訊協定之後，下一步則需確認該零組件是否已有原廠或社群提供、可支援 ROS 2 的套件（Package），若已有可用的 ROS 2 套件，就可直接安裝並依其介面進行整合，若無現成套件，則需依實際應用需求自行建立 ROS 2 套件，以將資料輸入、控制方式或通訊流程封裝為 ROS 2 可使用的節點；由於完整套件需遵循特定的目錄結構與設定規範，才能由 colcon 工具正確辨識、編譯與安裝，因此本節將以 Python 開發的 RTSP 影像串流應用為例，逐步說明 ROS 2 套件的建立流程。

### 步驟一：建立放置套件目錄

確定工作空間架構，標準 ROS 2 工作空間架構：
```text
~/ros2_ws/src/ipcam/
├── ipcam/                     # Python 模組資料夾（名稱需與套件同名）
│   ├── __init__.py            # 標記該目錄為可匯入的 Python 模組
│   └── rtsp_camera_node.py    # RTSP 攝影機主節點程式碼
├── resource/
│   └── ipcam                  # ament 索引標記檔案
├── package.xml                # ROS 2 套件元資料定義檔（依賴關係、維護者資訊）
├── setup.py                   # Python 建置與命令進入點（console_scripts）設定檔
└── setup.cfg                   # 指定可執行檔安裝路徑設定
```

若為全新套件，可先建立工作空間並將 `ipcam` 套件放入 `src` 目錄：

```bash
# 建立工作空間結構
mkdir -p ~/ros2_ws/src

# 將 ipcam 套件專案放置於 ~/ros2_ws/src/ipcam
```

確認節點檔案位於 `~/ros2_ws/src/ipcam/ipcam/rtsp_camera_node.py`，並給予執行權限：

```bash
chmod +x ~/ros2_ws/src/ipcam/ipcam/rtsp_camera_node.py
```

### 步驟二：設定進入點

定義 console_scripts，開啟 `~/ros2_ws/src/ipcam/setup.py`，確認 `entry_points` 欄位已設定 `console_scripts`，讓 ROS 2 能找到執行檔：

```python
from setuptools import setup

package_name = 'ipcam'

setup(
    name=package_name,
    version='0.0.0',
    packages=[package_name],
    data_files=[
        ('share/ament_index/resource_index/packages', ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
    ],
    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='weipo',
    maintainer_email='weipo@todo.todo',
    description='IP Camera ROS2 package',
    license='TODO: License declaration',
    tests_require=['pytest'],
    entry_points={
        'console_scripts': [
            # 格式：'可執行指令名稱 = 套件名稱.腳本檔名:主函數名稱'
            'rtsp_camera_node = ipcam.rtsp_camera_node:main',
        ],
    },
)
```

### 步驟三：使用 colcon 編譯套件

切換至工作空間根目錄並進行指定套件編譯：

```bash
cd ~/ros2_ws
colcon build --packages-select ipcam
```

> **提示**：若開發過程中僅修改 Python 程式碼，可改用 `colcon build --symlink-install --packages-select ipcam`，避免每次修改都需要重新 build。

### 步驟四：載入環境變數與啟動節點

開啟終端機，載入編譯後的環境設定並啟動 RTSP 攝影機節點：

```bash
cd ~/ros2_ws
source install/setup.bash
ros2 run ipcam rtsp_camera_node
```

> **預設串流來源位址**：`rtsp://admin:a-12345678@192.168.60.97:554/unicast/c1/s1/live`

### 步驟五：啟動 RViz2 進行視覺化監控

影像訂閱確認，開啟另一個新的終端機視窗，載入環境設定後啟動 RViz2：

```bash
source install/setup.bash
rviz2
```

**RViz2 畫面設定步驟：**
1. 將左側全域設定的 `Fixed Frame` 改為 `map` 或 `camera_link`。
2. 點擊左下角的 `Add` 按鈕，選擇 `Image` 顯示模組。
3. 將新增的 `Image` 模組下的 Topic 設定訂閱至 `/camera/ipc/image_raw`，即可看到 RTSP 即時傳輸畫面。