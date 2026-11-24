<h1 align="center">Deferred Rendering 소개</h1>

<p align="center">Deferred 렌더링의 원리와 한계, 해결 방안을 다룬 GDG 2025 발표 자료</p>

<p align="center">
  <img src="https://img.shields.io/badge/C++-00599C?logo=cplusplus&logoColor=white" alt="C++">
  <img src="https://img.shields.io/badge/Direct3D-11-5E9C36" alt="Direct3D 11">
</p>

![Deferred Rendering 발표 자료 표지](Resources/Thumbnail.png)

## Contents

* [프로젝트 소개](#프로젝트-소개)
* [Forward의 낭비와 Deferred의 두 단계](#forward의-낭비와-deferred의-두-단계)
* [G-buffer 구성](#g-buffer-구성)
* [Forward와 Deferred의 성능 비교](#forward와-deferred의-성능-비교)
* [Deferred의 한계](#deferred의-한계)
* [발표 자료](#발표-자료)

## 프로젝트 소개

부경대학교 GDG 2025에서 Deferred Rendering을 주제로 발표한 46쪽 분량의 자료입니다. Forward Rendering이 계산을 낭비하는 지점과 Deferred Rendering이 이를 줄이는 방식을 설명하고, C++과 Direct3D 11로 직접 만든 렌더러에서 두 방식을 같은 조건으로 측정해 "Deferred가 항상 빠른가?"에 답합니다. 창고 장면에서 작업등을 끄거나 8개 켰을 때는 Forward가 빨랐고, 32개와 128개에서는 Deferred가 빨랐습니다. 작업등 128개에서는 GPU 시간이 1.234 ms에서 0.784 ms로 줄었습니다. 발표 후반에는 이 성능 차이의 원인과 안티앨리어싱·투명 물체 처리 비용을 다뤘습니다. [발표 자료 PDF](Deferred%20Rendering%20%EC%86%8C%EA%B0%9C.pdf)

## Forward의 낭비와 Deferred의 두 단계

![작은 상자 위에 큰 상자가 차례로 그려지며 앞서 그린 부분이 가려지는 슬라이드](Resources/Forward-Overdraw.png)

Forward Rendering은 물체를 그리면서 재질과 조명을 함께 계산합니다. 나중에 그린 물체가 앞을 가리면 이미 계산한 픽셀의 조명은 버려지고, 광원이 많을수록 버려지는 계산도 커집니다.

![Forward는 그리면서 조명을 계산하고 Deferred는 기하 정보만 기록한 뒤 보이는 표면에만 조명을 적용하는 파이프라인 비교](Resources/Forward-vs-Deferred-Pipeline.png)

Deferred Rendering은 이를 두 단계로 나눕니다. 먼저 모든 불투명 물체의 법선과 재질을 G-buffer에 기록합니다. 깊이 테스트를 거치면 픽셀마다 맨 앞 표면의 정보만 남습니다. 그다음 남은 표면에만 조명을 적용하므로, 무거운 조명 계산이 화면에 실제로 보이는 표면에서만 실행됩니다.

## G-buffer 구성

![Normal, AlbedoRoughness, MetallicAO, Depth 네 버퍼의 디버그 화면](Resources/G-Buffer-Layout.png)

G-buffer는 모든 물체가 쓰고 조명 단계가 다시 읽으므로, 커질수록 메모리 대역폭이 늘어납니다. 직접 구현한 렌더러는 법선, 색상+거칠기, 금속성+차폐광, 깊이의 네 버퍼만 쓰고, MRT(Multiple Render Targets)로 드로우마다 세 색상 타깃에 표면 정보를 기록하고, 깊이 테스트를 통과한 표면의 깊이는 깊이 버퍼에 남깁니다.

위치 버퍼는 두지 않았습니다. 깊이 값으로 정규화 장치 좌표를 만들고 투영·뷰 역행렬을 곱한 뒤 w로 나누면 월드 위치를 복원할 수 있기 때문입니다. 포맷은 값의 범위에 맞췄습니다. 색상, 금속성, 차폐광, 거칠기처럼 [0, 1]로 충분한 값은 `R8G8B8A8_UNORM`에, 법선처럼 범위나 정밀도가 더 필요한 값은 `R16G16B16A16_FLOAT`에 담습니다.

![G-buffer 디버그 화면과 이를 이용해 조명을 계산한 구 렌더링 결과](Resources/G-Buffer-Result.png)

## Forward와 Deferred의 성능 비교

![선반과 상자가 늘어선 물류창고 통로를 작업등 128개로 비춘 실험 장면](Resources/Warehouse-Scene.png)

같은 창고 장면을 Forward와 Deferred로 그리면서 작업등 수에 따른 GPU 시간을 비교했습니다. 측정 환경은 RTX 4060 Laptop GPU, 1280×720입니다.

| 작업등 | Forward | Deferred |
| --- | ---: | ---: |
| 꺼짐 | 0.152 ms | 0.169 ms |
| 8개 | 0.177 ms | 0.193 ms |
| 32개 | 0.383 ms | 0.307 ms |
| 128개 | 1.234 ms | 0.784 ms |

작업등이 없거나 8개일 때는 G-buffer를 기록하는 비용 때문에 Forward가 빨랐습니다. 작업등이 32개와 128개일 때는 광원마다 라이트 볼륨 안의 픽셀에만 조명을 계산한 Deferred가 빨랐습니다. 작업등 128개에서는 Deferred의 GPU 시간이 Forward보다 **36.4% 짧았습니다.**

## Deferred의 한계

![구 렌더링 결과에서 FXAA가 찾은 경계를 빨간색으로 표시한 슬라이드](Resources/FXAA-Edge-Detection.png)

MSAA는 Deferred에서도 쓸 수 있지만, 한 픽셀에 여러 표면이 걸치면 G-buffer에 샘플마다 표면 정보를 저장하고 조명도 샘플마다 계산해야 합니다. 그래서 렌더러에는 최종 색상에서 경계를 찾아 주변 픽셀과 섞는 FXAA를 구현했습니다. G-buffer에 추가 샘플이 필요 없는 대신, 가는 선이나 텍스처가 흐려질 수 있습니다.

![디더링 강도를 달리해 반투명하게 표현한 세 상자](Resources/Dithering-Transparency.png)

G-buffer에는 픽셀마다 맨 앞 표면 하나만 남으므로, 유리처럼 뒤가 비치는 물체는 그대로 그릴 수 없습니다. 발표에서는 세 가지 방법을 비교했습니다. Deferred가 끝난 뒤 반투명 물체만 깊이순으로 정렬해 Forward로 그리는 방법은 기존 방식을 그대로 쓰지만, 물체 단위 정렬이라 서로 관통하는 반투명 물체를 표현하지 못합니다. 디더링은 투명도만큼 픽셀을 솎아 불투명 패스로 그리므로 가볍지만 패턴이 보입니다. Order-Independent Transparency는 픽셀마다 여러 표면을 모아 합성해 순서에 덜 의존하지만 저장 공간이 더 듭니다.

## 발표 자료

* [발표 자료 PDF](Deferred%20Rendering%20%EC%86%8C%EA%B0%9C.pdf)
* [발표 자료 PPTX](Deferred%20Rendering%20%EC%86%8C%EA%B0%9C.pptx)
