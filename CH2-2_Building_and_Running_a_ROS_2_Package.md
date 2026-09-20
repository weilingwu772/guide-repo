# ROS 2 套件建立與執行（Building and Running a ROS 2 Package）

繼硬體介面與通訊協定之後，下一步便是確認該零組件是否已有原廠或第三方提供的 ROS 2 套件；這裡先補充說明，原廠驅動程式與 ROS 2 套件並不相同：驅動程式主要負責讓作業系統能夠辨識硬體、存取資料或控制設備，但不一定具備 ROS 2 的通訊介面；ROS 2 套件則是 ROS 2 用來封裝與管理軟體功能的基本單位，可進一步將既有驅動程式、SDK 或 Library 所取得的資料與控制功能，封裝成 ROS 2 節點，並透過 Topic、Service、Action 等機制與其他節點進行溝通。

如果原廠或第三方開源社群已有提供適用的 ROS 2 套件，開發者通常可以直接安裝，並依照套件既有的介面進行系統整合；若硬體僅有原廠驅動程式、SDK 或 Library，而無對應的 ROS 2 套件，則需要自行開發可在 ROS 2 開發框架內使用的節點；不過，完成節點程式也並不完全表示 ROS 2 就能直接執行它，一個完整的套件還需遵循特定的目錄結構與設定規範，才能由 colcon 工具正確辨識、建置與安裝，因此本節在不討論 Python 或 C++ 程式碼如何開發的前提下，聚焦在節點程式完成之後，說明如何將其整理成能讓 ROS 2 執行的套件。

以下步驟以 Python 開發的 RTSP 網路攝影機應用為例，假設已經完成名為`rtsp_camera_node.py`的節點程式，以及`setup.py`安裝設定檔，接著逐步將它建立成名為`ipcam`的 ROS 2 套件。

---

## 步驟一：建立工作空間與套件目錄

首要之務是建立工作空間（Workspace），用來集中管理自行開發的 ROS 2 套件。例如先建立一個名為`ros2_ws`的工作空間，並在其中建`src`目錄，然後本例假設要建立的套件名稱為`ipcam`，接著將準備好的套件放置於目錄中；另外，對於 Python 型態的 ROS 2 套件而言，`ipcam` 套件內還會有另一個同名的 Python 模組目錄，因此節點程式`rtsp_camera_node.py`會位於`~/ros2_ws/src/ipcam/ipcam/rtsp_camera_node.py`，其工作空間與套件目錄結構如下：

```text
~/ros2_ws/
└── src/
    └── ipcam/
        ├── ipcam/
        │   ├── __init__.py
        │   └── rtsp_camera_node.py
        ├── resource/
        │   └── ipcam
        ├── package.xml
        ├── setup.py
        └── setup.cfg
```

## 步驟二：設定進入點

接下來則要設定程式的進入點，讓 ROS 2 知道如何找到並執行這支程式，因此，Python 型態的 ROS 2 套件需要透過`setup.py`設定檔，描述套件名稱、安裝內容以及可執行程式的進入點（`entry_points`欄位設定 `console_scripts`）：

```python
entry_points={
    'console_scripts': [
        'rtsp_camera_node = ipcam.rtsp_camera_node:main',
    ],
},
```

如此一來，後續才可以直接使用`ros2 run`來啟動節點，而不必再指定 Python 檔案路徑；此外，一個完整套件還會包含 `package.xml`、`setup.cfg`、`resource` 等檔案與目錄，分別負責描述套件資訊、安裝位置及提供 ROS 2 套件索引使用。

## 步驟三：使用 colcon 建置套件

當套件目錄與相關設定準備完成後，下一步便是進行建置（Build），ROS 2 通常使用`colcon`作為建置工具，首先切換至工作空間根目錄，並輸入以下指令在終端機以指定建置 `ipcam` 套件（代表此次只建置 `ipcam` 套件，而不是將工作空間中的所有套件全部重新建置）：

```bash
cd ~/ros2_ws
colcon build --packages-select ipcam
```

如果建置成功，原本只有`src`的工作空間根目錄，通常還會產生：

```text
~/ros2_ws/
├── build/
├── install/
├── log/
└── src/
```

## 步驟四：載入環境設定並啟動節點

完成建置之後，還不能直接假設目前的終端機已經知道 `ipcam` 套件存在，需要再輸入以下指令，用以載入`colcon`後產生的環境設定，即可使用`ros2 run`來啟動節點：

```bash
cd ~/ros2_ws
source install/setup.bash
ros2 run ipcam rtsp_camera_node
```

此時 ROS 2 會從`ipcam`套件中尋找名為`rtsp_camera_node`的執行入口，再依照先前`setup.py`中的設定找到`ipcam.rtsp_camera_node`模組，最後執行其中的`main()`函數；當該程式建立 ROS 2 Node 並開始運作後，才會成為 ROS 2 系統中正在執行的節點。同時，以本次假設的 RTSP 網路攝影機應用為例，若該攝影機節點的程式邏輯是將接收到的影像轉換成 ROS 2 Image Message 並發布至 Topic，此時其他 ROS 2 節點便可以訂閱這個 Topic，進一步使用攝影機影像。

## 步驟五：使用 RViz2 確認資料是否正常發布

如同前述章節（CH 1-2）的機器人感知系統五步驟，節點成功啟動後，最後便可以使用 RViz2 確認它是否有正常發布資料，這時需開啟另一個終端機，切換至相同工作空間並載入環境設定，接著啟動 RViz2，透過加入 Image 顯示項目，以及選擇攝影機節點所發布的 Topic 等操作流程，若能正常看到攝影機畫面，即代表這條資料傳輸流程已經成功建立，也可確認此套件中的節點已能在 ROS 2 環境中正常執行：

```bash
cd ~/ros2_ws
source install/setup.bash
rviz2
```

---

完成上述步驟後，可以發現自行建立 ROS 2 套件的核心目的，其實就是建立一套 ROS 2 能夠辨識及管理的標準結構；因此，當一個零組件沒有原廠或第三方提供的 ROS 2 套件時，並不代表它無法進入 ROS 2，只是需要開發者自行透過 Python、C++ 或其他方式編寫節點程式，再依照 ROS 2 規範將程式封裝成套件，就能讓原本各自獨立的硬體進入 ROS 2 的開發框架。