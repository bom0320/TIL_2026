# Advanced Animations with UIKit (WWDC 2017)

## Overview

- **Basics**
  - 애니메이션이 어떻게 동작하는지와 시간 기반으로 어떻게 진행되는지에 대한 기본 개념을 다룬다.
- **Interactive and Interruptible Animations**
  - 사용자 입력을 통해 애니메이션을 직접 제어하고, 실행 중인 애니메이션을 중단 및 재개하는 방법을 다룬다.
- **New Property Animator Behaviors**
  - ios 11에서 도입된 `UIViewPropertyAnimator`의 새로운 기능과, 애니메이션을 보다 유연하게 제어하는 방법을 다룬다.
- **Coordinating Animations**
  - 여러 애니메이션을 하나의 흐름으로 구성하고, 서로 다른 타이밍과 특성을 가진 애니메이션을 조합하는 방법을 설명한다.
- **Tips and Tricks**
  보다 완성도 높은 애니메이션을 만들기 위한 다양한 기법과 실전 활용 팁을 소개한다.

## Abstract

현대 인터페이스 디자인에서 유동성(fluidity)과 인터랙티브 애니메이션의 역할을 통합적으로 분석한다. 기존의 시간 기반 애니메이션 모델에서 벗어나, 사용자 입력에 의해 실시간으로 제어되는 동적 시스템으로의 전환을 중심으로 논의를 전개한다. Apple WWDC 세션을 기반으로, 반응성, 제스처 기반 상호작용, 물리 기반 애니메이션, 그리고 다중 애니메이션 조합 구조를 분석하고, 이를 통해 인터페이스를 사용자의 인지와 신체의 연장으로 만드는 설계 원칙을 제시한다.

---

## 1. Introduction

인터페이스는 더 이상 단순한 시각적 출력 시스템이 아니라, 사용자와 상호작용하는 동적 시스템으로 발전하고 있다. 특히 모바일 환경에서 인터페이스는 손의 움직임과 직접적으로 연결되며, 이는 인터페이스가 사용자의 신체와 인지의 연장처럼 느껴져야 함을 의미한다.

기존의 애니메이션 시스템은 시간 기반(duration-driven) 구조를 중심으로 설계되었으며, 사용자 입력과의 실시간 동기화가 어려웠다. 이에 따라 현대 인터페이스는 다음과 같은 방향으로 전환된다:

- 시간 기반 애니메이션 → 상태 기반 시스템
- 재생 중심 → 제어 중심
- 고정된 흐름 → 사용자 입력 기반 흐름

---

## 2. Fluid Interface Design Principles

### 2.1 Response and Latency

유동적인 인터페이스의 핵심은 즉각적인 반응성이다. 인간은 지연에 매우 민감하며, 작은 지연도 인터페이스와의 연결감을 단절시킨다. 따라서 모든 인터랙션은 즉각적으로 반응해야 한다.

---

### 2.2 Redirection and Interruptibility

사용자는 행동 중에도 의도를 변경한다. 인터페이스는 이러한 특성을 반영하여, 제스처 수행 중에도 방향을 변경할 수 있어야 한다.

이는 다음과 같은 구조를 가진다:

- 행동과 사고의 병렬 처리
- 중간 상태에서의 자유로운 전환

---

### 2.3 Spatial Consistency

인터페이스는 공간적 일관성을 유지해야 한다. 객체는 동일한 경로를 따라 등장하고 사라져야 하며, 이는 사용자의 공간적 기억과 일치해야 한다.

---

### 2.4 Gesture Hinting

인터페이스는 사용자의 다음 행동을 예측하고 이를 암시해야 한다. 이는 제스처의 방향과 결과를 시각적으로 전달함으로써 직관적인 사용성을 제공한다.

---

### 2.5 Light Input and Amplified Output

사용자의 입력은 최소화되어야 하며, 시스템은 이를 증폭하여 자연스럽고 만족스러운 결과를 생성해야 한다. 이는 속도, 위치, 힘 등의 정보를 활용하여 구현된다.

---

### 2.6 Elastic Boundaries

인터페이스의 경계는 단절이 아닌 탄성으로 표현되어야 한다. 이는 사용자에게 상태를 전달하면서도 자연스러운 연결감을 유지한다.

---

## 3. Interactive Animation Systems

### 3.1 From Time-Based to Interactive Animation

기존 애니메이션은 시간에 따라 자동으로 진행되었으나, 현대 인터페이스에서는 사용자 입력이 애니메이션 진행을 직접 제어한다.

---

### 3.2 UIViewPropertyAnimator

`UIViewPropertyAnimator`는 다음과 같은 기능을 제공한다:

- 애니메이션 진행률 제어 (fractionComplete)
- pause / continue 제어
- interruptible 구조 지원

---

### 3.3 Timing Functions

타이밍 함수는 시간과 진행률 간의 관계를 정의한다. Linear, ease-in, ease-out, 그리고 커스텀 Bezier 곡선이 사용된다.

특히 interactive animation에서는 linear timing이 중요한 역할을 한다.

---

### 3.4 Interactive Animation Mechanism

인터랙티브 애니메이션은 다음과 같은 과정으로 구성된다:

1. 애니메이터 생성
2. 애니메이션 pause
3. 사용자 입력 기반 진행률 조정
4. continueAnimation 호출

이 과정에서 timing curve는 자동으로 변환 및 재매핑된다.

---

### 3.5 Interruptible Animation

애니메이션은 실행 중에도 중단 및 재개가 가능해야 한다. 이를 위해 현재 진행 상태를 유지하고, 자연스럽게 이어지는 구조가 필요하다.

---

## 4. Dynamic Motion and Physical Modeling

### 4.1 Spring-Based Animation

스프링 기반 애니메이션은 물리적 특성을 반영하여 자연스러운 움직임을 생성한다.

- Critically damped: overshoot 없음
- Underdamped: oscillation 포함

---

### 4.2 Motion as Behavior

애니메이션은 사전 정의된 동작이 아니라, 사용자와의 상호작용을 기반으로 하는 행동으로 정의된다.

---

## 5. Coordinated Animation Systems

### 5.1 Multi-Animator Architecture

여러 애니메이션을 동시에 조합하여 하나의 인터랙션을 구성할 수 있다. 각 애니메이션은 서로 다른 타이밍을 가지면서도 동일한 진행률을 공유한다.

---

### 5.2 Custom Timing Control

기본 타이밍 함수 대신 커스텀 Bezier 곡선을 사용하여 더욱 정교한 움직임을 구현할 수 있다.

---

### 5.3 View Morphing

두 개의 뷰를 다음 요소를 통해 자연스럽게 연결할 수 있다:

- scale
- translation
- opacity blending

---

## 6. Advanced Techniques

### 6.1 Keyframe Animation

keyframe을 활용하여 애니메이션의 시작 시점과 종료 시점을 세밀하게 제어할 수 있다.

---

### 6.2 Additive Animation

여러 애니메이션을 동시에 적용하여 하나의 속성을 제어할 수 있으며, 이를 통해 복합적인 움직임을 생성할 수 있다.

---

### 6.3 Property Expansion

corner radius와 같은 속성도 인터랙티브하게 제어 가능하며, 부분적 적용(maskedCorners)도 가능하다.

---

## 7. Design Implications

현대 인터페이스는 다음과 같은 특성을 가져야 한다:

- 사용자 입력과 실시간 동기화
- 중단 및 재개 가능
- 물리적 직관 반영
- 일관된 움직임 시스템

---

## 8. Conclusion

인터페이스 애니메이션은 단순한 시각적 효과를 넘어, 사용자와 시스템 간의 상호작용을 정의하는 핵심 요소이다. 특히 인터랙티브 및 인터럽터블 구조는 사용자의 의도와 움직임을 실시간으로 반영하며, 이를 통해 인터페이스는 사용자 경험의 연장으로 기능한다.

---

## References

[1] Apple Inc., “Designing Fluid Interfaces,” WWDC 2018, Session 803.  
Available: https://developer.apple.com/videos/play/wwdc2018/803/

[2] Apple Inc., “Advanced Animations with UIKit,” WWDC 2017, Session 230.  
Available: https://developer.apple.com/videos/play/wwdc2017/230/

[3] Nonstrict, “WWDC 2017 Session 230 Transcript,”  
Available: https://nonstrict.eu/wwdcindex/wwdc2017/230/
