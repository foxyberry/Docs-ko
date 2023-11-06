# Geographic Data 처리
[GeoJson](https://geojson.org/)
- GeoJSON은 다양한 지리적 데이터 구조를 인코딩하기 위한 형식입니다.
- GeoJSON은 Point, LineString, Polygon, MultiPoint, MultiLineString 및 MultiPolygon과 같은 도형 유형을 지원합니다.
- 추가 속성이 있는 기하학적 개체는 지형 개체입니다. 기능 세트는 FeatureCollection 개체에 포함되어 있습니다.
- [GeoJSON 표준화(RFC 7946)](https://datatracker.ietf.org/doc/html/rfc7946)

## ES에서의 처리 (8.10 버전 기준)
8.10 버전 기준으로 ES의 Mapping에는 Geo 처리를 위한 2가지 type이 정의 되어 있다.  
### [geo-point](https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-point.html)
- 위도, 경도의 쌍

### [geo-shape](https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-shape.html)
- geo_shape 데이터 유형은 직사각형, 선, 다각형과 같은 임의의 지형 형태에 대한 색인 생성 및 검색을 용이하게 한다.
- 인덱싱되는 데이터에 Point 이외의 Shapes가 포함되어 있는 경우 Geo-Shape를 사용해야 한다.
- 데이터에 Point만 포함된 경우 geo_point를 써도 되고,  geo_shape로 써도 된다.

## Spring Data Elasticsearch 에서의 처리 (5.1 버전 기준)
## 1) 데이터 타입 사용
6.1.2. Mapping Rules 문서를 참고하자

##### Geospatial Types 사용하기
Point 및 GeoPoint와 같은 지리공간 유형은 위도/경도 쌍으로 변환됩니다.
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

##### GeoJson Types 사용하기
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
아래 GeoJson의 구현 클래스를 사용할 수 있다. 
- GeoJsonPoint
- GeoJsonMultiPoint
- GeoJsonLineString
- GeoJsonMultiLineString
- GeoJsonPolygon
- GeoJsonMultiPolygon
- GeoJsonGeometryCollection

org.springframework.data.elasticsearch.core.geo 에 실제 GeoJson 관련 클래스가 정의 되어 있다.


### converter
org.springframework.data.elasticsearch.core.convert 에 정의 되어 있다.


## 2) Geo 관련 어노테이션
문서 6.1.1. Mapping Annotation Overview 를 참고하쟈.

패키지 : package org.springframework.data.elasticsearch.annotations 


ES에서 지리 정보를 나타내는 2가지 어노테이션 타입은 두가지 이다. (4.1 버전으로)
1. GeoPoint 
2. GeoShape
   - 4.1 버전부터 추가되었다.

### 기타. GeoPoint 속성과 GeoShape 속성의 구분

1. GeoPoint 속성 : 해당 필드 타입을 GeoPoint 타입으로 지정했거나, GeoPointField 어노테이션을 선언한 경우 GeoPoint로 구분한다.
2. GeoShape 속성 : GeoJson 인터페이스를 구현한 타입을 사용했거나, GeoShapeField 어노테이션을 선언한 경우 GeoShape로 구분한다.

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

위의 구별 함수를 실제로 쓰는 곳은 org.springframework.data.elasticsearch.core.index.MappingBuilder 클래스의 buildPropertyMapping 함수이다.
```java
if (property.isGeoPointProperty()) {
    applyGeoPointFieldMapping(propertiesNode, property);
    return;
}

if (property.isGeoShapeProperty()) {
    applyGeoShapeMapping(propertiesNode, property);
}
```

위처럼 isGeoPointProperty 확인하여 true일 경우, 아래 처럼 applyGeoPointFieldMapping()를 호출한다.  
위처럼 isGeoShapeProperty 체크하여 true일 경우, 아래 처럼 applyGeoShapeMapping()를 호출한다. 

```java
private void applyGeoPointFieldMapping(ObjectNode propertiesNode, ElasticsearchPersistentProperty property)
        throws IOException {
    
    propertiesNode.set(property.getFieldName(),
    objectMapper.createObjectNode().put(FIELD_PARAM_TYPE, TYPE_VALUE_GEO_POINT));
    
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