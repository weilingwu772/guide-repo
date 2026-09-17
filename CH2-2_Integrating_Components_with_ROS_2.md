# ROS 2 零組件整合 (Integrating Components with ROS 2)

繼硬體介面與通訊協定之後，下一步則需確認該零組件是否已有原廠或社群提供、可支援 ROS 2 的套件（Package），若已有可用的 ROS 2 套件，就可直接安裝並依其介面進行整合，若無現成套件，則需依實際應用需求自行建立 ROS 2 套件，以將資料輸入、控制方式或通訊流程封裝為 ROS 2 可使用的節點；由於完整套件需遵循特定的目錄結構與設定規範，才能由 colcon 工具正確辨識、編譯與安裝，因此本節將以 Python 開發的 RTSP 影像串流應用為例，逐步說明 ROS 2 套件的建立流程。


以下步驟，是單純在ROS2流程，不涉及pyhton或c++的程式碼開發

### 步驟一：建立放置套件目錄


先建立工作空間以利之後形成的ROS2套件放入
假設未來需要建立的套件名稱為 `ipcam`，之後要將 `ipcam` 套件放入 `src` 目錄：

```bash
# 建立工作空間結構
mkdir -p ~/ros2_ws/src

# 將 ipcam 套件專案放置於 ~/ros2_ws/src/ipcam
```

確認節點檔案位於 `~/ros2_ws/src/ipcam/ipcam/rtsp_camera_node.py`，並給予執行權限：

`~/ros2_ws/src/ipcam/ipcam/rtsp_camera_node.py` 要另在python寫程式碼

```bash
chmod +x ~/ros2_ws/src/ipcam/ipcam/rtsp_camera_node.py
```

### 步驟二：設定進入點

先定義好進入路徑

建立 `~/ros2_ws/src/ipcam/setup.py`，提醒要確認 `entry_points` 欄位已設定 `console_scripts`，已讓 ROS 2 能找到執行檔：

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

切換至工作空間根目錄並進行指定套件編譯，輸入以下指令在終端機

```bash
cd ~/ros2_ws
colcon build --packages-select ipcam
```

若以 RTSP 為例，標準 ROS 2 工作空間架構應該長得像下面：
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

### 步驟四：載入環境變數與啟動節點

再輸入以下指令，以載入編譯後的環境設定並啟動 RTSP 攝影機節點(透過 run 生成節點)：

```bash
cd ~/ros2_ws
source install/setup.bash
ros2 run ipcam rtsp_camera_node
```

### 步驟五：啟動 RViz2 進行視覺化監控

開啟另一個新的終端機視窗，載入環境設定後啟動 RViz2：

```bash
source install/setup.bash
rviz2
```

進入ROS2，所有Topic都可以在RViz2上呈現，即確認完成