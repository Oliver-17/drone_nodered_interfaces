# drone_nodered_interfaces — 無人機服務的「格式定義」

只有三個 `.srv` 檔，**沒有任何程式**。它定義的是「呼叫無人機服務時，要傳什麼、會回什麼」。

> 這個 repo 本身就是一個 ROS 2 package，不是 workspace。clone 進 `src/` 底下即可。

---

## ⚠️ 這是三個套件中的一個

| 套件 | 角色 | GitHub |
|---|---|---|
| **drone_nodered_interfaces**（本包） | 服務格式（`.srv`）。很小，沒有程式碼 | https://github.com/Oliver-17/drone_nodered_interfaces |
| drone_nodered | 無人機專用：控制節點、Node-RED 流程、模擬、systemd 服務 | https://github.com/Oliver-17/drone_nodered |
| nodered_ros2 | 通用橋接：rosbridge + 兩個 Node-RED 積木 | https://github.com/Oliver-17/nodered_ros2 |

**相依方向**：

```
drone_nodered ──需要──> drone_nodered_interfaces（本包）
      └───────需要──> nodered_ros2

drone_nodered_interfaces ──不需要任何一個──>
```

本包不依賴另外兩個，但**另外兩個要一起裝才會有完整功能**——單獨裝這包只會得到格式定義，
沒有東西會去實作這些服務。

---

## 快速導覽

| 你想知道 | 看哪一節 |
|---|---|
| **為什麼要獨立成一個套件** | [1 為什麼獨立](#1-為什麼獨立) |
| 有哪些格式、欄位是什麼 | [2 三個服務格式](#2-三個服務格式) |
| 我要新增一個服務 | [3 新增格式的步驟](#3-新增格式的步驟) |
| 檔案在哪 | [4 檔案結構](#4-檔案結構) |
| 有什麼坑 | [5 踩過的雷](#5-踩過的雷) |

---

## 1 為什麼獨立

因為**跑 rosbridge 的那台電腦，必須認得這些格式**，才解得開 Node-RED 送來的 JSON。

如果格式放在 `drone_nodered` 裡面，那台電腦就得連 `px4_msgs`（幾百個訊息）和整包無人機
程式一起裝。獨立成一個只有三個文字檔的套件之後，裝起來幾秒就好。

實機時這個差別更明顯：Node-RED 在筆電、rosbridge 和控制節點在樹莓派，
兩邊都要認得格式，但筆電根本不需要無人機的程式碼。

> 同樣的道理，不需要參數的動作（盤旋、降落、狀態查詢）**刻意不放在這裡**，
> 直接用 ROS 內建的 `std_srvs/srv/Trigger` 就好。這包只放「內建格式表達不了」的東西。

---

## 2 三個服務格式

三個的回應欄位都一樣：

```
bool success      # 指令是否被接受（不代表動作已經完成）
string message    # 給人看的說明，失敗時寫原因
```

### 2.1 `Takeoff.srv` — 起飛

```
float32 altitude    # 起飛高度（離地公尺）
```

為什麼不用 `Trigger`：`Trigger` 的請求是空的，帶不了高度。
高度如果改放節點參數，就得「先設參數、再起飛」兩步，中間有人改了參數就會飛錯高度。

### 2.2 `SetAltitude.srv` — 空中調整高度

```
float32 altitude    # 離「起飛點」的高度（公尺），不是海拔
```

欄位和 `Takeoff` 一模一樣，但**刻意分成兩個型別**：型別名稱就是文件。
在 Node-RED 或 `ros2 service call` 看到 `SetAltitude` 就知道這不是起飛。

### 2.3 `Patrol.srv` — 巡航

```
float32[] x         # 每個點的 x（公尺，離起飛點；北為正）
float32[] y         # 每個點的 y（東為正）
float32 altitude    # 巡航高度。0 = 維持目前高度
float32 speed       # 水平速度（m/s）。0 = 用 PX4 預設
int32 loops         # 繞幾圈
```

為什麼用兩個平行陣列而不是 `geometry_msgs/Point[]`：從 Node-RED 傳 JSON 時
`{"x":[10,10,0,0],"y":[0,10,10,0]}` 比 `[{"x":10,"y":0,"z":0},…]` 好讀好改，
而且這裡的 `z` 本來就不是「每個點各自的高度」。

---

## 3 新增格式的步驟

```bash
# 1) 新增 srv/MyThing.srv
# 2) 在 CMakeLists.txt 的 rosidl_generate_interfaces() 裡加上那一行
# 3) 編譯
cd ~/ros2_ws && colcon build --packages-select drone_nodered_interfaces
source ~/ros2_ws/install/setup.bash

# 4) 確認格式真的產生出來了
ros2 interface show drone_nodered_interfaces/srv/MyThing
```

**第 5 步最容易忘**：rosbridge 和使用這個型別的節點都要**重新啟動**，
它們是在啟動時載入型別的。用 `drone_nodered` 的 systemd 服務時：

```bash
systemctl --user restart drone-nodered
```

---

## 4 檔案結構

```
drone_nodered_interfaces/
├── srv/
│   ├── Takeoff.srv
│   ├── SetAltitude.srv
│   └── Patrol.srv
├── CMakeLists.txt      用 rosidl_generate_interfaces 產生型別
└── package.xml         member_of_group rosidl_interface_packages
```

---

## 5 踩過的雷

| 現象 | 原因 | 解法 |
|---|---|---|
| 節點啟動時 `ModuleNotFoundError: drone_nodered_interfaces` | 那個終端機是在編譯之前開的 | 關掉重開，或 `source ~/ros2_ws/install/setup.bash` |
| 服務回「Unable to import …」 | **rosbridge 不認得這個型別**（它自己也要 import 才解得開 JSON） | rosbridge 要在 source 過的環境下啟動；改格式後要重啟它 |
| 拒絕訊息印出 `0.20000000298023224` | `float32` 存不下剛好 0.2 | 顯示時用 `:.2f` 格式化 |
| 新增了 `.srv` 卻找不到 | 忘了加進 `CMakeLists.txt` | `rosidl_generate_interfaces()` 裡要列出每一個檔案 |
