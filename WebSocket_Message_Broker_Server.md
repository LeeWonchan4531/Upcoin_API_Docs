## HTTP API 명세서: WebSocket Message Broker 서버

**Base URL:** `http://<서버_IP_주소>:<HTTP_STATUS_PORT>`
(기본 설정: `http://localhost:3000`)

---

### 1. 연결 상태 조회

지정된 `type` (예: "ticker", "orderbook")별 WebSocket 프록시 연결 및 클라이언트 상태를 조회합니다.

*   **Endpoint:** `/status`
*   **Method:** `GET`
*   **Description:** 현재 프록시 서버에 연결된 모든 `type`에 대한 상태 정보를 반환합니다. 각 `type`은 클라이언트가 WebSocket 연결 시 전송한 메시지의 `type` 필드 값에 해당합니다.

*   **Request:**
    *   **Headers:**
        *   `Accept: application/json` (권장)
    *   **Path Parameters:** 없음
    *   **Query Parameters:** 없음
    *   **Body:** 없음

*   **Response:**

    *   **`200 OK`**
        *   **Content-Type:** `application/json`
        *   **Body:**
            JSON 객체. 객체의 각 키는 클라이언트가 구독을 요청한 `type` (예: "ticker")이며, 값은 해당 `type`의 상태 정보를 담은 객체입니다.
            ```json
            {
              "<type1>": {
                "upbitConnected": <boolean>,
                "upbitConnecting": <boolean>,
                "upbitReconnecting": <boolean>,
                "clientCount": <integer>,
                "subscribedCodes": ["<string>", ...],
                "lastProcessingTime": <number>
              },
              "<type2>": {
                // ... 동일한 구조
              },
              // ... 추가 type들
            }
            ```
            **필드 설명:**
            *   `upbitConnected` (boolean): 프록시 서버가 해당 `type`에 대해 Upbit WebSocket 서버와 성공적으로 연결되어 있는지 여부.
            *   `upbitConnecting` (boolean): 프록시 서버가 해당 `type`에 대해 Upbit WebSocket 서버에 연결을 시도 중인지 여부.
            *   `upbitReconnecting` (boolean): 프록시 서버가 해당 `type`에 대해 Upbit WebSocket 서버와 연결이 끊어져 재연결을 시도 중인지 여부 (재연결 타이머가 설정된 경우 `true`).
            *   `clientCount` (integer): 해당 `type`으로 프록시 서버에 연결된 클라이언트 WebSocket 연결 수.
            *   `subscribedCodes` (array of strings): 해당 `type`에 대해 Upbit에 구독 요청한 코드 목록 (예: `["KRW-BTC", "KRW-ETH"]`). 구독 메시지가 없거나 `codes` 필드가 없는 경우 빈 배열 `[]`.
            *   `lastProcessingTime` (number): 해당 `type`으로 Upbit으로부터 마지막 메시지를 받아 클라이언트들에게 전송하는 데 걸린 시간 (밀리초 단위). 메시지를 처리한 적이 없으면 `0`.

        *   **예시 (성공):**
            ```json
            {
              "ticker": {
                "upbitConnected": true,
                "upbitConnecting": false,
                "upbitReconnecting": false,
                "clientCount": 2,
                "subscribedCodes": ["KRW-BTC", "KRW-ETH"],
                "lastProcessingTime": 0.52345
              },
              "orderbook": {
                "upbitConnected": false,
                "upbitConnecting": true,
                "upbitReconnecting": false,
                "clientCount": 1,
                "subscribedCodes": ["BTC-ETH"],
                "lastProcessingTime": 0
              }
            }
            ```
            **예시 (연결된 `type`이 없는 경우):**
            ```json
            {}
            ```

    *   **`404 Not Found`**
        *   **Content-Type:** `text/plain`
        *   **Body:**
            ```
            Not Found
            ```
        *   **설명:** `/status` 이외의 경로로 요청한 경우 발생합니다.

---

### 일반 오류 응답

*   **`404 Not Found`**: 요청한 리소스(경로)를 찾을 수 없을 때 반환됩니다.

---

이 명세서는 제공된 코드의 HTTP 서버 부분만을 다룹니다. WebSocket 프록시 자체의 프로토콜은 이 문서의 범위를 벗어납니다.
