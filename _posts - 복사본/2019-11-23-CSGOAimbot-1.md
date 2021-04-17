---
title: "Make CS:GO Aimbot_0x01"
date: 2019-11-23
categories:
- Machine Learning
tags:
- Machine Learning
---

# # Introduce
Tensorflow와 Object Detection을 이용하여 만든 Aimbot입니다.  

## ## Youtube
<iframe width="560" height="315" src="https://www.youtube.com/embed/gIVbY8Ni88U" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

왼쪽의 원본 영상을 토대로 캐릭터의 머리 이미지를 탐색하여 해당 좌표로 마우스를 이동, 클릭해주는 로직으로 구현되어 있습니다.  
제작하는 도중 업데이트가 되어 캐릭터가 추가되어 인식이 잘 안되는 구간이 있으며  
머신러닝 모델의 이미지 인식률이 높아 (오탐률도 다른 모델에 비해 높습니다.) 튀는 경우가 있습니다.  
테스트베드의 사양이 조금 더 좋고 좀 더 정확도와 탐지율이 높은 모델을 사용하면  
이전의 포스트에서 봤던 영상처럼 더 부드럽게 구현이 가능할 것으로 보입니다.

# #Relation Post
1. [[Make CS:GO Aimbot_0x00]](https://holi4m.github.io/machine%20learning/2019/10/31/CSGOAimbot-0/)
