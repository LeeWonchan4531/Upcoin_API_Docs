## HTTP API 명세서
이 명세서는 Node.js 암호화폐(업비트) 호가(Orderbook) 데이터 처리 서버의 HTTP API 엔드포인트에 대한 명세서입니다. API 서버는 포트 8080에서 실행됩니다.

**기본 URL:** `http://localhost:8080` (또는 서버가 실행되는 호스트 및 포트)

**공통 헤더:**
*   `Access-Control-Allow-Origin: *`
*   `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE`
*   `Access-Control-Allow-Headers: Content-Type`
    *   이 헤더들은 서버에서 응답 시 설정되며, 클라이언트가 CORS 요청을 보낼 때 필요합니다.

**공통 응답 형식 (오류):**
대부분의 API 오류는 다음과 유사한 JSON 형식으로 응답합니다.

```json
{
  "success": false,
  "message": "오류에 대한 설명"
}
```

---

### 1. 시스템 상태 조회

*   **엔드포인트:** `/status`
*   **메서드:** `GET`
*   **설명:** 현재 서버의 상태 정보를 조회합니다. 연결된 클라이언트 수, 클라이언트별 정보, 업비트 메시지 평균 처리 시간, 캐시된 심볼 목록 및 활성 구독 목록을 포함합니다.
*   **요청:**
    *   바디: 없음
*   **응답:**
    *   **성공 (200 OK):**
        *   `Content-Type: application/json`
        *   바디 예시:
            ```json
            {
                "connectedClients": 2,
                "clients": [
                    {
                        "connectionTime": "2023-10-27T10:30:00.123Z",
                        "symbol": "KRW-BTC"
                    },
                    {
                        "connectionTime": "2023-10-27T10:32:15.456Z",
                        "symbol": "KRW-ETH"
                    }
                ],
                "upbitAverageMessageProcessingTime": 5.75, // ms 단위
                "cachedSymbols": ["KRW-BTC", "KRW-ETH", "KRW-XRP"],
                "activeSubscriptions": ["KRW-BTC", "KRW-ETH", "KRW-XRP", "KRW-NEO", "..."]
            }
            ```
            *   `connectionTime`: 클라이언트가 WebSocket에 연결된 시간 (ISO 8601 형식) 또는 `null` (연결 직후 아직 심볼을 보내지 않은 경우).
            *   `symbol`: 해당 클라이언트가 구독 중인 심볼 또는 `null`.
            *   `upbitAverageMessageProcessingTime`: 최근 10개 업비트 메시지의 평균 처리 시간 (밀리초).
            *   `cachedSymbols`: 현재 서버에 캐시되어 있는 호가 데이터의 심볼 목록.
            *   `activeSubscriptions`: 현재 `subscriptionManager`를 통해 업비트에 구독 요청된 심볼 목록.

---

### 2. 구독 관리 API

#### 2.1. 구독 목록 조회

*   **엔드포인트:** `/api/subscription`
*   **메서드:** `GET`
*   **설명:** 현재 `subscriptionManager`에 의해 관리되고 있는 (업비트에 구독 요청된) 모든 코인 심볼 목록을 조회합니다.
*   **요청:**
    *   바디: 없음
*   **응답:**
    *   **성공 (200 OK):**
        *   `Content-Type: application/json`
        *   바디 예시:
            ```json
            {
                "success": true,
                "subscriptions": ["KRW-BTC", "KRW-ETH", "KRW-NEO", "..."]
            }
            ```

#### 2.2. 새 구독 추가

*   **엔드포인트:** `/api/subscription`
*   **메서드:** `POST`
*   **설명:** 새로운 코인 심볼을 구독 목록에 추가합니다. 추가된 심볼은 즉시 (만약 업비트 WebSocket이 연결된 상태라면) 또는 다음 연결 시 업비트로 구독 요청됩니다.
*   **요청:**
    *   `Content-Type: application/json`
    *   바디:
        ```json
        {
            "symbol": "KRW-DOGE"
        }
        ```
        *   `symbol` (string, 필수): 추가할 코인 심볼 (예: "KRW-BTC"). `KRW-` 형식이어야 합니다.
*   **응답:**
    *   **성공 (200 OK):**
        *   `Content-Type: application/json`
        *   바디 예시 (즉시 구독 성공 시):
            ```json
            {
                "success": true,
                "message": "Successfully subscribed to KRW-DOGE"
            }
            ```
        *   바디 예시 (큐에 추가되었을 경우, 업비트 연결 안 되어 있을 시):
            ```json
            {
                "success": true,
                "message": "Added KRW-DOGE to queue. Will be active on next connection."
            }
            ```
    *   **실패 (400 Bad Request):**
        *   `Content-Type: application/json`
        *   `symbol` 누락 시:
            ```json
            {
                "success": false,
                "message": "Symbol is required"
            }
            ```
        *   잘못된 JSON 데이터 형식 시:
            ```json
            {
                "success": false,
                "message": "Invalid JSON data"
            }
            ```
        *   `subscriptionManager`에서 발생한 오류 (예: 이미 구독 중, 잘못된 심볼 형식):
            ```json
            {
                "success": false,
                "message": "KRW-DOGE is already subscribed" // 또는 "Invalid coin symbol format: DOGE"
            }
            ```

#### 2.3. 구독 제거

*   **엔드포인트:** `/api/subscription/{symbol}`
*   **메서드:** `DELETE`
*   **설명:** 구독 목록에서 특정 코인 심볼을 제거합니다. 제거된 심볼은 즉시 (만약 업비트 WebSocket이 연결된 상태라면) 업비트 구독 목록에서 갱신(제외)됩니다.
*   **요청:**
    *   경로 매개변수:
        *   `symbol` (string, 필수): 제거할 코인 심볼 (예: `KRW-DOGE`). URL 인코딩이 필요할 수 있습니다.
    *   바디: 없음
*   **응답:**
    *   **성공 (200 OK):**
        *   `Content-Type: application/json`
        *   바디 예시 (즉시 구독 해제 성공 시):
            ```json
            {
                "success": true,
                "message": "Successfully unsubscribed from KRW-DOGE"
            }
            ```
        *   바디 예시 (목록에서만 제거, 업비트 연결 안 되어 있을 시):
            ```json
            {
                "success": true,
                "message": "Removed KRW-DOGE from subscription list."
            }
            ```
    *   **실패 (400 Bad Request):**
        *   `Content-Type: application/json`
        *   `subscriptionManager`에서 발생한 오류 (예: 구독 중이지 않은 심볼):
            ```json
            {
                "success": false,
                "message": "KRW-DOGE is not currently subscribed"
            }
            ```

---

### 404 Not Found

*   위에서 정의되지 않은 경로로 요청 시, 서버는 `404 Not Found` 상태 코드와 함께 `text/plain` 형식의 "Not Found" 메시지를 응답합니다.

---

**참고:**
*   클라이언트 WebSocket 연결은 HTTP API가 아닌 WebSocket 프로토콜을 통해 이루어집니다 (`ws://localhost:8080` 또는 `wss://`).
*   클라이언트 WebSocket은 연결 후 구독하고자 하는 심볼 문자열(예: "KRW-BTC")을 메시지로 전송하여 해당 심볼의 호가 데이터를 수신합니다.
