# Geographic Data 처리
[GeoJSON 표준화(RFC 7946)](https://datatracker.ietf.org/doc/html/rfc7946)

[GeoJson](https://geojson.org/)
- GeoJSON은 다양한 지리적 데이터 구조를 인코딩하기 위한 형식입니다.
- GeoJSON은 Point, LineString, Polygon, MultiPoint, MultiLineString 및 MultiPolygon과 같은 도형 유형을 지원합니다.
- 추가 속성이 있는 기하학적 개체는 지형 개체입니다. 기능 세트는 FeatureCollection 개체에 포함되어 있습니다.


## ES에서의 처리 (8.10 버전 기준)
8.10 버전 기준으로 ES의 Mapping에는 Geo 처리를 위한 2가지 type이 정의 되어 있다.  
### [geo-point](https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-point.html)
- 위도, 경도의 쌍

### [geo-shape](https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-shape.html)
- geo_shape 데이터 유형은 직사각형, 선, 다각형과 같은 임의의 지형 형태에 대한 색인 생성 및 검색을 용이하게 한다.
- 인덱싱되는 데이터에 Point 이외의 Shapes가 포함되어 있는 경우 Geo-Shape를 사용해야 한다.
- 데이터에 Point만 포함된 경우 geo_point를 써도 되고,  geo_shape로 써도 된다.

## Spring Data Elasticsearch 에서의 처리 (5.1 버전 기준)
## 관련 데이터 타입 사용
[6.1.2. Mapping Rules](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#elasticsearch.mapping.meta-model.rules)

### Geospatial Types 사용하기
- Point 및 GeoPoint와 같은 지리공간 유형은 위도/경도 쌍으로 변환됩니다.
```java
public class Address {
  String city, street;
  Point location;
}

{
  "city" : "Los Angeles",
  "street" : "2800 East Observatory Road",
  "location" : { "lat" : 34.118347, "lon" : -118.3026284 }
}
```

### GeoJson Types 사용하기
- Spring Data Elasticsearch는 GeoJson 인터페이스와 다양한 형상에 대한 구현을 제공하여 GeoJson 유형을 지원하고 있다.
- 위에서 설명한 GeoJson 사양에 따라 Elasticsearch 문서에 매핑될 수 있다.
- 엔터티의 속성을 GeoJson 중에 하나로 지정하면, 인덱스 매핑이 작성될 때 인덱스 매핑에 geo_shape로 지정된다.
```java
public class Address {

  String city, street;
  GeoJsonPoint location;
}

{
  "city": "Los Angeles",
  "street": "2800 East Observatory Road",
  "location": {
    "type": "Point",
    "coordinates": [-118.3026284, 34.118347]
  }
}
```
#### GeoJson 인터페이스를 구현한 구현 클래스의 종료 
아래 GeoJson의 구현 클래스를 사용할 수 있다.
org.springframework.data.elasticsearch.core.geo 에 실제 GeoJson 관련 클래스가 정의 되어 있다.

- GeoJsonPoint
- GeoJsonMultiPoint
- GeoJsonLineString
- GeoJsonMultiLineString
- GeoJsonPolygon
- GeoJsonMultiPolygon
- GeoJsonGeometryCollection



#### converter
org.springframework.data.elasticsearch.core.convert 에 정의 되어 있다.


## Geo 관련 어노테이션 사용하기

[문서 6.1.1. Mapping Annotation Overview](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#elasticsearch.mapping.meta-model.annotations)

패키지 : package org.springframework.data.elasticsearch.annotations 

### Geo 어노테이션 타입 
ES에서 지리 정보를 나타내는 2가지 어노테이션 타입은 두가지 이다. (4.1 버전으로)
1. GeoPoint 
2. GeoShape
   - 4.1 버전부터 추가되었다.

## Spring-data-elasticsearch에서 GeoPoint 속성과 GeoShape 속성의 구분
org.springframework.data.elasticsearch.core.mapping.SimpleElasticsearchPersistentProperty 클래스에서 GeoPoint 속성과 GeoShape를 구별하는 기준은 다음 아래처럼 구현되어 있다.
```java
 @Override
 public boolean isGeoPointProperty() { // GeoPoint 타입이거나, GeoPointField 어노테이션으로 선언되었을 경우
     return getActualType() == GeoPoint.class || isAnnotationPresent(GeoPointField.class);
 }

 @Override
 public boolean isGeoShapeProperty() { // GeoJson 클래스들의 구현체거나, GeoShapeField 어노테이션으로 선언되었을 경우 
     return GeoJson.class.isAssignableFrom(getActualType()) || isAnnotationPresent(GeoShapeField.class);
 }
```

### GeoPoint 속성 
해당 필드 타입을 GeoPoint 타입으로 지정했거나, GeoPointField 어노테이션을 선언한 경우 GeoPoint로 구분한다.

### GeoShape 속성 
GeoJson 인터페이스를 구현한 타입을 사용했거나, GeoShapeField 어노테이션을 선언한 경우 GeoShape로 구분한다.

### 구분된 속성이 적용되는 파트
위의 구별 함수를 실제로 쓰는 곳은 org.springframework.data.elasticsearch.core.index.MappingBuilder 클래스의 buildPropertyMapping 함수이다.


#### isGeoPointProperty 확인 및 적용 부분 
isGeoPointProperty 확인하여 true일 경우, 아래 처럼 applyGeoPointFieldMapping()를 호출한다.

```java
if (property.isGeoPointProperty()) {
    applyGeoPointFieldMapping(propertiesNode, property);
    return;
}
private void applyGeoPointFieldMapping(ObjectNode propertiesNode, ElasticsearchPersistentProperty property)
        throws IOException {
     propertiesNode.set(property.getFieldName(),
     objectMapper.createObjectNode().put(FIELD_PARAM_TYPE, TYPE_VALUE_GEO_POINT));
}
```

#### isGeoShapeProperty 확인 및 적용 부분

isGeoShapeProperty 체크하여 true일 경우, 아래 처럼 applyGeoShapeMapping()를 호출한다.

```java
if (property.isGeoShapeProperty()) {
    applyGeoShapeMapping(propertiesNode, property);
}

private void applyGeoShapeMapping(ObjectNode propertiesNode, ElasticsearchPersistentProperty property)
throws IOException {

   ObjectNode shapeNode = propertiesNode.putObject(property.getFieldName());
   
    // GeoShapeField 어노테이션이 선언되지 않은 경우는 GeoShapeField 속성들의 기본값을 따른다. 
    GeoShapeMappingParameters mappingParameters = GeoShapeMappingParameters
    .from(property.findAnnotation(GeoShapeField.class));
    mappingParameters.writeTypeAndParametersTo(shapeNode);

}

```

# Geo 관련 쿼리 작성 
## native query 이용하기 
package co.elastic.clients.elasticsearch._types.query_dsl; 안에 있는 Geo_xxx_Query 시리즈 이용


### GeoBoundingBoxQuery
https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-geo-bounding-box-query.html
- 경계 상자와 교차하는 geo_point 및 geo_shape 값을 일치시킵니다.
- 경계 상자와 교차하는 geo_point 값을 일치시키려면 geo_bounding_box 필터를 사용하십시오. 
- 상자를 정의하려면 반대쪽 두 모서리에 대한 지오포인트 값을 제공하세요.

##### Java 코드
```java
query.geoBoundingBox { geoBoundingBoxQuery: GeoBoundingBoxQuery.Builder ->
     geoBoundingBoxQuery.field("location")
         .boundingBox { geoBounds: GeoBounds.Builder ->
             geoBounds.tlbr { bl4: TopLeftBottomRightGeoBounds.Builder ->
                 bl4.topLeft { geoLocation: GeoLocation.Builder ->
                     geoLocation.coords(
                         listOf(30.0, 31.0)
                     )
                 }
                     .bottomRight { geoLocation: GeoLocation.Builder ->
                         geoLocation.coords(
                             listOf(32.0, 28.0)
                         )
                     }
             }
         }
 }
```

### GeoDistanceQuery
https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-geo-distance-query.html
- geopoint의 지정된 거리 내에서 geo_point 및 geo_shape 값을 일치시킵니다.
- geo_distance 필터를 사용하여 다른 geopoint의 지정된 거리 내에서 geo_point 값을 일치시킵니다.

##### Java 코드
```java
Query query = new GeoDistanceQuery(
   source,
   field,
   ((Number) value).doubleValue(),
   ((Point) geometry).getY(),
   ((Point) geometry).getX()
);
```
### GeoGridQuery
https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-geo-grid-query.html
- GeoGrid 집계의 그리드 셀과 교차하는 geo_point 및 geo_shape 값을 일치시킵니다.
- 쿼리는 버킷의 키를 제공하여 지오그리드 집계의 버킷 내부에 있는 문서를 일치시키도록 설계되었습니다.
- geohash 및 geotile 그리드의 경우 geo_point 및 geo_shape 필드에 쿼리를 사용할 수 있습니다. 
- geo_hex 그리드의 경우 geo_point 필드에만 사용할 수 있습니다.

```java
QueryBuilder queryBuilder = new GeoGridQueryBuilder("geometry").setGridId(
    GeoGridQueryBuilder.Grid.GEOHEX,
    bucket.getKeyAsString()
);
```

### ~~GeoPolygonPoints~~
- **Deprecated**
https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-geo-polygon-query.html

### GeoShapeQuery
https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-geo-shape-query.html
- geo_shape 또는 geo_point 유형을 사용하여 색인화된 문서를 필터링합니다.
- geo_shape 쿼리는 geo_shape 또는 geo_point 매핑과 동일한 인덱스를 사용하여 지정된 공간 관계(intersects, contained, within 또는 disjoint)를 사용하여 쿼리 모양과 관련된 모양을 가진 문서를 찾습니다.
- 쿼리는 전체 모양 정의를 제공하거나 다른 인덱스에 미리 인덱싱된 모양의 이름을 참조하여 쿼리 모양을 정의하는 두 가지 방법을 지원합니다. 두 형식 모두 아래에 예와 함께 정의되어 있습니다.

다음 쿼리는 Elasticsearch의 봉투 GeoJSON 확장을 사용하여 지점을 찾습니다.
```agsl
GET /example/_search
{
  "query": {
    "bool": {
      "must": {
        "match_all": {}
      },
      "filter": {
        "geo_shape": {
          "location": {
            "shape": {
              "type": "envelope",
              "coordinates": [ [ 13.0, 53.0 ], [ 14.0, 52.0 ] ]
            },
            "relation": "within"
          }
        }
      }
    }
  }
}
```
```agsl
POST /example/_search
{
    "from": 0,
    "query": {
        "geo_shape": {
            "geometry": {
                "shape": "POLYGON ((126.90727686146623 37.53054723433391, 126.90835778517184 37.53085785794988, 126.90949701621935 37.52790593739182, 126.90832966272394 37.52759509171798, 126.90727686146623 37.53054723433391))",
                "relation": "WITHIN"
            }
        }
    },
    "size": 10,
    "sort": [
        
    ],
    "track_scores": false,
    "version": true
}

```
#### Spatial relations
- INTERSECTS - (기본값) query geometry를 **교차하는** geo_shape나 geo_point 필드를 가진 모든 문서를 반환한다.  
- DISJOINT - query geometry를 **공통으로 하고 있지 않는** geo_shape나 geo_point 필드를 가진 모든 문서를 반환한다.
- WITHIN - query geometry **안에 있는** geo_shape나 geo_point 필드를 가진 모든 문서를 반환한다. 라인은 지원되지 않는다. 
- CONTAINS - query geometry **포함 하는** geo_shape나 geo_point 필드를 가진 모든 문서를 반환한다.

#### Ignore unmapped
true로 설정된 경우ignore_unmapped 옵션은 매핑되지 않은 필드를 무시하고 이 쿼리에 대한 어떤 문서와도 일치하지 않습니다.
이는 매핑이 다를 수 있는 여러 인덱스를 쿼리할 때 유용할 수 있습니다. false(기본값)로 설정하면 필드가 매핑되지 않은 경우 쿼리에서 예외가 발생합니다.

### Notes
데이터가 geo_shape 필드에서 모양 배열로 인덱싱되면 배열은 하나의 모양으로 처리됩니다. 이러한 이유로 다음 요청은 동일합니다.
```agsl
PUT /test/_doc/1
{
  "location": [
    {
      "coordinates": [46.25,20.14],
      "type": "point"
    },
    {
      "coordinates": [47.49,19.04],
      "type": "point"
    }
  ]
}
```

```agsl
PUT /test/_doc/1
{
  "location":
    {
      "coordinates": [[46.25,20.14],[47.49,19.04]],
      "type": "multipoint"
    }
}
```
geo_shape 쿼리는 geo_shape 필드가 기본 방향인 오른쪽(시계 반대 방향)을 사용한다고 가정합니다. 다각형 방향을 참조하세요.

##### Java 코드 

```kotlin
fun findAllByGeoShape(list: List<LatLonGeoLocation>): List<BuildingInfo> {

   val xx = list.map { it.lat() }.toDoubleArray()
   val yy = list.map { it.lon() }.toDoubleArray()

   val shape = JsonData.of(WellKnownText.toWKT(Polygon(LinearRing(xx, yy))))

   val nativeQuery = NativeQuery.builder().withQuery { query ->
      query.geoShape { geoShapeQuery ->
         geoShapeQuery
            .field("geometry")
            .shape { geoShapeFieldQuery ->
               geoShapeFieldQuery.relation(GeoShapeRelation.Within).shape(shape)
            }
      }
   }.withPageable(Pageable.ofSize(10)).build()

   val searchHint = elasticsearchOperations.search(nativeQuery, BuildingInfo::class.java)

   return searchHint.map { it.content }.toList()

}
```