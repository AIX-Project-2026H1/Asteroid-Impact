# Asteroid-Impact

<a id="overview"></a>

**Overview.** 이 프로젝트는 NASA 소행성 데이터를 기반으로 지구 근접 천체 `neo`와 잠재적 지구위협 소행성 `pha`를 분류하는 것을 목표로 한다.

원본 데이터에는 소행성의 물리적 특성, 궤도 요소, 식별자, 파생 변수 등이 포함되어 있다. 모델 학습에 앞서 target leakage 가능성이 있거나, 식별자, 상수 데이터 및 결측치가 많거나 다른 변수로부터 유도되는 feature를 제거하였다.

<br>

## Table of Contents

* [Overview](#overview)
* [Video Presentation](#video-presentation)
* [1. Data Preprocessing](#1-data-preprocessing)

  * [1.1 데이터 개요](#11-데이터-개요)
  * [1.2 Feature 요약](#12-feature-요약)
  * [1.3 제거한 Feature](#13-제거한-feature)
* [2. Model Training](#2-model-training)
* [3. Result](#3-result)

  * [3.1 NEO 분류 결과](#31-neo-분류-결과)
  * [3.2 PHA 분류 결과](#32-pha-분류-결과)
* [4. Conclusion](#4-conclusion)
* [5. Members](#5-members)

<br>

## Video Presentation

[![Video Presentation](https://img.youtube.com/vi/wDchsz8nmbo/0.jpg)](https://www.youtube.com/watch?v=wDchsz8nmbo)

[Video Presentation](https://www.youtube.com/watch?v=wDchsz8nmbo)

<br>

# 1. Data Preprocessing

## 1.1 데이터 개요

원본 데이터는 71,351개의 소행성 관측치와 32개의 feature로 구성되어 있다. 전처리 후에는 70,578개의 행과 9개의 feature가 남았다.

| 항목              |            값 |
| --------------- | -----------: |
| 원본 행 수          |       71,351 |
| 원본 feature 수    |           32 |
| 전처리 후 행 수       |       70,578 |
| 전처리 후 feature 수 |            9 |
| Target variable | `neo`, `pha` |

전처리 후 남은 feature는 다음과 같다.

| Feature    | 설명               |
| ---------- | ---------------- |
| `neo`      | 지구 근접 천체 여부      |
| `pha`      | 잠재적 지구위협 소행성 여부  |
| `sats`     | 위성 수             |
| `e`        | 이심률              |
| `a`        | 궤도 장반경           |
| `i`        | 궤도 경사            |
| `om`       | 승교점 경도           |
| `w`        | 근일점 인수           |
| `moid_jup` | 목성과의 최소 궤도 교차 거리 |

분류 흐름은 2단계로 구성하였다. 먼저 `neo`를 예측하여 지구 근접 천체 여부를 판단하고, `neo = Y`로 분류된 천체에 대해서만 추가로 `pha`를 예측한다.

## 1.2 Feature 요약

각 feature에 대해 데이터 타입, 결측치 개수, 결측률, 고유값 개수, 상수 여부를 확인하였다. 이를 통해 제거할 feature를 선정하였다.

Feature 요약 결과 `G`, `PC`, `diameter`, `extent`, `albedo`, `rot_per`, `BV`는 결측률이 90% 이상이었다. 해당 feature들은 관측값이 거의 존재하지 않아 모델 학습에 사용하기 어렵다고 판단하여 열 단위로 삭제하였다.

## 1.3 제거한 Feature

결측률 외에도 식별자, 상수 feature, 파생 변수, target leakage 가능성이 있는 feature를 제거하였다.

| 제거 feature                   | 제거 이유                                            |
| ---------------------------- | ------------------------------------------------ |
| `spkid`, `full_name`, `name` | 소행성 고유 식별자 또는 이름                                 |
| `orbit_id`                   | NASA/JPL 내부 궤도해 식별자                              |
| `equinox`                    | 모든 값이 동일한 상수 feature                             |
| `tp`, `tp_cal`               | 근일점 통과 시각과 그 달력형 변환값                             |
| `moid`, `moid_ld`            | `pha` 정의에 직접 관련된 변수이며, `moid_ld`는 `moid`의 단위 변환값 |
| `H`                          | `pha` 정의에 직접 관련된 변수                              |
| `class`                      | 궤도 특성을 바탕으로 부여된 분류 label                         |
| `q`, `ad`                    | `a`, `e`로부터 유도 가능한 변수                            |
| `n`, `per`, `per_y`          | 공전 주기 및 공전 각속도 관련 중복 변수                          |

일부 변수는 다른 궤도 요소로부터 계산할 수 있다. 예를 들어 근일점 거리 `q`와 원일점 거리 `ad`는 다음과 같이 계산된다.

$$
q = a(1 - e)
$$

$$
ad = a(1 + e)
$$

따라서 `q`와 `ad`는 중복 정보를 줄이기 위해 제거하였다. `per`, `per_y`, `n` 역시 공전 주기와 공전 각속도를 나타내는 중복 변수이므로 제거하였다.

또한 `om`과 `w`는 각도 단위(degree)로 표현된 feature이다. 각도형 변수는 0°와 360°가 같은 방향을 의미하지만, 모델은 이를 서로 멀리 떨어진 숫자로 인식할 수 있다. 이를 보완하기 위해 `om`, `w`를 그대로 사용하지 않고 sin, cos 변환을 적용하였다.

| 기존 feature | 변환 feature         |
| ---------- | ------------------ |
| `om`       | `om_sin`, `om_cos` |
| `w`        | `w_sin`, `w_cos`   |

전처리 직후 데이터에는 `neo`, `pha`를 포함해 9개의 feature가 남았다. 이후 모델 학습 단계에서는 target인 `neo`, `pha`를 제외하고, 각도형 변수 `om`, `w`를 sin/cos 값으로 변환하여 입력 feature를 구성하였다.

최종 모델 학습에는 다음 feature를 사용하였다.

| Feature            | 설명                  |
| ------------------ | ------------------- |
| `sats`             | 위성 수                |
| `e`                | 이심률                 |
| `a`                | 궤도 장반경              |
| `i`                | 궤도 경사               |
| `moid_jup`         | 목성과의 최소 궤도 교차 거리    |
| `om_sin`, `om_cos` | 승교점 경도의 sin/cos 변환값 |
| `w_sin`, `w_cos`   | 근일점 인수의 sin/cos 변환값 |

`q`는 `neo`를 결정하는 핵심 변수에 가깝지만, 본 프로젝트에서는 `q`를 입력 feature에서 제거하였다. 따라서 모델은 직접적인 `q`값 없이 `a`, `e`를 통해 다음 관계를 간접적으로 학습해야 한다.

$$
q = a(1 - e)
$$

이는 모델이 정답에 가까운 파생 변수를 사용하는 대신, 남겨진 궤도 요소를 통해 지구 근접 천체 여부를 추론할 수 있는지를 확인하기 위한 전처리 방향이다.

불필요한 feature를 제거하고 각도형 feature 변환을 적용한 뒤, 남아 있는 결측 행은 `na.omit()`으로 제거하였다. 이후 전처리된 데이터를 사용하여 먼저 `neo`를 분류하고, `neo = Y`로 분류된 천체에 대해 추가로 `pha`를 분류하였다.

<br>

# 2. Model Training

모델 학습은 2단계 분류 구조로 진행하였다. 1단계에서는 지구 근접 천체 여부인 `neo`를 예측하고, 2단계에서는 `neo = Y`로 분류된 천체에 대해서만 잠재적 지구위협 소행성 여부인 `pha`를 예측하였다.

비교 모델로는 Random Forest와 Keras/TensorFlow 기반 딥러닝 모델을 사용하였다. 딥러닝 모델은 다층 퍼셉트론 구조를 사용하였으며, 클래스 불균형 문제를 고려하여 class weight를 적용하였다.

`pha`는 전체 데이터에서 `Y` 비율이 매우 낮은 불균형 target이다. 따라서 accuracy만으로 모델을 평가하면 대부분을 `pha = N`으로 예측하는 모델도 높은 성능처럼 보일 수 있다. 이를 보완하기 위해 precision, recall, F1-score, AUROC를 함께 확인하였다.

또한 기본 threshold인 0.5만 사용하지 않고, train set에서 threshold sweep을 수행하여 F1-score가 가장 높은 threshold를 선택하였다. 선택된 threshold를 test set 평가에 적용하여 최종 성능을 비교하였다.

<br>

# 3. Result

## 3.1 NEO 분류 결과

`neo` 분류에서는 Random Forest와 딥러닝 모델 모두 매우 높은 성능을 보였다.

| Model         | Threshold |  AUROC | Accuracy | Precision | Recall | F1-score |
| ------------- | --------: | -----: | -------: | --------: | -----: | -------: |
| Random Forest |      0.38 | 1.0000 |   0.9948 |    0.9919 | 0.9993 |   0.9956 |
| Deep Learning |      0.39 | 1.0000 |   0.9978 |    0.9981 | 0.9982 |   0.9981 |

두 모델 모두 AUROC가 1.0000으로 나타났으며, F1-score 역시 0.99 이상으로 매우 높았다. 이는 `q`를 직접 입력하지 않았음에도 모델이 `a`, `e`를 통해 `q = a(1 - e)` 관계를 거의 학습했기 때문으로 해석할 수 있다.

## 3.2 PHA 분류 결과

`pha` 분류에서는 클래스 불균형의 영향이 뚜렷하게 나타났다.

| Model         | Threshold |  AUROC | Accuracy | Precision | Recall | F1-score |
| ------------- | --------: | -----: | -------: | --------: | -----: | -------: |
| Random Forest |      0.31 | 0.8793 |   0.9608 |    0.3471 | 0.1175 |   0.1756 |
| Deep Learning |      0.64 | 0.8578 |   0.8894 |    0.1514 | 0.4582 |   0.2276 |

Random Forest는 precision이 0.3471로 더 높았지만, recall이 0.1175에 그쳤다. 이는 `pha = Y`인 천체를 보수적으로 예측하여 false positive는 줄였지만, 실제 PHA를 많이 놓쳤다는 의미이다.

딥러닝 모델은 precision이 0.1514로 낮았지만, recall은 0.4582로 Random Forest보다 높았다. 즉, 잘못 경고하는 경우는 더 많았지만 실제 PHA를 더 많이 탐지하였다.

위험 천체 탐지 문제에서는 실제 위험 천체를 놓치는 false negative가 중요하다. 따라서 `pha` 분류에서는 accuracy보다 recall과 F1-score를 중심으로 모델을 평가하는 것이 적절하다.

<br>

# 4. Conclusion

본 프로젝트에서는 NASA 소행성 데이터를 기반으로 `neo`와 `pha`를 순차적으로 분류하는 2단계 모델을 구성하였다. 전처리 과정에서는 식별자, 결측치가 많은 feature, 상수 feature, 파생 변수, target leakage 가능성이 있는 feature를 제거하였다. 또한 각도형 변수인 `om`, `w`는 sin/cos 변환을 적용하여 모델이 각도 데이터의 순환 구조를 반영할 수 있도록 하였다.

실험 결과, `neo` 분류는 Random Forest와 딥러닝 모델 모두 매우 높은 성능을 보였다. 특히 `q`를 제거했음에도 `a`, `e`만으로 `neo` 여부를 거의 정확하게 분류할 수 있었다.

반면 `pha` 분류는 클래스 불균형과 핵심 leakage feature 제거의 영향으로 더 어려운 문제였다. Random Forest는 보수적인 예측 경향을 보였고, 딥러닝 모델은 더 많은 PHA를 탐지하는 경향을 보였다. 최종적으로 위험 천체 탐지 목적에서는 실제 PHA를 더 많이 포착하는 recall 중심의 평가가 필요하다.

<br>

# 5. Members

2026H1 AI:X 팀 프로젝트(G05)

| 이름  |
| --- |
| 김동현 |
| 이동헌 |
| 이준하 |
| 피소유 |
