---
layout: post
title: "지오해싱(Geohashing)과 공간 분할 완전 정복: Geohash·Google S2·Uber H3로 위치 데이터 처리하기"
date: 2026-07-23
categories: [cs, computer-science]
tags: [geohash, geospatial, spatial-indexing, google-s2, uber-h3, location, z-order-curve, hilbert-curve]
---

"반경 1km 이내의 음식점을 찾아줘." — 우리가 매일 사용하는 위치 기반 서비스의 이 간단한 요청 뒤에는 수십억 개의 지리적 좌표를 밀리초 안에 검색하는 정교한 공간 인덱싱 알고리즘이 숨어 있습니다.

위도·경도라는 2차원 좌표를 어떻게 빠르게 인덱싱하고 검색할까요? Geohash, Google S2, Uber H3 — 세 가지 핵심 공간 분할 알고리즘을 깊이 파헤쳐 보겠습니다.

## 공간 인덱싱의 핵심 문제

전통적인 B-트리 인덱스는 1차원 데이터(숫자, 문자열)에 최적화되어 있습니다. 하지만 지리 좌표는 2차원이며, "서울 중심으로 5km 이내" 같은 **범위 쿼리**를 효율적으로 처리하려면 공간적 근접성을 인덱스 구조에 반영해야 합니다.

핵심 아이디어: **2차원 공간을 1차원 문자열로 매핑**하되, 공간적으로 가까운 점들이 문자열에서도 가깝도록 유지합니다. 이를 **공간 충전 곡선(Space-Filling Curve)**이라고 합니다.

## Geohash: Base-32 인코딩의 우아함

### 원리: Z-order Curve(Morton Code)

Geohash는 지구를 재귀적으로 격자(grid)로 분할합니다. 각 단계에서 경도(longitude) 방향으로 2등분, 위도(latitude) 방향으로 2등분하여 4개의 셀로 나눕니다.

```
위도 범위: [-90, 90]
경도 범위: [-180, 180]

1단계 분할:
  경도 < 0 → 0비트, 경도 ≥ 0 → 1비트
  위도 < 0 → 0비트, 위도 ≥ 0 → 1비트
  
서울 좌표 (37.5665, 126.9780):
  경도 비트: 126.9780 ≥ 0 → 1
  위도 비트: 37.5665 ≥ 0 → 1
  → 11...
```

비트를 5개씩 묶어 Base-32 문자로 인코딩합니다 (0-9, b-z 중 a, i, l, o 제외).

```python
class Geohash:
    BASE32 = '0123456789bcdefghjkmnpqrstuvwxyz'
    DECODE_MAP = {c: i for i, c in enumerate(BASE32)}
    
    @classmethod
    def encode(cls, lat, lon, precision=6):
        """
        위도/경도를 Geohash 문자열로 인코딩
        precision: 문자 수 (높을수록 정밀도 증가)
        """
        lat_range = [-90.0, 90.0]
        lon_range = [-180.0, 180.0]
        
        hash_str = []
        bits = 0
        bit_count = 0
        even = True  # True: 경도 비트, False: 위도 비트
        
        while len(hash_str) < precision:
            if even:
                mid = (lon_range[0] + lon_range[1]) / 2
                if lon >= mid:
                    bits = (bits << 1) | 1
                    lon_range[0] = mid
                else:
                    bits = bits << 1
                    lon_range[1] = mid
            else:
                mid = (lat_range[0] + lat_range[1]) / 2
                if lat >= mid:
                    bits = (bits << 1) | 1
                    lat_range[0] = mid
                else:
                    bits = bits << 1
                    lat_range[1] = mid
            
            even = not even
            bit_count += 1
            
            if bit_count == 5:
                hash_str.append(cls.BASE32[bits])
                bits = 0
                bit_count = 0
        
        return ''.join(hash_str)
    
    @classmethod
    def decode(cls, geohash_str):
        """Geohash 문자열을 위도/경도 범위로 디코딩"""
        lat_range = [-90.0, 90.0]
        lon_range = [-180.0, 180.0]
        even = True
        
        for char in geohash_str:
            value = cls.DECODE_MAP[char]
            for bit_pos in range(4, -1, -1):  # 5비트씩 처리
                bit = (value >> bit_pos) & 1
                if even:
                    mid = (lon_range[0] + lon_range[1]) / 2
                    if bit:
                        lon_range[0] = mid
                    else:
                        lon_range[1] = mid
                else:
                    mid = (lat_range[0] + lat_range[1]) / 2
                    if bit:
                        lat_range[0] = mid
                    else:
                        lat_range[1] = mid
                even = not even
        
        lat = (lat_range[0] + lat_range[1]) / 2
        lon = (lon_range[0] + lon_range[1]) / 2
        return lat, lon
    
    @classmethod
    def neighbors(cls, geohash_str):
        """인접한 8개의 Geohash 셀 반환"""
        lat, lon = cls.decode(geohash_str)
        precision = len(geohash_str)
        
        # 셀 크기 계산
        lat_err = 90.0 / (2 ** (precision * 5 // 2))
        lon_err = 180.0 / (2 ** ((precision * 5 + 1) // 2))
        
        neighbors = {}
        for name, dlat, dlon in [
            ('N', lat_err*2, 0), ('S', -lat_err*2, 0),
            ('E', 0, lon_err*2), ('W', 0, -lon_err*2),
            ('NE', lat_err*2, lon_err*2), ('NW', lat_err*2, -lon_err*2),
            ('SE', -lat_err*2, lon_err*2), ('SW', -lat_err*2, -lon_err*2),
        ]:
            nlat = max(-90, min(90, lat + dlat))
            nlon = ((lon + dlon + 180) % 360) - 180
            neighbors[name] = cls.encode(nlat, nlon, precision)
        
        return neighbors


# 서울 시청
geohash = Geohash.encode(37.5665, 126.9780, precision=6)
print(f"서울 시청 Geohash: {geohash}")  # wydm9q 근방

decoded_lat, decoded_lon = Geohash.decode(geohash)
print(f"디코딩: ({decoded_lat:.4f}, {decoded_lon:.4f})")

neighbors = Geohash.neighbors(geohash)
print(f"북쪽 이웃: {neighbors['N']}")

# 근접 검색: 같은 prefix를 가진 해시들이 인접
print(f"\n같은 4글자 prefix '{geohash[:4]}'를 가진 셀들은 서로 인접합니다.")
```

### Geohash 정밀도 테이블

| 정밀도 | 셀 크기 (대략) | 용도 |
|---|---|---|
| 1 | 5,009 × 4,992 km | 대륙 단위 |
| 3 | 156 × 156 km | 도시 단위 |
| 5 | 4.9 × 4.9 km | 구/동 단위 |
| 6 | 1.2 × 0.6 km | 거리/블록 |
| 7 | 153 × 153 m | 건물 단위 |
| 9 | 4.8 × 4.8 m | 문 앞 |
| 12 | 3.7 × 1.9 cm | 최고 정밀도 |

### Geohash의 한계: 경계 문제

Geohash의 가장 큰 문제는 **경계 인접성 보장 불가**입니다. 공간적으로 매우 가까운 두 점이 서로 다른 prefix를 가질 수 있습니다. 특히 경도 ±180° 경계와 위도 0° 경계 근처에서 심각합니다.

이를 해결하기 위해 반드시 **8방향 이웃 셀을 함께 검색**해야 합니다.

## Google S2: 구를 정육면체로 펼치기

Google S2는 지구 구면(sphere)을 정육면체(cube)의 6개 면으로 투영하고, 각 면을 **힐버트 곡선(Hilbert Curve)**으로 인덱싱합니다. Geohash보다 훨씬 균일한 셀 크기와 우수한 공간 지역성을 제공합니다.

```python
# Python s2geometry 라이브러리 사용 (pip install s2geometry)
import s2geometry as s2

# S2 셀 계산
def get_s2_cell(lat, lon, level):
    """
    level: 0~30 (30이 최고 정밀도, 약 0.48cm²)
    """
    latlng = s2.S2LatLng.FromDegrees(lat, lon)
    cell_id = s2.S2CellId(latlng)
    return cell_id.parent(level)

# 서울 시청
seoul_cell = get_s2_cell(37.5665, 126.9780, level=13)
print(f"S2 Cell ID: {seoul_cell.id()}")
print(f"토큰: {seoul_cell.ToToken()}")

# 반경 내 셀 커버링 계산
def get_covering_cells(lat, lon, radius_km, max_cells=8):
    """특정 반경을 덮는 S2 셀들 반환"""
    center = s2.S2LatLng.FromDegrees(lat, lon).ToPoint()
    # 구면에서의 각도로 변환 (지구 반지름 ≈ 6371km)
    angle = s2.S1Angle.Radians(radius_km / 6371.0)
    cap = s2.S2Cap(center, angle)
    
    coverer = s2.S2RegionCoverer()
    coverer.set_max_cells(max_cells)
    coverer.set_min_level(10)
    coverer.set_max_level(15)
    
    covering = s2.S2CellUnion()
    coverer.GetCovering(cap, covering)
    return covering.cell_ids()

# 서울 시청 반경 2km 내 커버링
cells = get_covering_cells(37.5665, 126.9780, radius_km=2)
print(f"\n반경 2km 커버링 셀 수: {len(cells)}")
for cell_id in cells:
    print(f"  {s2.S2CellId(cell_id).ToToken()}")

# 두 점 간 거리 계산
busan = s2.S2LatLng.FromDegrees(35.1796, 129.0756)
seoul = s2.S2LatLng.FromDegrees(37.5665, 126.9780)
distance_km = busan.GetDistance(seoul).radians() * 6371
print(f"\n서울-부산 거리: {distance_km:.1f} km")
```

### S2 셀의 계층적 구조

S2의 진짜 강점은 **30단계의 계층적 셀 구조**입니다. 각 레벨의 셀은 상위 레벨 셀 4개로 정확히 분할됩니다. Cell ID는 64비트 정수로, 비트 연산만으로 계층 탐색이 가능합니다.

```
Level 0:  1/6 지구 표면 (정육면체 1면)
Level 2:  약 106만 km²  
Level 8:  약 1,536 km²  (서울 전체보다 넓음)
Level 13: 약 1.5 km²    (동네 수준)
Level 20: 약 2,255 m²   (건물 수준)
Level 30: 약 0.48 cm²   (최고 정밀도)
```

## Uber H3: 육각형의 힘

Uber H3는 지구를 **정이십면체(icosahedron)**에 투영한 후 육각형 격자로 분할합니다. 핵심 혁신은 정사각형 대신 **육각형 셀**을 사용하는 것입니다.

### 왜 육각형인가?

정사각형 격자에서 이웃한 셀에는 두 종류가 있습니다:
- **변 공유**: 4개 이웃, 거리 = 1 단위
- **꼭짓점 공유**: 4개 이웃, 거리 = √2 ≈ 1.41 단위

이 거리 불균일성은 공간 분석에서 편향을 일으킵니다. 육각형은 **6개의 이웃 모두 동일한 거리**를 가집니다.

```python
# h3-py 라이브러리 사용 (pip install h3)
import h3

# 기본 사용
def demo_h3():
    # 서울 시청 좌표
    lat, lon = 37.5665, 126.9780
    
    # 해상도 9의 H3 셀 (약 0.1 km² = 330m × 330m 육각형)
    cell = h3.latlng_to_cell(lat, lon, res=9)
    print(f"H3 셀 (해상도 9): {cell}")
    
    # 셀 중심 좌표
    center = h3.cell_to_latlng(cell)
    print(f"셀 중심: {center}")
    
    # k-ring: 반경 k 이내의 모든 셀
    k = 2  # 2링 = 중심 + 6 + 12 = 19개 셀
    ring = h3.grid_disk(cell, k)
    print(f"2-ring 셀 수: {len(ring)}")  # 19
    
    # 해상도별 셀 면적
    for res in [6, 7, 8, 9, 10, 11]:
        area_km2 = h3.average_hexagon_area(res, unit='km^2')
        edge_km = h3.average_hexagon_edge_length(res, unit='km')
        print(f"해상도 {res}: 면적 {area_km2:.4f} km², 변 길이 {edge_km:.4f} km")
    
    # 계층 변환 (해상도 업/다운)
    parent = h3.cell_to_parent(cell, res=8)
    children = h3.cell_to_children(cell, res=10)
    print(f"\n상위 셀 (res=8): {parent}")
    print(f"하위 셀 수 (res=10): {len(children)}")  # 7
    
    # 폴리곤 → H3 셀 변환 (지오펜싱)
    # 서울 광화문광장 근방 폴리곤
    polygon_coords = [
        [37.5759, 126.9770],
        [37.5759, 126.9820],
        [37.5730, 126.9820],
        [37.5730, 126.9770],
    ]
    cells_in_polygon = h3.polygon_to_cells(
        h3.H3Shape([polygon_coords[0][::-1]]),  # lon, lat 순서
        res=10
    )
    print(f"\n광화문광장 폴리곤 내 H3 셀 수: {len(cells_in_polygon)}")
    
    return cell

demo_h3()
```

### 실제 Uber 사용 사례: 동적 가격 계산

```python
import h3
from collections import defaultdict
import math

class SurgePricingEngine:
    """H3 기반 동적 가격 계산 시뮬레이터"""
    
    RESOLUTION = 9  # 약 0.1 km² 셀
    
    def __init__(self):
        self.demand = defaultdict(int)    # 셀별 요청 수
        self.supply = defaultdict(int)    # 셀별 드라이버 수
    
    def record_ride_request(self, lat, lon):
        cell = h3.latlng_to_cell(lat, lon, self.RESOLUTION)
        self.demand[cell] += 1
    
    def record_driver_location(self, lat, lon):
        cell = h3.latlng_to_cell(lat, lon, self.RESOLUTION)
        self.supply[cell] += 1
    
    def get_surge_multiplier(self, lat, lon, radius_rings=2):
        """
        현재 위치의 서지 배수 계산
        주변 k-ring 내 수요/공급 비율 기반
        """
        center_cell = h3.latlng_to_cell(lat, lon, self.RESOLUTION)
        nearby_cells = h3.grid_disk(center_cell, radius_rings)
        
        total_demand = sum(self.demand[c] for c in nearby_cells)
        total_supply = sum(self.supply[c] for c in nearby_cells)
        
        if total_supply == 0:
            return 3.0  # 드라이버 없음 → 최대 서지
        
        ratio = total_demand / total_supply
        
        # 서지 배수 계산 (Uber 공개 로직 근사)
        if ratio <= 1.0:
            return 1.0
        elif ratio <= 2.0:
            return 1.0 + (ratio - 1.0) * 0.5
        else:
            return min(3.0, 1.5 + (ratio - 2.0) * 0.3)
    
    def get_heatmap(self):
        """수요 히트맵 데이터 생성"""
        all_cells = set(self.demand.keys()) | set(self.supply.keys())
        heatmap = []
        
        for cell in all_cells:
            lat, lon = h3.cell_to_latlng(cell)
            demand = self.demand[cell]
            supply = self.supply[cell]
            surge = demand / max(1, supply)
            heatmap.append({
                'cell': cell,
                'lat': lat,
                'lon': lon,
                'demand': demand,
                'supply': supply,
                'surge_ratio': surge
            })
        
        return sorted(heatmap, key=lambda x: x['surge_ratio'], reverse=True)


# 시뮬레이션
engine = SurgePricingEngine()

# 강남역 근처 수요 폭발 시뮬레이션
import random
for _ in range(50):
    lat = 37.4979 + random.gauss(0, 0.003)
    lon = 127.0276 + random.gauss(0, 0.003)
    engine.record_ride_request(lat, lon)

for _ in range(10):
    lat = 37.4979 + random.gauss(0, 0.005)
    lon = 127.0276 + random.gauss(0, 0.005)
    engine.record_driver_location(lat, lon)

surge = engine.get_surge_multiplier(37.4979, 127.0276)
print(f"강남역 현재 서지 배수: {surge:.2f}x")

heatmap = engine.get_heatmap()
print(f"\n상위 3개 핫스팟:")
for spot in heatmap[:3]:
    print(f"  셀 {spot['cell']}: 수요 {spot['demand']}, 공급 {spot['supply']}, "
          f"서지 비율 {spot['surge_ratio']:.2f}")
```

## 세 알고리즘 비교

| 특성 | Geohash | Google S2 | Uber H3 |
|---|---|---|---|
| 셀 형태 | 직사각형 | 정사각형 (구면 상) | 육각형 |
| 이웃 수 | 8 (거리 불균일) | 8 (거리 불균일) | 6 (균일) |
| 계층 구조 | 문자열 prefix | 64비트 ID | 64비트 ID |
| 극 지역 왜곡 | 심각 | 없음 | 거의 없음 |
| 구현 복잡도 | 낮음 | 높음 | 중간 |
| 주요 사용처 | PostgreSQL PostGIS, 간단한 POI 검색 | Google Maps, 폴리곤 지오펜싱 | Uber, 공간 통계 분석 |

## 주의사항과 팁

### 1. Geohash 경계 문제 항상 고려

반경 검색 시 중심 셀 하나만 검색하면 경계 근방의 결과가 누락됩니다. 반드시 주변 8개 이웃 셀을 포함하세요.

### 2. 해상도/정밀도 선택

실시간 매칭(택시, 배달)에는 해상도 8~9 (100~500m), 통계 분석에는 해상도 5~7 (수 km)이 적합합니다.

### 3. 데이터베이스 인덱싱 전략

```sql
-- PostgreSQL + PostGIS 예시
CREATE INDEX idx_location_geohash ON places (ST_GeoHash(geom, 7));
-- 검색
SELECT * FROM places WHERE ST_GeoHash(geom, 5) = 'wydm9';
```

Redis를 사용할 경우 `GEOADD`/`GEORADIUS` 명령은 내부적으로 52비트 Geohash를 사용합니다.

### 4. 성능 최적화

대규모 POI 검색 시스템에서는 Geohash prefix를 Redis Sorted Set에 저장하고, `ZRANGEBYLEX`로 같은 prefix를 가진 항목을 O(log N + M)에 검색하는 패턴이 효과적입니다.

공간 인덱싱의 선택은 워크로드에 달려 있습니다. 단순한 POI 검색이라면 Geohash로 충분하지만, 복잡한 지오펜싱이나 전 지구적 커버리지가 필요하다면 S2, 육각형 기반 공간 통계 분석이라면 H3가 최선입니다.

## 참고 자료
- [Wikipedia: Geohash](https://en.wikipedia.org/wiki/Geohash)
- [Google S2 Geometry Library 공식 사이트](https://s2geometry.io/)
- [Uber H3 공식 문서](https://h3geo.org/)
- [Wikipedia: Z-order curve](https://en.wikipedia.org/wiki/Z-order_curve)
