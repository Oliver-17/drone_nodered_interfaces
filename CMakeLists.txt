cmake_minimum_required(VERSION 3.8)
project(drone_nodered_interfaces)

# 只放服務／訊息格式，沒有任何程式。
# 獨立成一個套件的原因：跑 rosbridge 的電腦必須認得這些格式才能解開 JSON。
# 如果格式放在 drone_nodered 裡，那台電腦就得連 px4_msgs 一起裝。
find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "srv/Takeoff.srv"
  "srv/SetAltitude.srv"
  "srv/Patrol.srv"
  "srv/Track.srv"
  "srv/GoTo.srv"
)

ament_export_dependencies(rosidl_default_runtime)
ament_package()
