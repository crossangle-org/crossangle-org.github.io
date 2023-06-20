---
title:  "마크다운으로 한번 블로그를 써보자"
classes: wide
categories: 
  - sample
tags:
  - tag1
  - tag2
toc: true
toc_label: "목차 셈플"
---

# 가장 큰 재목

## 제목 목차

### 샾이많아질수록

#### 점점더 작아지지

##### 아마도

###### 어디까지 될까??



구분선은 아래처럼 엔터는 백슬레시\
**문단이랑은** 붙어있을수 없다

문단구분은 엔터 두번

---

목차:
- 리스트
- 리스트 2 
- 리스트 3

순번있는 목차
1. 아마
2. 이렇게
3. 쓸겁니다.

---

여기는 간단한 표 만들기

| 1 | 2 | 3 |
|---|---|---|
| 4 | 5 | 6 |
| 7 | 8 | 9 |

![이미지 링크](https://images.unsplash.com/photo-1686595092928-0252b92e007e?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=774&q=80)

![이미지 리사이즈](https://images.unsplash.com/photo-1686595092928-0252b92e007e?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=774&q=80){: width="400", height="200"}

<img src="https://images.unsplash.com/photo-1686595092928-0252b92e007e?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=774&q=80" width="200" height="400">

[링크 텍스트](https://example.com)

유투브 임베드 하기

{% include video id="UaifuwEgOh4" provider="youtube" %}

---

```python
# Python code
def get_hello_world():
    return "Hello, World!"

print(get_hello_world())
```