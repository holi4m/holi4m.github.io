---
title: VirtualKD-Redux Installation & Kernel Debugging
date: 2020-03-31
categories:
- Kernel
tags:
- Kernel
- VirtualKD
---

# # Introduce
기존에 커널 디버깅 시 VirtualKD를 애용하고 있었는데 집 환경에서 VirtualKD에서 커널 디버깅이 되지 않아 찾게 되었습니다.  

# # VirtualKD-Redux
VirtualKD의 개선판으로 기존 개발자가 아닌 다른 개발자가 수정 및 보완하여 올린 것으로 생각됩니다.

## ## VirtualKD-Redux 설치
1. [[Github]](https://github.com/4d61726b/VirtualKD-Redux/releases)에서 최신 버전을 다운로드 받습니다.  
2. 압축을 풀고 vmmon64.exe를 실행합니다.  
3. 디버깅 할 Guest OS에 맞춰 tartget32 또는 target64를 Guest OS로 복사하여 설치합니다.  
4. 설치 및 재부팅 후 OS 선택 화면이 나오면 VirtualKD에서 TestSigning으로 OS를 시작합니다.  
5. 즐겁게 커널 디버깅을 합니다.

## ## VirtualKD-Redux를 설치하게 된 이유
기존에 사용하던 환경(Windows 10 1809)에서는 정상실행이 되었으나 새로운 환경(Windows 10 1909)에서 VirtualKD를 이용해 정상적으로 커널 디버깅이 되지 않아 찾게 되었습니다.  
Windows 10 build 1909로 업데이트 하신 분들은 해당 VirtualKD-Redux를 사용하시면 정상적으로 다시 커널 디버깅이 가능합니다.  

# # Reference
1. [[VirtualKD-Redux Github]](https://github.com/4d61726b/VirtualKD-Redux)



