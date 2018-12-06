# 자율 주행 차량에 관한 설명



### 개요

이것은 Siraj Raval의 유튜브 [비디오](https://youtu.be/yt015gM-ync) 에 관한 코드입니다. 시뮬레이터는 [여기](https://github.com/udacity/self-driving-car-sim)에서 찾을 수 있습니다. 

이 프로젝트의 목적은 딥러닝 신경망을 통해 인간의 운전 행동을 똑같이 따라하는 것입니다. 이러한 목적을 달성하기 위해, 간단한 자동차 시뮬레이터를 사용합니다. 훈련 단계에서 우리는 키보드를 사용하여 시뮬레이터 안의 차량을 조종합니다. 차량을 조종하는 동안 시뮬레이터는 훈련 이미지와 핸들의 각도를 측정합니다. 그리고 신경망을 훈련시키는 데 그 기록 데이터를 사용합니다. 훈련된 모델은 두 개의 트랙, 즉 훈련 트랙과 검증 트랙에서 테스트됩니다. 이어지는 애니메이션은 최종 모델이 훈련 트랙과 검증 트랙에서의 성과를 보여줍니다.

훈련 | 검증
------------|---------------
![training_img](./images/track_one.gif) | ![validation_img](./images/track_two.gif)

### 의존성

이 프로젝트는 **Python 3.5**과 이하의 Python 라이브러리의 설치를 요구합니다:

- [Keras](https://keras.io/)
- [NumPy](http://www.numpy.org/)
- [SciPy](https://www.scipy.org/)
- [TensorFlow](http://tensorflow.org)
- [Pandas](http://pandas.pydata.org/)
- [OpenCV](http://opencv.org/)
- [Matplotlib](http://matplotlib.org/) (Optional)
- [Jupyter](http://jupyter.org/) (Optional)

터미널 프롬프트에서 다음 커맨드를 입력하여 [OpenCV](http://opencv.org/)를 설치합니다. 이는 이미지 처리에 유용할 것입니다:

- `conda install -c https://conda.anaconda.org/menpo opencv3`

### 모델 사용법

이 저장소에서는 다음 커맨드를 사용하여 즉시 훈련된 모델을 사용할 수 있습니다.

- `python drive.py model.json`

## 구현

### 자료 수집

훈련 중에는 시뮬레이터가 10hz의 진동수로 데이터를 수집합니다. 또한, 매시간 좌, 우, 중앙 카메라의 이미지를 기록합니다. 다음 그림은 제가 훈련 중 수집한 예시입니다.

좌측| 중앙 | 우측
----|--------|-------
![left](./images/left.png) | ![center](./images/center.png) | ![right](./images/right.png)

수집된 데이터는 딥러닝 모델에 넣어지기 전에 전처리 과정을 거칩니다. 전처리 과정은 밑에서 설명되어 있습니다.

### 데이터 집합 통계
데이터 집합은 24108개의 이미지 (카메라 당 8036개의 이미지)로 구성되어 있습니다. 훈련 트랙은 다수의 얕은 회전과 직선 도로로 구성되어 있습니다. 따라서 대부분의 핸들 각도 기록은 0입니다. 그러므로 훈련 모델을 검증 트랙과 같이 아직 접하지 못한 트랙에 대해 일반화시키기 위해 이미지와 거기에 대응되는 핸들의 각도를 전처리하는 것이 필수적입니다.

다음으로 데이터 처리 파이프라인에 대해 설명드리도록 하겠습니다.

### 데이터 처리 파이프라인
다음은 우리의 데이터 처리 파이프라인을 보여주는 그림입니다.

<p align="center">
 <img src="./images/pipeline.png" width="550">
</p>

파이프라인의 첫 단계에서는 무작위로 기울임 (shear) 동작을 적용합니다. 그러나 0.9의 확률로만 이 과정을 적용하며 10퍼센트의 원래 이미지와 핸들 각도는 보존하여 훈련 트랙에서 차량을 조종할 수 있도록 합니다. 다음 그림은 샘플 이미지에 기울임 동작을 적용한 결과를 보여줍니다.

<p align="center">
 <img src="./images/sheared.png">
</p>

시뮬레이터에 의해 수집된 이미지는 모델을 세우는 데 직접적으로 도움이 되지 않는 상세 정보들을 많이 가지고 있습니다. 이런 상세 정보들이 차지하는 추가적인 공간은 더 많은 처리를 요구합니다. 따라서 이미지의 상단으로부터 10퍼센트의 공간 중 35퍼센트를 제거합니다. 이 과정은 절단 (crop) 단계에서 이루어집니다. 다음 그림은 절단 단계의 결과물을 보여줍니다.

<p align="center">
 <img src="./images/cropped.png">
</p>

데이터 처리 파이프라인의 다음 단계는 무작위로 뒤집는 (flip) 작업입니다. 이 단계에서 우리는 무작위로 (0.5의 확률로) 이미지를 뒤집습니다. 이 동작의 배경 아이디어는 훈련 트랙에서 좌회전 구간이 우회전 구간보다 더 많다는 점에서 착안하였습니다. 그러므로 우리의 모델을 일반화시키기 위해서는 이미지와 그에 따른 핸들 각도를 뒤집어야 합니다. 다음 그림은 뒤집기 동작이 적용된 결과입니다.

<p align="center">
 <img src="./images/flipped.png">
</p>

파이프라인의 마지막 단계에서는 훈련 시간을 줄이기 위해 이미지의 크기를 64x64로 재조정합니다. 크기가 재조정된 샘플 이미지는 아래 그림에서 볼 수 있습니다. 크기가 재조정된 이미지는 신경망으로 넣어지게 됩니다. 다음 그림은 크기 재조정 과정이 적용된 결과입니다.

<p align="center">
 <img src="./images/resized.png">
</p>

다음으로 신경망의 아키텍쳐에 관해 논해봅시다.

### 신경망 아키텍쳐 

우리의 CNN 아키텍쳐는 NVIDIA의 자율 주행 차량을 위한 end to end 학습 보고서에서 영감을 얻었습니다. 우리 모델이 NVIDIA의 것과 주로 다른 점은 훈련 시간을 줄이기 위해 Max-pooling층을 각 합성곱 층의 바로 다음에 넣었다는 것입니다. 우리의 신경망에 관해 더 자세한 정보는 다음 그림을 참고하시기 바랍니다.

<p align="center">
 <img src="./images/conv_architecture.png" height="600">
</p>

### 훈련
이미지의 절단과 크기 재조정을 거친 다음에도 훈련 데이터 집합은 매우 커서 주기억장치에 모두 들어가지 못합니다. 따라서 모델을 훈련시키는 데 Keras의 `fit_generator` API를 사용하였습니다.

우리는 두 개의 생성기를 만들었습니다.
* `train_gen = helper.generate_next_batch()`
* `validation_gen = helper.generate_next_batch()` 

훈련 집단 크기는 `train_gen`과 `validation_gen` 모두 64입니다. 우리는 각 주기별로 20032개의 이미지를 사용하였습니다. 이 이미지는 위에서 설명한 데이터 처리 파이프라인을 사용하여 그때그때 봐 가며 생성되었습니다. 더하여 우리는 6400개의 이미지 (역시 그때그때 봐 가며 생성)를 검증 단계에 사용하였습니다. 우리는 `1e-4`의 학습률로 `Adam`옵티마이저를 사용하였습니다. 마지막으로 (잘 모르겠습니다). 하지만 `8`은 훈련과 검증 트랙에서 모두 잘 작동하였습니다.

## 결과
프로젝트의 초기 단계에서는 제가 직접 만든 데이터 집합을 사용하였습니다. 데이터 집합은 작았고 노트북 키보드를 사용하여 주행하는 동안 기록된 것이었습니다. 그러나 이렇게 세워진 모델은 시뮬레이터에서 자동으로 주행하는 데 충분하지 않았습니다. 하지만 나중에는 Udacity에서 게시한 데이터 집합을 사용하였고, 그 데이터 집합으로 (증가된 데이터의 도움과 함께) 개발된 모델은 다음 비디오에서 볼 수 있듯 잘 작동하였습니다.

#### 훈련 트랙
[![training_track](https://img.youtube.com/vi/nSKA_SbiXYI/0.jpg)](https://www.youtube.com/watch?v=nSKA_SbiXYI)

#### 검증 트랙
[![validation_track](https://img.youtube.com/vi/ufoyhOf5RFw/0.jpg)](https://www.youtube.com/watch?v=ufoyhOf5RFw)

## 결론 및 향후 방향
이 프로젝트에서 우리는 자율 주행 차량의 맥락에서 회귀 문제를 다루었습니다. 첫 단계에서 우리는 적합한 신경망 아키텍쳐를 찾는 것과 우리의 데이터 집합으로 모델을 훈련시키는 데 집중했습니다. 평균 제곱 오차 (**MSE**)에 따르면 우리의 모델은 잘 작동하였습니다. 그러나 시뮬레이터에서 모델을 테스트할 때에는 예상만큼 수행하지 못했습니다. 따라서 평균 제곱 오차는 수행 능력을 측정하는 데 좋은 지표로 동작하지 못한다고 명백히 결론지었습니다.

그 다음 단계에서는 새로운 데이터 집합을 사용하기 시작했고 (실제로는 Udacity에서 게시한 데이터 집합) 추가적으로 우리는 최종 모델을 만드는 데 평균 제곱 오차에 전적으로 의존하지 않았습니다. 또한 우리는 상대적으로 작은 훈련 주기를 사용하였습니다 (namely `8` epochs). 데이터를 증가시키는 작업과 새로운 데이터 집합은 놀라울 정도로 잘 작동하였고 우리의 최종 모델은 두 트랙에서 최고의 수행 능력을 보였습니다. 

저는 확장성과 향후 방향에 관해 다음 내용을 강조하고 싶습니다.

* 실제 도로 조건에서 모델을 훈련시킵니다. 이를 위해서는 새로운 시뮬레이터를 찾아야 할 수도 있습니다.
* 다른 데이터 증가 기술을 적용하여 실험합니다.
* 우리가 자동차를 운전할 때 핸들을 돌리거나 브레이크를 밟는 행위 등은 그 순간의 운전 결정에만을 기초로 하는 것이 아닙니다. 사실은 현재의 운전 결정은 몇 초 전의 교통 상황이나 도로 상황 등이 무엇이었느냐에 기초를 둡니다. 따라서 **LSTM**이나 **GRU** 등의 반복 신경망(**RNN**)이 이 문제를 어떻게 해결하는지 지켜보는 것도 흥미로울 것입니다.
* 마지막으로, (딥) 강화 학습 또한 추가적인 프로젝트로서 흥미로운 주제가 될 수 있습니다.


## 크레딧

Credits for this code go to [upul](https://github.com/upul/Behavioral-Cloning). 
