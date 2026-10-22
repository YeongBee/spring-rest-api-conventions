# Spring Boot 3.2.5 REST API Endpoint (@RequestMapping) 설계 컨벤션

## 1. 기본 URL 설계 원칙 (Best Practices)

1. **리소스는 복수형 명사(Plural Noun)를 사용한다.**
   - Good: `/api/v1/users`, `/api/v1/orders`
   - Bad: `/api/v1/user`, `/api/v1/order`
   - 예외: 단일 리소스(Singleton)인 경우 단수 허용 (`/api/v1/users/me/profile`)

2. **소문자(Lowercase)와 케밥 케이스(Kebab-case)만 사용한다.**
   - 경로(Path) 내 단어 구분은 하이픈(`-`)을 사용한다.
   - Good: `/api/v1/order-items`, `/api/v1/delivery-addresses`
   - Bad: `/api/v1/orderItems`, `/api/v1/order_items`

3. **계층 구조는 슬래시(`/`)로 표현하되 Depth는 최대 2~3단계로 제한한다.**
   - Good: `/api/v1/users/{userId}/orders`
   - Bad: `/api/v1/users/{userId}/orders/{orderId}/order-items/{itemId}`
   - 개선: 독립적인 PK가 있는 하위 리소스는 루트 경로(`/api/v1/order-items/{itemId}`)로 분리한다.

4. **식별자는 `@PathVariable`, 필터/정렬/페이징은 `@RequestParam`을 사용한다.**
   - 단건 리소스 식별: `GET /api/v1/orders/{orderId}`
   - 목록 필터링 및 페이징: `GET /api/v1/orders?status=PAID&page=0&size=20`
   - 참고: Query Parameter의 Key는 자바 객체(`@ModelAttribute`) 바인딩을 위해 카멜 케이스(`startDate`, `orderStatus`)를 사용한다.

5. **클래스 레벨과 메서드 레벨 어노테이션의 역할을 분리한다.**
   - 클래스 레벨: `@RequestMapping("/api/v1/...")`으로 공통 Prefix 선언
   - 메서드 레벨: `@GetMapping`, `@PostMapping`, `@PutMapping`, `@PatchMapping`, `@DeleteMapping` 사용

6. **CRUD로 표현하기 어려운 비즈니스 액션은 서브 리소스 또는 끝에 제어 동사를 사용한다.**
   - 상태 변경 액션: `POST /api/v1/orders/{orderId}/cancel`
   - 복잡한 조건의 조회(Body 필요 시): `POST /api/v1/orders/search`

---

## 2. 절대 사용하면 안 되는 규칙 (Anti-Patterns)

### ❌ 1) URL 끝에 슬래시(Trailing Slash, `/`) 붙이기 금지
- **Bad:** `@GetMapping("/users/")` 또는 클라이언트가 `/api/v1/users/`로 요청
- **Good:** `@GetMapping("/users")`
- **이유:** Spring Boot 3.0+부터 `TrailingSlashMatch` 설정이 기본 `false`로 변경(Deprecated)되어 `/users`와 `/users/`를 서로 다른 경로로 인식함 (`404 Not Found` 발생).

### ❌ 2) 경로 중간에 `**` 와일드카드 사용 금지
- **Bad:** `@GetMapping("/api/**/details")`
- **Good:** `@GetMapping("/api/*/details")` 또는 맨 끝에만 `@GetMapping("/files/**")` 사용
- **이유:** Spring Boot 3.x의 기본 매처인 `PathPatternParser`는 경로 중간에 `**`가 위치하면 서버 실행 시 `PatternParseException`을 발생시킴.

### ❌ 3) URL에 행위(CRUD 동사) 포함 금지
- **Bad:** `GET /api/v1/getUsers`, `POST /api/v1/createOrder`, `DELETE /api/v1/deleteOrder/1`
- **Good:** `GET /api/v1/users`, `POST /api/v1/orders`, `DELETE /api/v1/orders/1`
- **이유:** 행위는 HTTP Method(`GET`, `POST`, `PUT`, `PATCH`, `DELETE`)로 표현해야 함.

### ❌ 4) 경로(PathVariable)에 개인정보 및 민감 정보 포함 금지
- **Bad:** `GET /api/v1/users/010-1234-5678`, `GET /api/v1/members/user@email.com`
- **Good:** 내부 식별자(ID, UUID)를 사용하거나 `POST /api/v1/users/search`의 Request Body로 전달
- **이유:** URL은 웹 서버/프록시 로그 및 브라우저 히스토리에 평문으로 기록되어 보안 사고의 원인이 됨.

### ❌ 5) 파일 확장자(`.do`, `.json`, `.xml`) 사용 금지
- **Bad:** `/api/v1/users.json`, `/selectOrderList.do`
- **Good:** `/api/v1/users` (응답 포맷은 `Accept` / `Content-Type` 헤더로 제어)

### ❌ 6) `GET` 요청에 `@RequestBody` 사용 및 상태 변경 로직 금지
- **Bad:** `GET /api/v1/orders`에 JSON Body 포함, 또는 `GET /api/v1/orders/1/cancel`로 취소 처리
- **Good:** 조회 조건이 크면 `POST /api/v1/orders/search`, 상태 변경은 `POST` 또는 `PATCH` 사용
- **이유:** `GET` Body는 프록시/방화벽에서 유실될 수 있으며, `GET`에 상태 변경을 두면 크롤러나 프리페칭에 의해 데이터가 오염됨.

---

## 3. Spring Boot 3.2.5 표준 컨트롤러 작성 예시

```java
package com.example.order.presentation;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    // 1. 목록 조회 (복수형 명사 + 필터링/페이징은 Query Parameter)
    // GET /api/v1/orders?status=PAID&page=0&size=20
    @GetMapping
    public ResponseEntity<Page<OrderResponse>> getOrders(
            @RequestParam(name = "status", required = false) String status,
            @PageableDefault(size = 20) Pageable pageable
    ) {
        return ResponseEntity.ok(Page.empty());
    }

    // 2. 단건 조회 (식별자는 PathVariable 명시)
    // GET /api/v1/orders/{orderId}
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderResponse> getOrder(
            @PathVariable("orderId") Long orderId
    ) {
        return ResponseEntity.ok(new OrderResponse(orderId));
    }

    // 3. 리소스 생성 (201 Created 반환)
    // POST /api/v1/orders
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody OrderCreateRequest request
    ) {
        return ResponseEntity.status(HttpStatus.CREATED).body(new OrderResponse(1L));
    }

    // 4. 하위 리소스 조회 (케밥 케이스 활용, Depth 2단계 유지)
    // GET /api/v1/orders/{orderId}/delivery-tracking
    @GetMapping("/{orderId}/delivery-tracking")
    public ResponseEntity<DeliveryTrackingResponse> getDeliveryTracking(
            @PathVariable("orderId") Long orderId
    ) {
        return ResponseEntity.ok(new DeliveryTrackingResponse());
    }

    // 5. 비즈니스 상태 변경 액션 (명확한 제어 동사 사용)
    // POST /api/v1/orders/{orderId}/cancel
    @PostMapping("/{orderId}/cancel")
    public ResponseEntity<Void> cancelOrder(
            @PathVariable("orderId") Long orderId,
            @RequestBody OrderCancelRequest request
    ) {
        return ResponseEntity.noContent().build();
    }
}
```
