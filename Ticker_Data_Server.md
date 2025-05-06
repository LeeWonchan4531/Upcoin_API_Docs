## HTTP API 요청 명세서

이 명세서는 Node.js 암호화폐(업비트) 데이터 처리 서버의 HTTP API 엔드포인트에 대한 명세서입니다. API 서버는 포트 `8083`에서 실행됩니다.

---

### 공통 사항

*   **기본 URL**: `http://localhost:8083` (실제 배포 환경에 따라 변경될 수 있습니다.)
*   **CORS**: 모든 출처(`*`)에 대해 GET, POST, OPTIONS, DELETE 메소드 및 `Content-Type` 헤더를 허용하도록 설정되어 있습니다.
*   **응답 형식**: 대부분의 API는 JSON 형식으로 응답합니다. 성공 시 `success: true`와 함께 데이터를, 실패 시 `success: false`와 함께 에러 `message`를 포함합니다.

---

### 1. 시스템 상태 확인 API

**1.1. 현재 시스템 상태 조회**

*   **Endpoint**: `/status`
*   **Method**: `GET`
*   **Description**: 현재 WebSocket 서버에 연결된 클라이언트 수와 각 클라이언트의 연결 시간, 최근 메시지 처리 시간 평균 등의 상태 정보를 반환합니다.
*   **Request**:
    *   Headers: 없음
    *   Body: 없음
*   **Response**:
    *   **Success (200 OK)**:
        ```json
        {
          "connectedClients": 1, // 현재 연결된 WebSocket 클라이언트 수
          "clients": [
            {
              "connectionTime": "2023-10-27T10:00:00.000Z", // 클라이언트 연결 시간
              "averageMessageProcessingTime": 5.25 // 해당 클라이언트의 최근 10개 메시지 평균 처리 시간 (ms)
            }
            // 추가 클라이언트 정보...
          ]
        }
        ```
    *   **Failure**: 일반적인 서버 오류 발생 시 적절한 HTTP 상태 코드와 함께 오류 메시지를 반환할 수 있습니다 (예: 500 Internal Server Error).
*   **Example (cURL)**:
    ```bash
    curl -X GET http://localhost:8083/status
    ```

---

### 2. 구독 관리 API

이 API는 `subscription_manager.js` 모듈에 의해 처리되며, Upbit 프록시 WebSocket으로부터 수신할 암호화폐 목록을 동적으로 관리합니다.

**2.1. 현재 구독 중인 코인 목록 조회**

*   **Endpoint**: `/api/subscription`
*   **Method**: `GET`
*   **Description**: 현재 시스템이 Upbit 프록시로부터 시세 데이터를 구독 중인 모든 코인 심볼 목록을 반환합니다.
*   **Request**:
    *   Headers: 없음
    *   Body: 없음
*   **Response**:
    *   **Success (200 OK)**:
        ```json
        {
          "success": true,
          "subscriptions": [
            "KRW-BTC",
            "KRW-ETH",
            "KRW-XRP"
            // 기타 구독 중인 코인 심볼...
          ]
        }
        ```
    *   **Failure**: 일반적인 서버 오류 발생 시 적절한 HTTP 상태 코드와 함께 오류 메시지를 반환할 수 있습니다.
*   **Example (cURL)**:
    ```bash
    curl -X GET http://localhost:8083/api/subscription
    ```

**2.2. 새로운 코인 구독 추가**

*   **Endpoint**: `/api/subscription`
*   **Method**: `POST`
*   **Description**: 지정된 코인 심볼을 구독 목록에 추가합니다. 성공적으로 추가되면, 시스템은 해당 코인의 시세 데이터를 수신하기 시작합니다 (WebSocket 연결이 활성 상태일 경우 즉시, 그렇지 않으면 다음 연결 시).
*   **Request**:
    *   Headers:
        *   `Content-Type: application/json`
    *   Body (JSON):
        ```json
        {
          "symbol": "KRW-NEWCOIN" // 추가할 코인 심볼 (예: KRW-ADA)
        }
        ```
*   **Response**:
    *   **Success (200 OK)**:
        ```json
        {
          "success": true,
          "message": "Successfully subscribed to KRW-NEWCOIN"
        }
        ```
        또는 WebSocket 연결이 없는 경우:
        ```json
        {
          "success": true,
          "message": "Added KRW-NEWCOIN to queue. Will be active on next connection."
        }
        ```
    *   **Failure (400 Bad Request)**:
        *   요청 본문이 유효하지 않거나 `symbol` 필드가 누락된 경우:
            ```json
            {
              "success": false,
              "message": "Symbol is required"
            }
            ```
        *   코인 심볼 형식이 유효하지 않은 경우 (`KRW-XXX` 형식이어야 함):
            ```json
            {
              "success": false,
              "message": "Invalid coin symbol format: INVALID-SYMBOL"
            }
            ```
        *   이미 구독 중인 코인을 추가하려고 할 경우:
            ```json
            {
              "success": false,
              "message": "KRW-BTC is already subscribed"
            }
            ```
        *   WebSocket 구독 메시지 전송 실패 시 (드문 경우):
            ```json
            {
              "success": false,
              "message": "WebSocket error: <error_details>"
            }
            ```
*   **Example (cURL)**:
    ```bash
    curl -X POST -H "Content-Type: application/json" -d '{"symbol":"KRW-DOGE"}' http://localhost:8083/api/subscription
    ```

**2.3. 기존 코인 구독 제거**

*   **Endpoint**: `/api/subscription/{symbol}`
    *   예: `/api/subscription/KRW-XRP`
*   **Method**: `DELETE`
*   **Description**: 경로 매개변수로 지정된 코인 심볼을 구독 목록에서 제거합니다. 시스템은 해당 코인의 시세 데이터 수신을 중단합니다 (필수 코인 제외).
*   **Request**:
    *   Path Parameters:
        *   `symbol`: 구독을 취소할 코인 심볼 (URL 인코딩된 문자열, 예: `KRW-XRP`).
    *   Headers: 없음
    *   Body: 없음
*   **Response**:
    *   **Success (200 OK)**:
        ```json
        {
          "success": true,
          "message": "Successfully unsubscribed from KRW-XRP"
        }
        ```
        또는 WebSocket 연결이 없는 경우:
        ```json
        {
          "success": true,
          "message": "Removed KRW-XRP from subscription list."
        }
        ```
    *   **Failure (400 Bad Request)**:
        *   구독 중이지 않은 코인을 제거하려고 할 경우:
            ```json
            {
              "success": false,
              "message": "KRW-NONEXISTENT is not currently subscribed"
            }
            ```
        *   필수 구독 코인(`DEFAULT_COINS`에 포함된 코인)을 제거하려고 할 경우:
            ```json
            {
              "success": false,
              "message": "Cannot unsubscribe from essential coin: KRW-BTC"
            }
            ```
        *   WebSocket 구독 메시지 전송 실패 시 (드문 경우):
            ```json
            {
              "success": false,
              "message": "WebSocket error: <error_details>"
            }
            ```
*   **Example (cURL)**:
    ```bash
    curl -X DELETE http://localhost:8083/api/subscription/KRW-DOGE
    ```

---

이 문서는 API의 현재 구현 상태를 기준으로 작성되었습니다. 기능 변경 또는 추가 시 업데이트될 수 있습니다.
