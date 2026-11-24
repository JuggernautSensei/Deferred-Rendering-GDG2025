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

부경대학교 GDG 2025에서 Deferred Rendering의 원리와 한계, 해결 방법을 발표했습니다. C++과 Direct3D 11로 만든 렌더러에서 같은 창고 장면을 Forward와 Deferred로 그려, **조명이 많아질 때 G-buffer의 비용을 감수할 만큼 조명 계산이 줄어드는지** 비교했습니다. 발표 후반에는 안티앨리어싱과 반투명 물체를 처리하는 방법을 다뤘습니다.

## Forward의 낭비와 Deferred의 두 단계

Forward는 물체를 그리면서 재질과 조명을 함께 계산합니다. 앞쪽 물체가 나중에 그려지면, 먼저 그린 뒤쪽 표면의 조명 계산이 그대로 버려질 수 있습니다.

![뒤쪽 상자를 먼저 그려 이미 계산한 픽셀이 앞쪽 상자에 가려지는 예](Resources/Forward-Overdraw.png)

Deferred는 기하 패스와 조명 패스를 분리합니다. 기하 패스에서 불투명 표면의 법선과 재질을 G-buffer에 기록하면, 깊이 테스트를 거쳐 각 픽셀에 가장 가까운 표면만 남습니다. 조명 패스는 이 표면을 읽어 빛을 계산합니다.

![Forward와 Deferred에서 조명을 계산하는 시점을 비교한 파이프라인](Resources/Forward-vs-Deferred-Pipeline.png)

## G-buffer 구성

렌더러는 MRT(Multiple Render Targets)로 법선, 알베도·거칠기, 금속성·차폐 값을 세 렌더 타깃에 기록하고, 깊이 버퍼를 함께 사용합니다. 아래 디버그 화면은 조명 패스에 넘기는 네 버퍼를 보여줍니다.

![법선, 알베도와 거칠기, 금속성과 차폐, 깊이를 각각 표시한 디버그 화면](Resources/G-Buffer-Layout.png)

G-buffer를 쓰고 읽는 대역폭을 줄이기 위해 월드 위치는 깊이에서 복원했습니다. 픽셀 좌표와 깊이에 역투영·역뷰 변환을 적용하면 조명 계산에 필요한 위치를 얻을 수 있습니다. 재질 값은 8비트 정규화 포맷에, 음수도 표현해야 하는 법선은 16비트 부동소수점 포맷에 담았습니다.

![G-buffer 디버그 화면과 같은 버퍼로 조명을 계산한 결과](Resources/G-Buffer-Result.png)

## Forward와 Deferred의 성능 비교

![작업등 128개로 비춘 물류창고 실험 장면](Resources/Warehouse-Scene.png)

RTX 4060 Laptop GPU에서 1280×720으로 불투명 도형 2,556개를 그렸습니다. 카메라와 PBR 재질을 고정하고 작업등 수를 바꿨습니다. Forward는 픽셀 셰이더가 실행될 때 모든 작업등의 범위를 확인하고, Deferred는 광원마다 구형 라이트 볼륨을 그려 해당 영역의 픽셀을 검사합니다.

| 작업등 | Forward GPU 시간 | Deferred GPU 시간 |
| --- | ---: | ---: |
| 꺼짐 | 0.152 ms | 0.169 ms |
| 8개 | 0.177 ms | 0.193 ms |
| 32개 | 0.383 ms | 0.307 ms |
| 128개 | 1.234 ms | 0.784 ms |

작업등이 꺼져 있거나 8개일 때는 G-buffer를 쓰고 읽는 비용 때문에 Forward가 빨랐습니다. 32개와 128개에서는 라이트 볼륨으로 조명 계산 범위를 좁힌 Deferred가 빨랐고, **128개에서는 GPU 시간이 36.4% 줄었습니다.** Deferred에서 G-buffer를 만드는 시간은 조명 수와 관계없이 약 0.15 ms였습니다.

표는 가까운 물체부터 그린 결과입니다. 작업등 128개에서 그리기 순서만 먼 물체 우선에서 가까운 물체 우선으로 바꾸자, Early-Z로 가려진 픽셀을 일찍 제외해 Forward 시간이 1.581 ms에서 1.234 ms로 줄었습니다. 같은 가까운 순서에서 Deferred는 0.784 ms였습니다.

## Deferred의 한계

### 안티앨리어싱

한 픽셀에 여러 표면이 걸치면 MSAA를 적용한 Deferred에서는 샘플별로 표면 정보를 저장하고 조명을 계산해야 할 수 있습니다. 이 렌더러에는 최종 색상의 차이로 경계를 찾아 주변 픽셀과 섞는 FXAA를 구현했습니다. 추가 G-buffer 샘플 없이 경계를 부드럽게 만들 수 있고, 가는 선이나 텍스처는 흐려질 수 있습니다.

![FXAA가 감지한 구의 경계를 빨간색으로 표시한 결과](Resources/FXAA-Edge-Detection.png)

### 반투명 물체

G-buffer에는 픽셀마다 가장 가까운 표면만 남습니다. 유리처럼 겹친 표면을 함께 보여주려면 색을 별도로 합성해야 합니다. 발표에서는 다음 방법을 비교했습니다.

- **Forward 패스**에서 반투명 물체를 뒤에서 앞으로 그립니다. 물체가 서로 교차하면 물체 단위 정렬만으로 픽셀별 순서를 맞추기 어렵습니다.
- **디더링**으로 투명도에 따라 일부 픽셀을 건너뜁니다. 불투명 패스에 넣을 수 있고, 아래 그림처럼 패턴이 보일 수 있습니다.
- **순서 독립 투명도**(OIT)는 그리기 순서에 덜 의존하도록 여러 반투명 표면을 합성합니다. 픽셀마다 표면을 모으는 방식은 추가 저장 공간이 필요합니다.

![투명도가 다른 세 상자에서 디더링 패턴이 보이는 예](Resources/Dithering-Transparency.png)

## 발표 자료

* [발표 자료 PDF](Deferred%20Rendering%20%EC%86%8C%EA%B0%9C.pdf)
* [발표 자료 PPTX](Deferred%20Rendering%20%EC%86%8C%EA%B0%9C.pptx)
