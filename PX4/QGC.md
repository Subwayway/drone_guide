---
layout: post
title: QGroundControl
category: PX4
---
PX4 오토파일럿은 GCS로 QGroundControl을 사용함.

시리얼 포트에 접근하기 위해서는 다음의 작업이 필요함.
```bash
sudo usermod -a -G dialout $USER
sudo apt-get remove modemmanager -y
sudo apt install gstreamer1.0-plugins-bad gstreamer1.0-libav gstreamer1.0-gl -y
```
QGroundControl을 다운받음.\
https://s3-us-west-2.amazonaws.com/qgroundcontrol/latest/QGroundControl.AppImage

다음 명령을 통해 실행.
```bash
chmod +x ./QGroundControl.AppImage
./QGroundControl.AppImage  (or double click)
```
<p align="center"><img src="https://user-images.githubusercontent.com/48232366/113685896-ac839c80-9701-11eb-8876-012b93889f47.PNG" width="80%"></p>


참고:https://docs.qgroundcontrol.com/master/en/getting_started/download_and_install.html
