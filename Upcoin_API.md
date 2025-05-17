### Upcoin API 요청 문서

### 기본 정보

*   **Base URL:** `https://api.upcoin.kr`
*   **인증 방식:**
    *   사용자 인증을 위해 **세션 기반 인증**을 사용합니다.
    *   로그인 후 브라우저 개발자 도구 등에서 **`PHPSESSID` 쿠키 값**을 확인하여 요청 시 `Cookie` 헤더에 포함해야 합니다.
    *   예: `Cookie: PHPSESSID=YOUR_PHPSESSID_VALUE`
*   **응답 형식:** JSON (`Content-Type: application/json; charset=utf-8`)
*   **성공 응답:** 일반적으로 `success: true` 필드를 포함하며, `data` 필드에 요청 결과를 담습니다.
*   **실패 응답:** 일반적으로 `success: false` 필드를 포함하며, `message` 필드에 오류 내용을 담습니다. `data` 필드는 `null`이거나 해당 API의 기본 빈 구조를 가질 수 있습니다.
*   **참고:** school_name, grade, class 필드는 교내 대회 용도로 사용되며, API 응답에는 포함되지만 사용하지 않습니다.

---

### 1. 현재 자산 조회 (Get Current Assets)

*   **설명:** 로그인된 사용자의 현재 보유 자산 (KRW, 암호화폐), 미체결 주문 내역, 자산 요약 정보를 조회합니다.
*   **URL:** `/accounts.php`
*   **Method:** `GET`
*   **인증:** **필수** (`PHPSESSID` 쿠키 필요)
*   **요청 파라미터:** 없음
*   **성공 응답 (200 OK):**
    ```json
    {
        "success": true,
        "message": "Wallet data calculated successfully.",
        "data": {
            "summary": {
                "krw_balance": 1500000.50, // 보유 KRW
                "total_crypto_value": 8650000.00, // 보유 암호화폐 평가액 (현재가 기준)
                "total_assets": 10150000.50, // 총 보유자산 (KRW + 암호화폐 평가액)
                "total_profit_loss": 150000.50, // 총 평가손익 (총 보유자산 - 초기자본금)
                "total_profit_loss_rate": 1.50 // 총 평가손익률 (%)
            },
            "holdings": [ // 보유 암호화폐 목록
                {
                    "symbol": "BTC", // 코인 심볼
                    "korean_name": "비트코인", // 코인 한글명
                    "balance": 0.1, // 보유 수량
                    "avg_buy_price": 85000000.0, // 매수 평균가
                    "current_price": 86500000.0, // 현재가
                    "total_value": 8650000.0, // 평가 금액 (현재가 * 보유수량)
                    "total_invested": 8500000.0, // 매수 금액 (매수평균가 * 보유수량)
                    "profit_loss": 150000.0, // 평가 손익 (평가금액 - 매수금액)
                    "profit_rate": 1.76 // 평가 손익률 (%)
                }
                // ... other holdings
            ],
            "open_orders": [ // 미체결 주문 목록
                {
                    "order_id": "buy_BTC_1678886400_abcdef12", // 주문 ID
                    "user_id": 123,
                    "school_name": "Default",
                    "grade": 0,
                    "class": 0,
                    "name": "홍길동",
                    "symbol": "BTC", // 코인 심볼
                    "order_type": "buy", // 주문 종류 (buy/sell)
                    "price": 86000000.0, // 주문 가격
                    "quantity": 0.05, // 주문 수량
                    "timestamp": 1678886400, // 주문 시간 (Unix Timestamp)
                    "korean_name": "비트코인", // 코인 한글명
                    "transaction_type_korean": "매수", // 주문 종류 (한글)
                    "estimated_total_krw": 4308600.0 // 예상 체결 금액 (수수료 포함)
                }
                // ... other open orders (sorted by timestamp ascending)
            ]
        }
    }
    ```
*   **오류 응답:**
    *   **401 Unauthorized:** 인증 실패
        ```json
        {
            "success": false,
            "message": "Authentication required.",
            "data": {
                "summary": null,
                "holdings": [],
                "open_orders": []
            }
        }
        ```
    *   **500 Internal Server Error:** 서버 내부 오류 (DB 연결 실패, 쿼리 준비 실패 등)
        ```json
        {
            "success": false,
            "message": "An error occurred: KRW balance prepare failed",
            "data": {
                "summary": null,
                "holdings": [],
                "open_orders": []
            }
        }
        ```

        ```json
        {
            "success": false,
            "message": "Could not fetch trade price for BTC",
            "data": {
                "summary": null,
                "holdings": [],
                "open_orders": []
            }
        }
        ```
        
        ```json
        {
            "success": false,
            "message": "Crypto holdings prepare failed",
            "data": {
                "summary": null,
                "holdings": [],
                "open_orders": []
            }
        }
        ```
    *   **503 Service Unavailable:** 명시적으로 반환하지 않지만, 미체결 주문 조회 등 외부 서비스 의존적 기능 실패 시
        ```json
        {
            "success": false, // 또는 true일 수 있으나, message로 구분
            "message": "Warning: Could not retrieve open orders. Data source unavailable.", 
            "data": { // 자산 정보는 성공적으로 가져왔을 수 있음
                "summary": { /* ... */ },
                "holdings": [ /* ... */ ],
                "open_orders": []
            }
        }
        ```
*   **cURL 예시:**
    ```bash
    curl -X GET "https://api.upcoin.kr/accounts.php" \
         -b "PHPSESSID=YOUR_PHPSESSID_VALUE" \
         -H "Content-Type: application/json"
    ```

---

### 2. 지정가 주문 취소 (Cancel Limit Order)

*   **설명:** 사용자가 등록한 지정가 미체결 주문을 취소합니다.
*   **URL:** `/cancel_order.php`
*   **Method:** `POST`
*   **인증:** **필수** (`PHPSESSID` 쿠키 필요)
*   **요청 파라미터 (form-data):**
    | 파라미터명    | 타입   | 필수 | 설명                                     |
    | ------------- | ------ | ---- | ---------------------------------------- |
    | `order_id`    | String | Y    | 취소할 주문의 고유 ID                    |
    | `symbol`      | String | Y    | 취소할 주문의 코인 심볼 (예: "BTC")      |
    | `order_type`  | String | Y    | 취소할 주문의 종류 ("buy" 또는 "sell") |
*   **성공 응답 (200 OK):**
    ```json
    {
        "success": true
    }
    ```
*   **오류 응답:**
    *   **401 Unauthorized:** 명시적 HTTP 상태코드를 반환하지 않으므로 메시지 참고
        ```json
        {
            "success": false,
            "error": "로그인이 필요합니다.",
            "data": null
        }
        ```
    *   **400 Bad Request:** 필수 파라미터 누락
        ```json
        {
            "success": false,
            "error": "잘못된 요청입니다.",
            "data": null
        }
        ```
    *   **404 Not Found:** 주문을 찾을 수 없음
        ```json
        {
            "success": false,
            "error": "주문을 찾을 수 없습니다.",
            "data": null
        }
        ```
    *   **500 Internal Server Error:** 주문 취소 처리 중 DB 또는 Redis 오류 발생
        ```json
        {
            "success": false,
            "error": "주문 취소 처리 중 오류: 잔액 복구 실패", // 실제 오류 메시지
            "data": null
        }
        ```
*   **cURL 예시:**
    ```bash
    curl -X POST "https://api.upcoin.kr/cancel_order.php" \
         -b "PHPSESSID=YOUR_PHPSESSID_VALUE" \
         -d "order_id=buy_BTC_1678886400_abcdef12" \
         -d "symbol=BTC" \
         -d "order_type=buy"
    ```

---

### 3. 지정가 매수 주문 (Limit Buy Order)

*   **설명:** 지정된 가격과 수량으로 암호화폐 매수 주문을 등록합니다.
*   **URL:** `/limit_buy.php`
*   **Method:** `POST`
*   **인증:** **필수** (`PHPSESSID` 쿠키 필요)
*   **요청 파라미터 (form-data):**
    | 파라미터명 | 타입   | 필수 | 설명                                      |
    | ---------- | ------ | ---- | ----------------------------------------- |
    | `code`     | String | Y    | 매수할 코인 심볼 (예: "BTC")               |
    | `quantity` | Number | Y    | 매수할 수량 (0보다 커야 함)               |
    | `price`    | Number | Y    | 매수 희망 가격 (단위: KRW, 0보다 커야 함) |
*   **성공 응답 (201 Created):**
    ```json
       {
        "success": true,
        "message": "Limit buy order placed successfully.",
        "data": {
            "order_id": "buy_BTC_1678886401_fedcba98", // 생성된 주문 ID
            "user_id": 123,
            "school_name": "Default",
            "grade": 0,
            "class": 0,
            "name": "홍길동",
            "symbol": "BTC",
            "order_type": "buy",
            "price": 85000000.0,
            "quantity": 0.01,
            "timestamp": 1678886401
        }
    }
    ```
*   **오류 응답:**
    *   **400 Bad Request:**
        *   필수 필드 누락: `{"success": false, "message": "Missing required fields: code, quantity, price.", "data": null}`
        *   잘못된 숫자 형식: `{"success": false, "message": "Invalid number format for quantity or price.", "data": null}`
        *   0 이하의 값: `{"success": false, "message": "Quantity and price must be greater than zero.", "data": null}`
        *   허용되지 않은 코인: `{"success": false, "message": "Invalid or disallowed coin code.", "data": null}`
        *   최소 주문 금액 미달: `{"success": false, "message": "Minimum order value is 5,000 KRW.", "data": null}`
        *   KRW 잔액 부족: `{"success": false, "message": "Insufficient KRW balance to place this limit order. Required: 1,002 KRW, Available: 0 KRW", "data": null}`
    *   **401 Unauthorized:** 인증 실패
        ```json
        {
            "success": false,
            "message": "Authentication required.",
            "data": null
        }
        ```
    *   **405 Method Not Allowed:** 잘못된 HTTP 메소드
        ```json
        {
            "success": false,
            "message": "Invalid request method. Only POST is allowed.",
            "data": null
        }
        ```
    *   **500 Internal Server Error:** DB 또는 Redis 오류
        *   DB 오류: `{"success": false, "message": "DB prepare failed (fetch balance)", "data": null}`
        *   Redis 연결 오류: `{"success": false, "message": "Order placement failed: Cannot connect to matching engine.", "data": null}`
        *   Redis 등록 오류: `{"success": false, "message": "Order placement failed: Could not register order with matching engine.", "data": null}`
*   **cURL 예시:**
    ```bash
    curl -X POST "https://api.upcoin.kr/limit_buy.php" \
         -b "PHPSESSID=YOUR_PHPSESSID_VALUE" \
         -d "code=BTC" \
         -d "quantity=0.01" \
         -d "price=85000000"
    ```

---

### 4. 지정가 매도 주문 (Limit Sell Order)

*   **설명:** 지정된 가격과 수량으로 보유한 암호화폐 매도 주문을 등록합니다.
*   **URL:** `/limit_sell.php`
*   **Method:** `POST`
*   **인증:** **필수** (`PHPSESSID` 쿠키 필요)
*   **요청 파라미터 (form-data):**
    | 파라미터명 | 타입   | 필수 | 설명                                      |
    | ---------- | ------ | ---- | ----------------------------------------- |
    | `code`     | String | Y    | 매도할 코인 심볼 (예: "BTC")               |
    | `quantity` | Number | Y    | 매도할 수량 (0보다 커야 함)               |
    | `price`    | Number | Y    | 매도 희망 가격 (단위: KRW, 0보다 커야 함) |
*   **성공 응답 (201 Created):**
    ```json
    {
        "success": true,
        "message": "Limit sell order placed successfully.",
        "data": {
            "order_id": "limit_sell_BTC_1678886402_12345678"
        }
    }
    ```
*   **오류 응답:**
    *   **400 Bad Request:**
        *   필수 필드 누락: `{"success": false, "message": "Missing required fields: code, quantity, price."}`
        *   잘못된 수량/가격: `{"success": false, "message": "Invalid quantity: Must be a positive number."}`
        *   허용되지 않은 코인: `{"success": false, "message": "Invalid coin symbol provided."}`
    *   **401 Unauthorized:** 인증 실패
        ```json
        {
            "success": false,
            "message": "Authentication required."
        }
        ```
    *   **405 Method Not Allowed:** 잘못된 HTTP 메소드
        ```json
        {
            "success": false,
            "message": "Invalid request method. Only POST is allowed."
        }
        ```
    *   **422 Unprocessable Entity:** 보유 수량 부족
        ```json
        {
            "success": false,
            "message": "Order failed: Insufficient coin balance.",
            "data": {
                "symbol": "BTC",
                "current_balance": 0.05, // 실제 보유량
                "required_quantity": 0.1  // 요청 수량
            }
        }
        ```
    *   **500 Internal Server Error:** DB 또는 Redis 오류
        *   DB 오류: `{"success": false, "message": "Database failed"}`
        *   Redis 연결 오류: `{"success": false, "message": "Order failed: Matching engine connection error."}`
*   **cURL 예시:**
    ```bash
    curl -X POST "https://api.upcoin.kr/limit_sell.php" \
         -b "PHPSESSID=YOUR_PHPSESSID_VALUE" \
         -d "code=BTC" \
         -d "quantity=0.1" \
         -d "price=88000000"
    ```

---

### 5. 캔들 차트 데이터 조회 (Get Candlestick Data)

*   **설명:** 지정된 기간과 간격에 맞는 코인의 캔들 차트 데이터를 조회합니다.
*   **URL:** `/load.php`
*   **Method:** `GET`
*   **인증:** **불필요**
*   **요청 파라미터 (Query String):**
    | 파라미터명   | 타입    | 필수 | 설명                                                                   |
    | ------------ | ------- | ---- | ---------------------------------------------------------------------- |
    | `start_time` | String  | Y    | 조회 시작 시간 (예: '2023-10-26 00:00:00')                                |
    | `end_time`   | String  | Y    | 조회 종료 시간 (예: '2023-10-27 00:00:00')                                |
    | `code`       | String  | Y    | 조회할 코인 심볼 (예: "BTC")                                              |
    | `interval`   | Integer | N    | 시간 간격(분). 기본값 1. 허용값: 1, 3, 5, 10, 15, 30, 60                   |
*   **성공 응답 (200 OK):**
    ```json
    [
        {
            "time": "2023-10-26 10:00:00", // 캔들 시작 시간 (UTC+9)
            "open": 85000000.0, // 시가
            "high": 85500000.0, // 고가
            "low": 84900000.0, // 저가
            "close": 85300000.0, // 종가
            "volume": 1.234 // 거래량
        },
        {
            "time": "2023-10-26 10:01:00",
            "open": 85300000.0,
            "high": 85400000.0,
            "low": 85100000.0,
            "close": 85250000.0,
            "volume": 0.567
        }
        // ... more candles (time ascending)
    ]
    ```
*   **오류 응답:** (이 API는 오류 시 HTTP 200을 반환하므로 에러 메시지 참고)
    *   **400 Bad Request:**
        *   필수 파라미터 누락:
         ```json
          {"error": "필수 파라미터 누락"}
         ```
        *   지원하지 않는 시간 간격:
         ```json
          {"error": "지원하지 않는 시간 간격입니다."}
         ```
        *   잘못된 코인 ID:
         ```json
          {"error": "잘못된 코인 ID"}
         ```
    *   **500 Internal Server Error:** DB 쿼리 준비/실행 오류
         ```json
          {"error": "데이터 로드 중 오류 발생"}
         ```

         ```json
          {"error": "현재 분봉 데이터 처리 중 오류 발생 (Redis)"}
         ```
*   **cURL 예시:**
    ```bash
    curl -X GET "https://api.upcoin.kr/load.php?start_time=2023-10-26%2000%3A00%3A00&end_time=2023-10-27%2000%3A00%3A00&code=BTC&interval=1" \
         -H "Accept: application/json"
    ```

---

### 6. 시장가 매수 주문 (Place Market Buy Order)

*   **설명:** 지정된 수량만큼 현재 시장 가격으로 즉시 암호화폐를 매수합니다.
*   **URL:** `/price_buy.php`
*   **Method:** `POST`
*   **인증:** **필수** (`PHPSESSID` 쿠키 필요)
*   **요청 파라미터 (form-data):**
    | 파라미터명 | 타입   | 필수 | 설명                       |
    | ---------- | ------ | ---- | -------------------------- |
    | `code`     | String | Y    | 매수할 코인 심볼 (예: "BTC") |
    | `quantity` | Number | Y    | 매수할 수량 (0보다 커야 함) |
*   **성공 응답 (200 OK):**
    ```json
    {
      "success": true,
      "message": "Market buy order successful.",
      "data": {
          "transaction_id": 5678, // 생성된 거래 ID
          "code": "BTC",
          "quantity_bought": 0.1, // 실제 매수된 수량
          "price_per_unit": 86500000.0, // 체결 가격 (단위: KRW)
          "cost_before_fee": 8650000.0, // 수수료 제외 금액
          "fee_paid": 17300.0, // 지불한 수수료
          "total_krw_deducted": 8667300.0, // 총 차감된 KRW
          "new_average_buy_price": 85750000.0 // 변경된 매수 평균가
      }
    }

    ```
*   **오류 응답:**
    *   **400 Bad Request:**
        *   필수 파라미터 누락: `{"success": false, "message": "Missing required parameters: code or quantity.", "data": null}`
        *   잘못된 수량: `{"success": false, "message": "Invalid quantity. Must be a positive number.", "data": null}`
        *   허용되지 않은 코인: `{"success": false, "message": "Invalid or unsupported coin code.", "data": null}`
        *   최소 주문 금액 미달: `{"success": false, "message": "Minimum purchase amount is 5,000 KRW (excluding fees). Your order amount: 1,000 KRW.", "data": null}`
        *   KRW 잔액 부족: `{"success": false, "message": "Insufficient KRW balance. Required: 1,002 KRW, Available: 0 KRW.", "data": null}`
    *   **401 Unauthorized:** 인증 실패
        ```json
        {
            "success": false,
            "message": "Authentication required.",
            "data": null
        }
        ```
    *   **405 Method Not Allowed:** 잘못된 HTTP 메소드
        ```json
        {
            "success": false,
            "message": "Invalid request method. Only POST is allowed.",
            "data": null
        }
        ```
    *   **503 Service Unavailable:** 현재 시장가 조회 실패
        ```json
        {
            "success": false,
            "message": "Failed to retrieve current market price for BTC.",
            "data": null
        }
        ```
    *   **500 Internal Server Error:** DB 오류
        `{"success": false, "message": "Prepare statement failed (fetch balances): [DB 에러]", "data": null}`
*   **cURL 예시:**
    ```bash
    curl -X POST "https://api.upcoin.kr/price_buy.php" \
         -b "PHPSESSID=YOUR_PHPSESSID_VALUE" \
         -d "code=BTC" \
         -d "quantity=0.1"
    ```

---

### 7. 시장가 매도 주문 (Place Market Sell Order)

*   **설명:** 지정된 수량만큼 현재 시장 가격으로 즉시 보유한 암호화폐를 매도합니다.
*   **URL:** `/price_sell.php`
*   **Method:** `POST`
*   **인증:** **필수** (`PHPSESSID` 쿠키 필요)
*   **요청 파라미터 (form-data):**
    | 파라미터명 | 타입   | 필수 | 설명                       |
    | ---------- | ------ | ---- | -------------------------- |
    | `code`     | String | Y    | 매도할 코인 심볼 (예: "BTC") |
    | `quantity` | Number | Y    | 매도할 수량 (0보다 커야 함) |
*   **성공 응답 (200 OK):**
    ```json
    {
        "success": true,
        "message": "Market sell order executed successfully.",
        "data": {
            "transaction_id": 5679, // 생성된 거래 ID
            "code": "BTC",
            "quantity_sold": 0.05, // 실제 매도된 수량
            "price_per_unit": 86400000.0, // 체결 가격 (단위: KRW)
            "total_value": 4320000.0, // 총 매도 금액 (수수료 제외 전)
            "fee": 8640.0, // 발생한 수수료
            "total_received": 4311360.0, // 실제 입금된 KRW (수수료 제외 후)
            "timestamp": 1678886403 // 체결 시간 (Unix Timestamp)
        }
    }
    ```
*   **오류 응답:**
    *   **400 Bad Request:**
        *   필수 파라미터 누락: `{"success": false, "message": "Missing required parameters: code, quantity.", "data": null}`
        *   잘못된 수량: `{"success": false, "message": "Invalid quantity. Must be a positive number.", "data": null}`
        *   허용되지 않은 코인: `{"success": false, "message": "Invalid or unsupported coin code.", "data": null}`
        *   보유 수량 부족: `{"success": false, "message": "Insufficient balance. Required: 0.1 BTC, Available: 0.05 BTC", "data": null}`
    *   **401 Unauthorized:** 인증 실패
        ```json
        {
            "success": false,
            "message": "Authentication required.",
            "data": null
        }
        ```
    *   **405 Method Not Allowed:** 잘못된 HTTP 메소드
        ```json
        {
            "success": false,
            "message": "Invalid request method. Only POST is allowed.",
            "data": null
        }
        ```
    *   **503 Service Unavailable:** 현재 시장가 조회 실패
        ```json
        {
            "success": false,
            "message": "Failed to retrieve current market price for BTC.",
            "data": null
        }
        ```
    *   **500 Internal Server Error:** DB 오류
        `{"success": false, "message": "Database prepare error (get balance): [DB 에러]", "data": null}`
*   **cURL 예시:**
    ```bash
    curl -X POST "https://api.upcoin.kr/price_sell.php" \
         -b "PHPSESSID=YOUR_PHPSESSID_VALUE" \
         -d "code=BTC" \
         -d "quantity=0.05"
    ```

---

### 8. 거래 내역 조회 (Get Transaction History)

*   **설명:** 사용자의 과거 거래 내역을 페이지네이션하여 조회합니다.
*   **URL:** `/history.php`
*   **Method:** `GET`
*   **인증:** **필수** (`PHPSESSID` 쿠키 필요)
*   **요청 파라미터 (Query String):**
    | 파라미터명         | 타입    | 필수 | 설명                                         |
    | ------------------ | ------- | ---- | -------------------------------------------- |
    | `offset`           | Integer | N    | 조회 시작 위치 (기본값: 0).                  |
    | `records_per_page` | Integer | N    | 페이지 당 조회할 레코드 수 (기본값: 10).     |
    | `all`              | Boolean | N    | 모든 거래 내역 조회 (`true` 또는 `1`로 설정) |

*   **참고:**
    - `all` 파라미터가 `true` 또는 `1`로 설정된 경우, `offset`과 `records_per_page` 파라미터는 무시됩니다.
    - 많은 거래 내역이 있는 경우 `all` 파라미터 사용 시 응답 시간이 길어질 수 있습니다.
*   **성공 응답 (200 OK):**
    ```json
    [
        {
            "asset_symbol": "BTC", // 코인 심볼
            "transactionType": "sell", // 거래 유형 (sell/buy)
            "amount": 0.05, // 거래 수량
            "total": "4,311,360", // 거래 금액 (KRW, 포맷팅된 문자열)
            "transaction_time": "2023-10-27 10:30:05" // 거래 시간 (UTC+9)
        },
        {
            "asset_symbol": "ETH",
            "transactionType": "buy",
            "amount": 1.0,
            "total": "2,505,000",
            "transaction_time": "2023-10-26 15:20:10"
        }
        // ... more transactions (time descending)
    ]
    ```
*   **오류 응답:**
    *   **401 Unauthorized:** 인증 실패
        ```json
        { 
            "success": false,
            "message": "Authentication required.",
            "data": [] // 또는 null
        }
        ```
*   **cURL 예시:**
    ```bash
    curl -X GET "https://api.upcoin.kr/history.php?offset=0&records_per_page=10" \
         -b "PHPSESSID=YOUR_PHPSESSID_VALUE" \
         -H "Accept: application/json"
    ```

---

### 9. 코인 현재 시세 조회 (Get Coin Current Price)

*   **설명:** 지정된 코인 심볼의 현재 시세, 변동 상태, 변동폭 등의 정보를 조회하여 반환합니다.
*   **URL:** `/get_coin_price.php`
*   **Method:** `GET`
*   **인증:** **불필요**
*   **요청 파라미터 (Query String):**
    | 파라미터명 | 타입   | 필수 | 설명                                      |
    | ---------- | ------ | ---- | ----------------------------------------- |
    | `symbol`   | String | Y    | 조회할 코인의 심볼 (예: "BTC", "ETH" 등) |
*   **성공 응답 (200 OK):**
    ```json
    {
        "success": true,
        "message": "", // 성공 시 메시지는 비어있을 수 있음
        "data": {
            "symbol": "BTC",                 // 요청한 코인 심볼 (대문자)
            "korean_name": "비트코인",       // 코인 한글명 (Redis에 있는 경우)
            "trade_price": 65000000.0,       // 현재 거래 가격 (KRW)
            "change_status": "RISE",         // 가격 변동 상태 ("RISE": 상승, "FALL": 하락, "EVEN": 보합)
            "change_price": 500000.0,        // 전일 대비 변동 가격 (KRW)
            "change_rate": 0.00775           // 전일 대비 변동률 (예: 0.01은 1%)
        }
    }
    ```
    *   `change_status`, `change_price`, `change_rate` 필드는 Redis에 해당 데이터가 있는 경우에만 반환될 수 있습니다. (코드 상에서는 `null`로 처리됨)
*   **오류 응답:**
    *   **400 Bad Request:** 필수 파라미터 누락 또는 잘못된 코인 심볼 형식
        ```json
        {
            "success": false,
            "message": "Coin symbol is required.", // 또는 "Invalid or unsupported coin symbol."
            "data": null
        }
        ```
    *   **404 Not Found:** 요청한 코인 심볼에 대한 데이터를 찾을 수 없음
        ```json
        {
            "success": false,
            "message": "Coin data not found for symbol: XXX",
            "data": null
        }
        ```
    *   **503 Service Unavailable:** 데이터 소스(Redis) 연결 오류 또는 데이터 조회 실패
        ```json
        {
            "success": false,
            "message": "Data source error: Could not retrieve coin data.",
            "data": null
        }
        ```
    *   **500 Internal Server Error:** 서버 내부의 예기치 않은 오류 발생
        ```json
        {
            "success": false,
            "message": "An unexpected error occurred.",
            "data": null
        }
        ```
*   **cURL 예시:**
    ```bash
    # 비트코인(BTC) 시세 조회
    curl -X GET "https://api.upcoin.kr/get_coin_price.php?symbol=BTC" \
         -H "Accept: application/json"

    # 이더리움(ETH) 시세 조회
    curl -X GET "https://api.upcoin.kr/get_coin_price.php?symbol=eth" \
         -H "Accept: application/json"
    ```

---
