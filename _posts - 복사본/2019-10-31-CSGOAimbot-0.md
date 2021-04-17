---
title: "Make CS:GO Aimbot_0x00"
date: 2019-10-31
categories:
- Machine Learning
tags:
- Machine Learning
---

# # Introduce
에임봇을 알아보던 중 이미지서칭 기반의 머신러닝 에임핵이 있다는 것을 알게 되었으며  
자료를 찾아 보다 영상을 보고 만들어 보자는 생각이 들었습니다.  
머신러닝 에임핵을 만들면서 가장 처음에 삽질했던 설치 방법에 대하여 해당 포스팅에서 정리하고  
차후에 이미지 서칭 기반의 에임핵을 만들어 결과물을 포스팅 할 예정입니다.  

## ## Testbed Spec
CPU: i7-8750H 2.2Ghz  
GPU: GTX1070 8G  
RAM: 16GB  
OS: Windows 10  

## ## 에임핵 영상
<iframe width="560" height="315" src="https://www.youtube.com/embed/vd7NlRYPkZw" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## ## Machine Learning
위키백과에선  
> 기계 학습(機械學習) 또는 머신 러닝(영어: machine learning)은 인공 지능의 한 분야로, 컴퓨터가 학습할 수 있도록 하는 알고리즘과 기술을 개발하는 분야를 말한다. 가령, 기계 학습을 통해서 수신한 이메일이 스팸인지 아닌지를 구분할 수 있도록 훈련할 수 있다.

라고 정의되어 있습니다.  
이 포스팅 및 향후의 에임봇 포스팅에서는 Tensorflow 라이브러리와 Tensorflow에서 지원하는 객채 탐지  
(Object Detection) 알고리즘을 이용하여 구현하였습니다.  

## ## Aimbot
Aimbot이란 슈팅 게임 장르에서 적에게 자동으로 조준을 해주는 봇을 의미하며 TriggerBot과 같이 사용되어 AimHack이라고도 자주 불립니다.  
슈팅 게임 장르를 좋아하는 저로써는 게임을 망치는데 아주 큰 기여를 하는 것 중 하나가 AimHack이라고 생각합니다.

## ## Tensorflow
구글에서 만든 머신러닝 시스템으로, 간단하게 개인 데스크탑에서 머신러닝을 할 수 있게 해주는 라이브러리입니다.  
CPU, GPU 다 사용하여 머신러닝이 가능하며 C++, GO, Java 언어 등등을 지원 한다고 하는데저는 Python을 이용하여 라이브러리를 사용했습니다.  

## ## Installation
1. Nvidia 드라이버 & CUDA 10.0 버전을 설치합니다.
2. Python 3.x 버전을 설치합니다.
3. Cuda와 동일버전인 Cuddn을 다운로드합니다. (Cuddn 10.0)  
[[Download Link]](https://developer.nvidia.com/cudnn)
4. tensorflow model을 다운로드합니다.  
[[Download Link]](https://github.com/tensorflow/models)
5. protoc 3.4.0 x32 다운로드합니다.  
[[Download Link]](https://github.com/protocolbuffers/protobuf/releases/tag/v3.4.0)
6. Tensorflow Model & Protoc & CUDDN를 C:\ 경로에 압축을 풀어줍니다.
압축 푼 폴더를 쓰기 편하게 이동해 줍니다.
```
move C:\models-master\models-master C:\tensorflow
move C:\cudnn-10.0-windows10-x64-v7.6.3.30\cuda C:\cuddn
```
7. Tensorflow 모델 및 cuddn의 편한 사용을 위하여 환경변수에 추가합니다.
```
setx PYTHONPATH "C:\tensorflow\research;C:\tensorflow\research\slim' -m"
setx Path "%Path%;C:\cuddn\bin" -m
```
8. Tensorflow와 Object-detection에 필요한 python 모듈을 설치합니다
(cython, contextlib2 , lxml , jupyter, matplotlib, pillow)  

9. Object-detection 설치를 위해 proto파일을 python 파일로 컴파일 후 설치 해줍니다.
```
cd C:\Tensorflow\research
"C:\protoc-3.4.0-win32\bin\protoc.exe" object_detection\protos\*.proto --python_out
python ./setup.py install
```
10. Tensorflow GPU를 설치합니다.
```
pip install tensorflow-gpu
```
11. Tensorflow를 python에 import 하여 정상적으로 모듈이 설치되었는지 확인합니다.
```
python
import tensorflow as tf
```

- future warning 이 발생 할 경우 numpy를 1.17 이하 버전으로 설치하시면 됩니다.
```
pip install numpy<=1.17
```
# #Relation Post
1. [[Make CS:GO Aimbot_0x01]](https://holi4m.github.io/machine%20learning/2019/11/23/CSGOAimbot-1/)

# #Reference
1. [[Tensorflow Object Detection CS:GO Aimbot for Python Lessons]](https://www.youtube.com/watch?v=HX2yXajg8Ts&list=PLbMO9c_jUD46x-d4pIj_STGK6CWWkw0Vv)  
2. [[Wikipedia]](https://en.wikipedia.org/)


