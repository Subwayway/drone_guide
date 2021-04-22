---
layout: post
title: 개발환경 구축(Ubuntu)
category: PX4
---
PX4의 개발환경은 리눅스 환경을 권장하고 있다.
<p align="center"><img src="https://user-images.githubusercontent.com/48232366/113682212-de92ff80-96fd-11eb-90d8-b02c54465a7a.PNG" width="80%"></p>

Ubuntu Linux LTS 18.04를 virtual box에 설치함.

Ubuntu에서의 PX4 개발환경은 bash script를 통해 간편하게 구성할 수 있도록 제공되고 있음.
```bash
wget https://github.com/PX4/PX4-Autopilot/blob/master/Tools/setup/ubuntu.sh
./ubuntu.sh
```

스크립트의 실행권한이 필요하다면 권한을 변경해줌.
```bash
chmod 755 ubuntu.sh
```

PX4의 펌웨어 코드를 clone함.
```bash
git clone https://github.com/PX4/PX4-Autopilot.git --recursive
```

빌드를 테스트 하기위해 다음 명령을 입력해 jmavsim 시뮬레이션을 통해 정상적으로
동작하는지 확인한다.
```bash
make px4_sitl jmavsim
```
<p align="center"><img src="https://user-images.githubusercontent.com/48232366/113683936-b2787e00-96ff-11eb-9b90-79f14ec7ad9f.png" width="80%"></p>

픽스호크에 맞는 펌웨어를 빌드하려면 다음 명령으로 빌드함.
<p align="center"><img src="https://user-images.githubusercontent.com/48232366/113684346-19963280-9700-11eb-8558-8f1d0aa3e958.PNG" width="80%"></p>

빌드한 펌웨어의 업로드는 다음과같다.
```bash
make px4_fmu-v3_default upload
```
<p align="center"><img src="https://user-images.githubusercontent.com/48232366/113684923-ad67fe80-9700-11eb-9147-c4125494cca9.PNG" width="80%"></p>

### Troubleshooting
이미 픽스호크에 펌웨어가 올라가 있어 usb케이블 연결시 부팅이 진행되어 업로드시의 초기화 타이밍과 맞지 않아 오류가 발생할 수 있다.

픽스호크의 경우 led가 깜빡이는 부팅과정에 VirtualBox의 usb연결을 수동으로 연결시켜줘야 펌웨어 업로드 타이밍을 잡을수 있다.

참고:https://docs.px4.io/master/ko/dev_setup/building_px4.html
