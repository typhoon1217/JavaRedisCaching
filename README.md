# JavaRedisCaching

JavaRedisCaching Implementation without @SpingCaching 

```
@Override
public ApiResponse<RegionResponseDTO> getChildrenRegions(Long cortaNo) {
    String cacheKey = childrenRegionsCacheKeyPrefix + cortaNo;
    List<FlatRegionDTO> childrenRegions = getCachedFlatRegions(cacheKey, 
        () -> regionRepository.findByParentCortarNo(cortaNo).stream()
                .map(FlatRegionDTO::from)
                .collect(Collectors.toList())
    );
    
    if (childrenRegions.isEmpty()) {
        throw new RegionException(RegionErrorCode.REGION_NOT_FOUND);
    }
    
    return ApiResponse.ok("자식 지역 리스트 조회 성공", new RegionResponseDTO(childrenRegions));
}



private List<FlatRegionDTO> getCachedFlatRegions(String cacheKey, RegionSupplier<List<FlatRegionDTO>> regionSupplier) {
    // 1. 캐시에서 데이터 조회 시도
    try {
        String cachedData = redisTemplate.opsForValue().get(cacheKey);
        if (cachedData != null) {
            return objectMapper.readValue(cachedData, new TypeReference<List<FlatRegionDTO>>() {});
        }
    } catch (Exception e) {
        log.warn("Redis 캐시 접근/역직렬화 실패: {}", cacheKey, e);
        // 캐시 조회 실패시 무시하고 DB 조회로 진행
    }
    
    // 2. DB에서 데이터 조회
    List<FlatRegionDTO> regions;
    try {
        regions = regionSupplier.get();
    } catch (Exception e) {
        log.error("데이터베이스 조회 실패", e);
        throw new RegionException(RegionErrorCode.CACHE_ACCESS_UNAVAILABLE);
    }
    
    // 3. 조회된 데이터 캐싱
    if (regions != null && !regions.isEmpty()) {
        try {
            redisTemplate.opsForValue().set(
                cacheKey, 
                objectMapper.writeValueAsString(regions), 
                60, 
                TimeUnit.DAYS
            );
        } catch (Exception e) {
            log.warn("Redis 캐싱 실패: {}", cacheKey, e);
            // 캐싱 실패해도 데이터 반환에는 영향 없음
        }
    }
    
    return regions;
}
```

스프링 캐싱(`@SpringCaching`) 대신 위와 같은 커스텀 Java Redis 캐싱 구현을 사용하는 장점은 다음과 같습니다:

1. **세밀한 캐싱 제어**  
   - 커스텀 구현은 캐시 조회, 저장, 직렬화/역직렬화 로직을 완전히 제어할 수 있어, 특정 비즈니스 요구사항에 맞춘 최적화가 가능합니다. 예를 들어, 캐시 키 생성 방식, TTL(Time-To-Live) 설정, 예외 처리 등을 세밀하게 조정할 수 있습니다.
   - `@SpringCaching`은 어노테이션 기반으로 동작하므로, 복잡한 캐싱 로직이나 조건을 구현하려면 추가적인 설정이나 커스텀 코드가 필요할 수 있습니다.

2. **Redis 전용 최적화**  
   - Redis의 고유 기능(예: `opsForValue`, `opsForHash`, TTL 설정 등)을 직접 활용하여 Redis의 성능을 최대한 끌어낼 수 있습니다.
   - `@SpringCaching`은 추상화된 캐싱 레이어를 제공하므로, Redis 특화 기능(예: Lua 스크립트, 트랜잭션 등)을 활용하려면 별도 설정이 필요하거나 제한적일 수 있습니다.

3. **명시적 예외 처리**  
   - 캐시 조회 실패, 직렬화/역직렬화 오류, Redis 연결 문제 등에 대한 명시적 예외 처리가 가능합니다. 위 코드에서는 캐시 조회 실패 시 DB로 폴백하고, 캐싱 실패 시에도 데이터 반환을 보장합니다.
   - `@SpringCaching`은 예외 처리가 추상화되어 있어, 특정 상황에서의 동작을 커스터마이징하려면 추가적인 `CacheErrorHandler` 구현이 필요합니다.

4. **캐시 데이터 직렬화/역직렬화 제어**  
   - `ObjectMapper`를 사용해 JSON 직렬화/역직렬화를 직접 관리하므로, 데이터 구조 변경이나 특정 필드 제외 등 직렬화 방식을 유연하게 조정할 수 있습니다.
   - `@SpringCaching`은 기본적으로 Java 직렬화를 사용하거나, 별도의 `CacheSerializer`를 설정해야 하므로 추가적인 설정이 필요할 수 있습니다.

5. **캐시 무효화 및 관리 용이성**  
   - 캐시 키 생성 로직(`childrenRegionsCacheKeyPrefix + cortaNo`)을 직접 관리하므로, 특정 조건에서 캐시를 무효화하거나 업데이트하는 로직을 쉽게 추가할 수 있습니다.
   - `@SpringCaching`은 캐시 무효화가 `@CacheEvict`나 조건 기반으로 제한적이어서 복잡한 무효화 로직은 별도 구현이 필요합니다.

6. **의존성 감소**  
   - 스프링 캐싱 모듈에 의존하지 않으므로, 스프링 캐싱 관련 설정(`CacheManager`, 어노테이션 등)을 관리할 필요가 없어 코드와 설정이 간소화됩니다.
   - Redis 클라이언트(`redisTemplate`)만 있으면 되므로, 스프링 캐싱 추상화 레이어 없이 가볍게 구현 가능합니다.

7. **디버깅 및 로깅 용이성**  
   - 캐시 조회, 저장, 실패 시 명시적으로 로깅(`log.warn`, `log.error`)하여 디버깅이 용이합니다. 문제 발생 시 정확한 원인을 파악하고 대응하기 쉽습니다.
   - `@SpringCaching`은 캐싱 동작이 프레임워크 내부에서 처리되므로, 디버깅을 위해 추가적인 로깅 설정이 필요할 수 있습니다.

8. **비즈니스 로직과의 통합성**  
   - 캐싱 로직이 비즈니스 로직(`getChildrenRegions`)과 밀접하게 통합되어 있어, 캐시와 데이터 조회 간의 일관성을 유지하기 쉽습니다. 예를 들어, 빈 결과 처리나 특정 조건에 따른 캐싱 여부를 쉽게 조정할 수 있습니다.
   - `@SpringCaching`은 메서드 레벨에서 캐싱을 적용하므로, 복잡한 비즈니스 로직과 캐싱 로직을 분리하기 어려울 수 있습니다.

### 요약
커스텀 Java Redis 캐싱은 `@SpringCaching` 대비 세밀한 제어, Redis 전용 최적화, 명시적 예외 처리, 직렬화 제어, 캐시 관리 용이성, 의존성 감소, 디버깅 편의성, 비즈니스 로직 통합성에서 장점을 가집니다. 특히, 복잡한 캐싱 요구사항이나 Redis의 고급 기능을 활용해야 하는 경우 적합합니다. 반면, `@SpringCaching`은 간단한 캐싱 시나리오에서 설정이 간편하고 추상화된 접근을 제공하는 장점이 있으므로, 프로젝트의 요구사항에 따라 선택해야 합니다.
