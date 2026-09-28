1. Убедиться, что камера работает на хосте:
## Порядок запуска ROS2-драйвера камеры в Docker-контейнере

1. Проверить работу камеры на хосте, вне контейнера

```bash
gst-launch-1.0 -q \
  nvarguscamerasrc num-buffers=1 sensor-id=0 \
  ! 'video/x-raw(memory:NVMM),width=1280,height=720,format=NV12,framerate=30/1' \
  ! nvvidconv \
  ! 'video/x-raw,format=BGRx' \
  ! fakesink
```

Код возврата должен быть `0`:

```bash
echo $?
```

2. Пересоздать контейнер jetbot_educational, если он уже есть на роботе. Обычного `restart` недостаточно, потому что новые mounts, environment и группы применяются только при создании:

```bash
cd ~/IntelligentRoboticsOld/jetbot_ros2_mipt
docker compose config -q
docker compose up -d --force-recreate
```

**Команды ниже выполняются внутри контейнера**

3. Проверить NVIDIA-плагины внутри контейнера:

```bash
docker exec jetbot_ros2_mipt-jetbot_educational-1 bash -lc '
id
test -S /tmp/argus_socket
gst-inspect-1.0 nvarguscamerasrc >/dev/null
gst-inspect-1.0 nvvidconv >/dev/null
echo CAMERA_RUNTIME_OK
'
```

В выводе `id` должна быть группа `video`, а в конце — `CAMERA_RUNTIME_OK`.

4. Пересобрать workspace (достаточно только пакет камеры):

```bash
docker exec jetbot_ros2_mipt-jetbot_educational-1 bash -lc '
source /opt/ros/humble/install/setup.bash
cd /home/app/ros2_ws
colcon build \
  --symlink-install \
  --packages-select pi_camera \
  --event-handlers console_direct+
'
```

5. Запустить драйвер:

```bash
docker exec -it jetbot_ros2_mipt-jetbot_educational-1 bash
source /opt/ros/humble/install/setup.bash
source /home/app/ros2_ws/install/setup.bash
ros2 run pi_camera publish_raw
```

7. В другом терминале проверить публикацию:

```bash
docker exec -it jetbot_ros2_mipt-jetbot_educational-1 bash -lc '
source /opt/ros/humble/install/setup.bash
source /home/app/ros2_ws/install/setup.bash
ros2 topic hz /camera/image/raw
'
```

Ожидается примерно 28–30 FPS.

Команду с добавлением `mode = "csv"` в `/etc/nvidia-container-runtime/config.toml` на остальных одинаковых роботах повторять не обязательно. На этой версии JetPack/runtime фактически сработали `NVIDIA_VISIBLE_DEVICES=all` из Compose, добавление группы `video`, Argus socket и пересоздание контейнера.
