# 공간 인덱스는 어떻게 후보를 줄일까? — PostGIS GiST의 동작 원리

## 본문

[공간 인덱스는 어떻게 후보를 줄일까? — PostGIS GiST의 동작 원리](https://qkrqudgns.notion.site/PostGIS-GiST-40a041facb658158a107ebe8fdfa3b77?source=copy_link)

## PostGIS GiST의 동작 원리

주차의민족에서 목적지 주변 주차장을 검색할 때는 모든 주차장과의 거리를 계산할 필요가 없습니다. 공간 인덱스로 멀리 떨어진 지역을 먼저 제외하고, 남은 후보의 거리만 판정할 수 있습니다.

이 글은 반경 600m 주차장 검색을 예로, 공간 범위를 계층으로 묶는 방식과 검색 범위 밖의 가지를 건너뛰는 과정을 설명합니다. 이어 PostGIS의 GiST 인덱스와 `ST_DWithin`이 실제 쿼리에서 어떻게 역할을 나누는지 살펴봅니다.
