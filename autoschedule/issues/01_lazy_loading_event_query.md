# CalendarEvent 조회 API LazyInitializationException 트러블슈팅

## 문제 상황

`/api/events/team/{teamId}` 및 `/api/events/{id}` 엔드포인트에서 500 Internal Server Error가 발생했습니다. 
프론트엔드에서 이벤트 목록 조회 및 상세 조회 시 서버 오류가 발생하여 캘린더에 이벤트가 표시되지 않는 문제가 있었습니다.

## 에러 메시지

```
GET http://localhost:8080/api/events/team/7 500 (Internal Server Error)
GET http://localhost:8080/api/events/160 500 (Internal Server Error)
```

백엔드 로그에서 확인된 예외:
```
org.hibernate.LazyInitializationException: could not initialize proxy [com.example.sbb.domain.Team#7] - no Session
```

## 원인 분석

### Lazy Loading + 트랜잭션 종료 문제

1. **엔티티 관계 설정**
   - `CalendarEvent` 엔티티는 `Team`과 `Owner(User)`와 `@ManyToOne(fetch = FetchType.LAZY)` 관계를 가지고 있습니다.
   - Lazy loading은 연관 객체에 접근할 때까지 실제 조회를 지연시킵니다.

2. **트랜잭션 범위 문제**
   - Repository 메서드가 트랜잭션 없이 실행되거나, 트랜잭션이 조기에 종료되는 경우
   - Service 메서드에서 `toResponse()` 변환 시점에 이미 Hibernate 세션이 닫혀있어어
   - `e.getTeam().getName()` 또는 `e.getOwner().getName()` 호출 시 LazyInitializationException 발생했습니다.

3. **문제 발생 시나리오**
   ```
   1. Repository에서 CalendarEvent 엔티티 조회 (Team, Owner는 프록시 객체)
   2. 트랜잭션 종료 → Hibernate 세션 닫힘
   3. Service의 toResponse()에서 e.getTeam().getName() 호출
   4. LazyInitializationException 발생 (세션이 없어서 프록시 초기화 불가)
   ```

## 변경전 코드 요약

### Repository (변경 전)
```java
public interface CalendarEventRepository extends JpaRepository<CalendarEvent, Long> {
    List<CalendarEvent> findByTeam_Id(Long teamId);
    Optional<CalendarEvent> findById(Long id);
}
```

### Service (변경 전)
```java
public List<CalendarEventResponse> findByTeam(Long teamId) {
    return calendarEventRepository.findByTeam_Id(teamId).stream()
        .map(this::toResponse)
        .collect(Collectors.toList());
}

private CalendarEventResponse toResponse(CalendarEvent e) {
    CalendarEventResponse r = new CalendarEventResponse();
    r.setId(e.getId());
    r.setTeamId(e.getTeam() != null ? e.getTeam().getId() : null);
    r.setTeamName(e.getTeam() != null ? e.getTeam().getName() : null);  //  여기서 LazyInitializationException 발생
    r.setOwnerId(e.getOwner() != null ? e.getOwner().getId() : null);
    r.setOwnerName(e.getOwner() != null ? e.getOwner().getName() : null);  //  여기서도 발생 가능
    // ... 나머지 필드 설정
    return r;
}
```

**문제점:**
- 엔티티를 조회한 후 `toResponse()`에서 연관 객체에 접근합니다.
- 트랜잭션이 종료된 후 Lazy loading 시도로 인한 예외 발생합니다.

## 해결 과정

### 1단계: JOIN FETCH 시도 (임시 해결책)
처음에는 JOIN FETCH를 사용하여 연관 객체를 즉시 로드하는 방식으로 해결을 시도했습니다:

```java
@Query("SELECT e FROM CalendarEvent e LEFT JOIN FETCH e.team LEFT JOIN FETCH e.owner WHERE e.team.id = :teamId")
List<CalendarEvent> findByTeam_Id(@Param("teamId") Long teamId);
```

하지만 이 방식은 여전히 엔티티를 거쳐야 하고, 트랜잭션 관리가 복잡해질 수 있습니다.

### 2단계: DTO Projection 적용 (최종 해결책)
**DTO Projection :** 엔티티를 거치지 않고, JPQL에서 직접 DTO로 조회

## 변경된 Repository 코드 (DTO 프로젝션 버전)

```java
package com.example.sbb.repository;

import com.example.sbb.domain.CalendarEvent;
import com.example.sbb.dto.response.CalendarEventResponse;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.OffsetDateTime;
import java.util.List;
import java.util.Optional;

public interface CalendarEventRepository extends JpaRepository<CalendarEvent, Long> {
    // DTO 프로젝션: 엔티티를 거치지 않고 바로 DTO로 조회 (Lazy loading 문제 원천 제거)
    @Query("""
        SELECT new com.example.sbb.dto.response.CalendarEventResponse(
            e.id,
            e.team.id,
            e.team.name,
            e.owner.id,
            e.owner.name,
            e.title,
            e.startsAt,
            e.endsAt,
            e.fixed,
            e.location,
            e.attendees,
            e.notes,
            e.recurrenceType,
            e.recurrenceEndDate,
            e.createdAt,
            e.updatedAt
        )
        FROM CalendarEvent e
        LEFT JOIN e.team
        LEFT JOIN e.owner
        WHERE e.id = :id
    """)
    Optional<CalendarEventResponse> findResponseById(@Param("id") Long id);
    
    @Query("""
        SELECT new com.example.sbb.dto.response.CalendarEventResponse(
            e.id,
            e.team.id,
            e.team.name,
            e.owner.id,
            e.owner.name,
            e.title,
            e.startsAt,
            e.endsAt,
            e.fixed,
            e.location,
            e.attendees,
            e.notes,
            e.recurrenceType,
            e.recurrenceEndDate,
            e.createdAt,
            e.updatedAt
        )
        FROM CalendarEvent e
        LEFT JOIN e.team
        LEFT JOIN e.owner
        WHERE e.team.id = :teamId
    """)
    List<CalendarEventResponse> findResponsesByTeamId(@Param("teamId") Long teamId);
    
    @Query("""
        SELECT new com.example.sbb.dto.response.CalendarEventResponse(
            e.id,
            e.team.id,
            e.team.name,
            e.owner.id,
            e.owner.name,
            e.title,
            e.startsAt,
            e.endsAt,
            e.fixed,
            e.location,
            e.attendees,
            e.notes,
            e.recurrenceType,
            e.recurrenceEndDate,
            e.createdAt,
            e.updatedAt
        )
        FROM CalendarEvent e
        LEFT JOIN e.team
        LEFT JOIN e.owner
        WHERE e.team.id = :teamId AND e.startsAt BETWEEN :start AND :end
    """)
    List<CalendarEventResponse> findResponsesByTeamIdAndRange(@Param("teamId") Long teamId, @Param("start") OffsetDateTime start, @Param("end") OffsetDateTime end);
    
    // findByOwner, findByOwnerAndRange도 동일한 패턴으로 구현
}
```

**핵심 변경사항:**
- `SELECT new com.example.sbb.dto.response.CalendarEventResponse(...)` 사용하여여
- 필요한 필드만 직접 조회하여 DTO 생성하고고
- 엔티티 객체를 생성하지 않습니다.

## 변경된 Service 코드

```java
@Transactional(readOnly = true)
public CalendarEventResponse findById(Long id) {
    return calendarEventRepository.findResponseById(id)
        .orElseThrow(() -> new IllegalArgumentException("이벤트를 찾을 수 없습니다: " + id));
}

@Transactional(readOnly = true)
public List<CalendarEventResponse> findByTeam(Long teamId) {
    return calendarEventRepository.findResponsesByTeamId(teamId);
}

@Transactional(readOnly = true)
public List<CalendarEventResponse> findByTeamAndRange(Long teamId, OffsetDateTime start, OffsetDateTime end) {
    return calendarEventRepository.findResponsesByTeamIdAndRange(teamId, start, end);
}
```

**변경사항:**
- `toResponse()` 변환 로직 제거했으며,
- Repository에서 바로 DTO 반환합니다.
- 코드가 훨씬 간결해졌습니다.

## DTO Response에 생성자 추가

DTO 프로젝션을 사용하기 위해 `CalendarEventResponse`에 생성자를 추가했습니다:

```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor  // ← JPQL DTO 프로젝션을 위해 추가
public class CalendarEventResponse {
    private Long id;
    private Long teamId;
    private String teamName;
    private Long ownerId;
    private String ownerName;
    // ... 나머지 필드
}
```

## 왜 이 방식이 더 좋은지 ..

### JOIN FETCH 대비 장점

| 항목 | JOIN FETCH | DTO Projection |
|------|-----------|----------------|
| **LazyInitializationException** | 트랜잭션 관리 필요 | 원천천 제거 |
| **성능** | 엔티티 전체 로드 | 필요한 필드만 조회 |
| **메모리 사용** | 엔티티 객체 생성 | DTO만 생성 |
| **코드 복잡도** | 엔티티 변환 로직 필요 | 변환 로직 불필요 |
| **유지보수성** | 엔티티 구조 변경 시 영향 | DTO만 수정 |
| **쿼리 최적화** | N+1 문제 가능성 | 한 번의 쿼리로 해결 |

### 구체적인 장점

1. **LazyInitializationException 원천천 제거**
   - 엔티티를 거치지 않으므로 Lazy loading이 발생하지 않습니다.
   - 트랜잭션 범위에 덜 의존적입니다. 

2. **성능 최적화**
   - 필요한 필드만 조회하여 네트워크 트래픽 감소이 감소합니다.
   - 엔티티 객체 생성 오버헤드 제거됩니다.
   - 메모리 사용량도 감소합니다. 

3. **코드 단순화**
   - `toResponse()` 같은 변환 메서드 불필요하며
   - Service 레이어가 더 간결해집니다. 

4. **유지보수성 향상**
   - API 레이어가 엔티티 구조에 덜 의존하며
   - 엔티티 변경 시 DTO만 수정하면 됩니다. 
   - API 계약이 명확해집니다.


## 재발 방지 전략

### 1. 코드 리뷰 체크리스트
- [ ] 조회 메서드에서 엔티티를 반환하는 경우 Lazy loading 가능성 확인
- [ ] `toResponse()` 같은 변환 메서드에서 연관 객체 접근 시 트랜잭션 범위 확인
- [ ] DTO 프로젝션 적용 가능 여부 검토

### 2. 개발 가이드라인
```
권장: DTO 프로젝션 사용
   - 조회 전용 API는 가능한 한 DTO 프로젝션 사용
   - 엔티티를 거치지 않고 바로 DTO로 조회

주의: JOIN FETCH 사용 시
   - @Transactional(readOnly = true) 필수
   - 트랜잭션 범위 내에서만 연관 객체 접근

금지: Lazy loading에 의존한 조회
   - 트랜잭션 없이 엔티티 조회 후 연관 객체 접근
```

### 3. 테스트 전략
- **통합 테스트**: 트랜잭션 없이 조회 API 호출하여 LazyInitializationException 발생 여부 확인
- **성능 테스트**: 대량 데이터 조회 시 메모리 사용량 및 쿼리 성능 측정



## 참고 자료

- [Spring Data JPA - Projections](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#projections)
- [Hibernate - LazyInitializationException](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#fetching-lazyinitializationexception)
- [JPA Best Practices - DTO Projection](https://vladmihalcea.com/the-best-way-to-map-a-projection-query-to-a-dto-with-jpa-and-hibernate/)

---

**작성일**: 2025-11-19  
**작성자**: 진소원
**관련 이슈**: CalendarEvent 조회 API 500 에러

