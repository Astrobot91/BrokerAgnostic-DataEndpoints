# Broker-Agnostic Data Endpoints

A FastAPI service that exposes a unified HTTP API for market-data (LTP, OHLC, full market quotes, historical candles, and master/instrument data) across multiple brokers. The project is designed to be broker-agnostic via a pluggable broker adapter system and includes a background token-rotation service to keep broker access tokens current. An Upstox adapter is implemented as an example.

Table of contents
- Features
- Architecture overview
- Prerequisites
- Secrets / configuration
- Installation
- Running the service
- API endpoints and examples
- Token rotation and auth automation
- Adding a new broker
- Logging & monitoring
- Testing
- Security & operational notes
- Contributing

Features
- Unified HTTP API (FastAPI) for multiple broker data sources.
- Broker abstraction (BaseBroker) + BrokerFactory for registering adapters.
- Upstox adapter with:
  - LTP, OHLC, full-market quote and historical candle retrieval.
  - Master instrument download and conversion (Polars).
  - Request chunking and simple rate-limit handling.
- Background TokenRotationService that periodically checks token health and rotates tokens via broker-specific rotators.
- Token storage and config via AWS Secrets Manager.
- CloudWatch logging via watchtower.

Architecture overview
- main.py — application entrypoint: builds FastAPI app, mounts router, starts TokenRotationService.
- api/endpoints.py — HTTP endpoints and dependency to initialize brokers.
- brokers/ — broker implementations and a factory:
  - base/ — BaseBroker and abstract interfaces.
  - upstox/ — Upstox-specific broker, authenticator, token rotator.
- services/token_rotation_service.py — periodic health checks and token rotation orchestration.
- logger.py — logger factory that writes to console and CloudWatch.
- requirements.txt — runtime and test dependencies.

Prerequisites
- Python 3.9+ (or compatible).
- AWS credentials (IAM role or environment variables) with permission to read/write Secrets Manager and write CloudWatch logs.
- Chrome/Chromium + matching ChromeDriver (or other WebDriver) available on PATH for Selenium-based authentication (used by UpstoxAuthenticator).
- Network access to Upstox API endpoints.

Secrets / configuration
This project loads broker configuration and tokens from AWS Secrets Manager. Environment variables (optional) and defaults used by the code:

- UPSTOX_CONFIG_SECRET_NAME (default: `my_upstox_config`)  
  - Expected to contain broker credentials & metadata as JSON. Example:
  ```json
  {
    "API_KEY": "your_upstox_api_key",
    "API_SECRET": "your_upstox_api_secret",
    "REDIRECT_URL": "https://your-redirect-url",
    "PHONE_NO": "91XXXXXXXXXX",
    "TOTP_KEY": "base32totpkey",
    "PIN_CODE": "your_pin_code"
  }
  ```

- UPSTOX_TOKEN_SECRET_NAME (default: `my_upstox_access_token`)  
  - Expected to contain current access token JSON. Example:
  ```json
  {
    "access_token": "current_access_token_string"
  }
  ```

Make sure these secrets exist in AWS Secrets Manager with the expected structure or update environment variables accordingly.

Installation
1. Clone the repository:
   ```bash
   git clone https://github.com/Astrobot91/BrokerAgnostic-DataEndpoints.git
   cd BrokerAgnostic-DataEndpoints
   ```
2. Create and activate a virtual environment:
   ```bash
   python -m venv venv
   source venv/bin/activate    # Linux/macOS
   # or
   venv\Scripts\activate       # Windows
   ```
3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

Running the service
- Ensure AWS credentials available via environment variables or instance role:
  - AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION (or use IAM role)
- (Optional) export custom secret names:
  ```bash
  export UPSTOX_CONFIG_SECRET_NAME="my_upstox_config"
  export UPSTOX_TOKEN_SECRET_NAME="my_upstox_access_token"
  ```
- Start the app (development):
  ```bash
  # Run with Uvicorn
  uvicorn main:app --host 0.0.0.0 --port 8001 --reload
  # OR
  python main.py
  ```

API endpoints (prefix: /api/v1)
- GET /api/v1/master-data  
  - Returns broker master/instrument data (depends on broker implementation).

- POST /api/v1/ltp-quote  
  - Body: JSON list of instruments:
  ```json
  [
    {"exchange_token": "21195", "exchange": "NSE", "instrument_type": "EQ"},
    {"exchange_token": "9305", "exchange": "NSE", "instrument_type": "FUTIDX"}
  ]
  ```
  - Returns last traded price data keyed by exchange_token.

- POST /api/v1/ohlc-quote  
  - Body: same as ltp-quote. Returns OHLC data (supports intervals per broker).

- POST /api/v1/full-mkt-quote  
  - Body: same structure. Returns full market quote.

- POST /api/v1/historical-data  
  - Body: JSON object:
  ```json
  {
    "exchange": "NSE",
    "exchange_token": "2885",
    "instrument_type": "EQ",
    "interval": "day",
    "from_date": "2025-01-01",
    "to_date": "2025-01-29"
  }
  ```
  - Returns list of candle dictionaries with keys: datetime, open, high, low, close, volume, oi.

Example curl (LTP):
```bash
curl -X POST "http://localhost:8001/api/v1/ltp-quote" \
  -H "Content-Type: application/json" \
  -d '[{"exchange_token":"21195","exchange":"NSE","instrument_type":"EQ"}]'
```

Token rotation & authentication
- TokenRotationService (services/token_rotation_service.py) initializes brokers and periodically checks token health using lightweight API calls. When a broker token is unhealthy, it calls the broker-specific token rotator to rotate and store a fresh token in Secrets Manager and re-initializes the broker.
- UpstoxTokenRotator uses UpstoxAuthenticator which automates the login flow using Selenium (mobile number, TOTP, PIN) to obtain a new access token.
- Operational note: Selenium-based auth requires a headless browser environment and stable credentials in Secrets Manager.

Adding a new broker
1. Implement a subclass of BaseBroker in `brokers/<broker_name>/broker.py` implementing required methods:
   - _get_broker_name, initialize, fetch_access_token, ltp_quote, historical_data, full_market_quote, etc.
2. Provide broker-specific TokenRotator and Authenticator in the same package if the broker requires token rotation.
3. Register the broker in `brokers/__init__.py` via BrokerFactory.register_broker('<broker_name>', <BrokerClass>).
4. Ensure secrets/configs for that broker exist in Secrets Manager and adapt `api/endpoints.py` get_broker_config if needed.

Logging & monitoring
- Loggers are created by logger.py and send logs to both stdout and AWS CloudWatch via watchtower.
- Ensure IAM permissions for CloudWatch logs and Secrets Manager (see Security / IAM section).

Testing
- Tests (pytest + pytest-asyncio) are referenced in requirements. Run tests with:
  ```bash
  pytest
  ```

Security & operational notes
- Secrets Manager usage: keep secrets safe and limit IAM permissions to required resources.
- Selenium-based auth: storing TOTP keys and automated login credentials is sensitive; rotate credentials and secure access.
- Rate limits: Upstox endpoints are chunked and rate-limited in code—monitor for 429 responses and adjust backoff strategy for production.
- Network & firewall: Ensure the service can reach Upstox endpoints and AWS Secrets Manager.

Minimal IAM permissions (examples)
- Secrets Manager:
  - secretsmanager:GetSecretValue
  - secretsmanager:PutSecretValue
- CloudWatch Logs (for watchtower):
  - logs:CreateLogGroup
  - logs:CreateLogStream
  - logs:DescribeLogStreams
  - logs:PutLogEvents

Contributing
- Contributions welcome. Please open an issue or pull request describing changes or new broker adapters, and include tests for new functionality.

Contact / Support
- For questions or help implementing new brokers, open an issue in this repository.

DISCLAIMER
- This project automates authentication for broker APIs using Selenium for demonstration. Evaluate security and compliance requirements before running automated login flows in production. Make sure secrets and tokens are stored and rotated according to your security policies.
