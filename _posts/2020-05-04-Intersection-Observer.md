---
title: "[Observer API] IntersectionObserver"

categories:
  - Develop
tags:
  - Security
---

## IntersectionObserver란?

`IntersectionObserver`는 자바스크립트에서 제공하는 `Observer API` 중 타켓 요소와 `viewport` 사이의 교차 지점 변화를 비동기적으로 관찰하는 객체이다.

`Intersection`은 말 그대로 눈에 보이는 영역에 타겟 요소가 교차하는 것을 말하며 교차 비율에 따라 어떤 로직을 수행할지 정할 수 있다.

대표적으로 `Infinity Scroll`과 `Skeleton Loading` 기능 구현에 적합하다.

## 어떻게 사용할까?

### IntersectionObserver 객체 생성

```jsx
const observer = new IntersectionObserver(callback, options);
```

- `callback` - 교차 지점에 들어왔을 때 수행할 함수
- `options` - 교차 비율, 뷰포트에 해당하는 요소 등 환경 설정

### callback 함수 정의

`callback` 함수는 `IntersectionObserverEntry` 객체들과 감시자 요소를 파라미터로 받는다. `IntersectionObserverEntry` 객체들의 속성 중 가장 많이 쓰이는 것은 교차하는 비율을 나타내는 `IntersectionRatio`와 타겟 요소가 현재 교차하고 있는지 여부를 나타내는 `isIntersecting`이다.

```jsx
const callback = (entries, observer) => {
  entries.forEach((entry) => {
    // 타겟 요소가 바뀌는 로직
  });
};
```

### options 정의

```jsx
const options = {
  root: document.querySelector("#scroll-area"),
  rootMargin: "0px",
  threshold: 1.0,
};
```

- `root` - 교차 지점을 감시하는 감시자 요소, viewport가 기본값
- `rootMargin` - 감시자 요소가 가지는 여백, 0이 기본값
- `threshold` - 타켓 요소가 얼마나 보이는지에 대한 비율, 1이 기본값 / e.g.) 0.5 = 50%

### Observer 등록

```jsx
...
observer.observe(target);
```
