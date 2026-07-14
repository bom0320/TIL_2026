# Lighthouse LCP 성능 개선 트러블슈팅

## 문제 상황

포트폴리오 웹사이트를 모바일 환경에서 Lighthouse로 측정한 결과, 성능 점수와 LCP 지표가 낮게 나타났다.

### 개선 전

![Lighthose 개선 전](https://private-user-images.githubusercontent.com/167315197/621322798-bca0aba1-5773-4a51-9e94-7c033f460742.png?jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3ODQwMTYwMzEsIm5iZiI6MTc4NDAxNTczMSwicGF0aCI6Ii8xNjczMTUxOTcvNjIxMzIyNzk4LWJjYTBhYmExLTU3NzMtNGE1MS05ZTk0LTdjMDMzZjQ2MDc0Mi5wbmc_WC1BbXotQWxnb3JpdGhtPUFXUzQtSE1BQy1TSEEyNTYmWC1BbXotQ3JlZGVudGlhbD1BS0lBVkNPRFlMU0E1M1BRSzRaQSUyRjIwMjYwNzE0JTJGdXMtZWFzdC0xJTJGczMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDI2MDcxNFQwNzU1MzFaJlgtQW16LUV4cGlyZXM9MzAwJlgtQW16LVNpZ25hdHVyZT1lZTBlMWIyNmUxMWU0MzA0ZTI5ZTlkMDI5ODI1M2Y0OTVkMDVlNDM0MDk5MjMxYzgwODQzMWJlY2IzMjcxNjBjJlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCZyZXNwb25zZS1jb250ZW50LXR5cGU9aW1hZ2UlMkZwbmcifQ.gzq-OeGnEWXX47JgVCzaVTmzhyFKNAOXDOPsvUyviMk)

| 지표        | 결과 |
| ----------- | ---: |
| Performance |   86 |
| FCP         | 1.0s |
| LCP         | 4.1s |
| TBT         | 90ms |
| CLS         |    0 |

FCP와 TBT는 정상 범위였지만, LCP가 4.1초로 측정되며 전체 성능 점수에 가장 큰 영향을 주고 있었다.

## LCP란?

LCP는 `Largest Contentful Paint`의 약자로, 초기 화면에서 가장 큰 콘텐츠 요소가 사용자에게 표시되기까지 걸린 시간을 의미한다.

일반적으로 다음 기준으로 평가한다.

- 2.5초 이하: 좋음
- 2.5초 초과 4초 이하: 개선 필요
- 4초 초과: 느림

단순히 Lighthouse 점수만으로는 어떤 요소가 LCP로 측정됐는지 정확히 알기 어려웠기 때문에, Chrome DevTools의 Performance 패널을 사용해 실제 요소를 추적했다.

## 원인 추적

### Chrome Performance 패널에서 LCP 요소 확인

다음 순서로 실제 LCP 요소를 확인했다.

1. Chrome DevTools 상단에서 `Performance` 탭으로 이동
2. `Record and reload` 실행
3. 페이지 로딩 측정이 끝난 뒤 타임라인의 `LCP` 마커 선택
4. 하단 `Summary` 패널의 `Related node` 확인

즉, 모바일 초기 화면에서 `life-motion` 영역의 이미지가 가장 큰 콘텐츠 요소로 판단되고 있었다.

## 기존 구현

기존에는 첫 8개의 이미지에 모두 `priority`를 적용하고 있었다.

```ts
const shouldPreload = index < 8;

<Image
  src={item.src}
  alt={item.title}
  fill
  priority={shouldPreload}
  loading={shouldPreload ? undefined : "lazy"}
  sizes="(max-width: 640px) 45vw, (max-width: 900px) 32vw, 24vw"
  className="life-motion__image"
/>;
```

`priority` 는 초기 화면에서 반드시 필요한 핵심 이미지를 우선적으로 불러오기 위한 옵션이다.

하지만 첫 8개의 이미지에 모두 적용하면서, 브라우저가 여러 이미지를 동시에 높은 우선순위로 요청하고 있었다.

결과적으로 실제로 중요한 이미지와 그렇지 않은 이미지가 모두 같은 우선순위로 처리되면서 불필요한 네트워크 경쟁이 발생할 가능성이 있었다.

또한 Lighthouse의 이미지 전달 분석에서 일부 이미지의 압축률을 더 높일 수 있다는 안내가 표시 됐다.

## 해결 방법

`life-motion` 이미지는 Hero의 대표 이미지가 아니라 반복되는 갤러리 이미지이므로, 전체 이미지에 적용된 `priority` 설정을 제거했다.

Next.js Image 컴포넌트의 기본 지연 로딩을 활용하고, 이미지 품질 값을 낮춰 전송 용량도 함께 줄였다.

```ts
<Image
  src={item.src}
  alt={item.title}
  fill
  quality={60}
  sizes="(max-width: 640px) 45vw, (max-width: 900px) 32vw, 24vw"
  className="life-motion__image"
/>
```

### 변경 내용

- 첫 8개 이미지의 `priority` 제거
- 명시적인 `loading="lazy"` 제거
- Next.js Image 기본 지연 로딩 활용
- 이미지 품질 75에서 60으로 조정
- 반응형 `sizes` 설정 유지
- 더 이상 사용하지 않는 `index` prop 제거

## 개선 결과

![](https://private-user-images.githubusercontent.com/167315197/621322797-75408e93-d56c-4f6c-bdf8-089c8674c169.png?jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3ODQwMTg4NzYsIm5iZiI6MTc4NDAxODU3NiwicGF0aCI6Ii8xNjczMTUxOTcvNjIxMzIyNzk3LTc1NDA4ZTkzLWQ1NmMtNGY2Yy1iZGY4LTA4OWM4Njc0YzE2OS5wbmc_WC1BbXotQWxnb3JpdGhtPUFXUzQtSE1BQy1TSEEyNTYmWC1BbXotQ3JlZGVudGlhbD1BS0lBVkNPRFlMU0E1M1BRSzRaQSUyRjIwMjYwNzE0JTJGdXMtZWFzdC0xJTJGczMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDI2MDcxNFQwODQyNTZaJlgtQW16LUV4cGlyZXM9MzAwJlgtQW16LVNpZ25hdHVyZT0wNTczMDc1MWY2NDFhMGU3NjM0NWMyZDBjNTk1YTYxOWI1MDJkNDYzZWQyZjgyMjg1ZjM1YTZkNDIxNTYxN2M5JlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCZyZXNwb25zZS1jb250ZW50LXR5cGU9aW1hZ2UlMkZwbmcifQ.xqCJ62CTTETKphA-8HGX7Ntg9_ZoGsiqXxOSxVrBi0c)

변경 사항을 배포한 뒤, 동일한 시크릿 모바일 환경에서 Lighthouse를 다시 측정했다.

| 지표        | 개선 전 | 개선 후 |
| ----------- | ------- | ------- |
| Performance | 86      | 99      |
| FCP         | 1.0s    | 0.3s    |
| LCP         | 4.1s    | 1.0s    |
| TBT         | 90ms    | 0ms     |
| CLS         | 0       | 0       |

가장 큰 문제였던 LCP가 4.1초에서 1.0초로 감소했고, Lighthouse 성능 점수도 86점에서 99점으로 개선됐다.

## 배운 점

- 이번 문제를 통해 Lighhouse 점수만 보고 바로 코드를 수정하기 보다, 실제 병목 요소를 먼저 확인하는 과정이 중요하다는 것을 알게 되었다.
- 특히 LCP 요소가 이미지로 표시됐다고 해서 단순히 이미지 파일 크기만 문제라고 판단해서는 안 된다.
- Chrome Performance 패널의 LCP 마커와 `Related node`를 통해 실제 DOM 요소를 확인하고, 해당 요소의 로딩 우선순위와 렌더링 방식을 코드에서 검증해야한다.
- 또한 `priority`는 많이 사용할수록 좋은 옵션이 아니라, 초기 화면에서 반드시 필요한 소수의 이미지에만 선택적으로 적용해야 한다.

## 요약

> 모바일 Lighthouse 측정에서 LCP가 4.1초로 나타난 문제를 분석했습니다. Chrome Performance 패널의 LCP 마커와 Related node를 통해 실제 병목 요소가 life-motion 이미지임을 확인했습니다. 이후 첫 8개 이미지에 모두 적용돼 있던 priority를 제거하고, Next.js Image의 기본 지연 로딩과 이미지 품질 최적화를 적용했습니다. 그 결과 Lighthouse 성능 점수를 86점에서 99점으로, LCP를 4.1초에서 1.0초로 개선했습니다.
