# 분리배출 분석 Worker

공유형 분리수거 백과입니다. 공개 화면: https://topksw.github.io/recycle-ai-guide/

index.html은 사진 압축과 화면을 담당하며, worker.js는 Cloudflare의 AI 중계 서버 소스입니다. 기본 공유 분석에는 방문자의 API 키가 필요하지 않습니다.

## 배포 시 필요한 설정
- D1 데이터베이스를 DB 이름으로 바인딩합니다. quotas 테이블은 요청 시 생성됩니다.
- Worker Secrets에 GEMINI_API_KEY 및/또는 OPENROUTER_API_KEY를 설정합니다. 키를 HTML, Git, 공개 변수, 채팅에 넣지 마세요.
- 대기 중에도 만료된 할당량 행을 정리하려면 Cron Trigger를 매시 정각(0 * * * *)으로 설정합니다. scheduled 핸들러가 정리합니다.
- 현재 중계 주소: https://recycle-ai-proxy.lbluedl.workers.dev/analyze. 현재 공유 분석은 전용 OpenRouter Secret으로 가격 0인 이미지 모델을 우선 사용합니다. Google AI의 일부 Cloudflare 지역 차단을 피하기 위해 OpenRouter Secret이 있으면 Gemini 직접 호출을 생략합니다. Gemini Secret은 OpenRouter Secret이 없는 구성에서 사용합니다. 개인 키 모드는 설정에서 선택적으로 사용할 수 있습니다.

## 계약
모든 경로에 Origin: https://topksw.github.io가 필요합니다. 없거나 다르면 403입니다. OPTIONS는 지정된 메서드와 Content-Type 헤더만 허용합니다.

GET /health: {"ready":true|false}. 키 하나 이상과 DB 바인딩 유무만 확인하며 실제 DB 연결, 키 유효성, 모델 가용성은 검사하지 않습니다. 준비되지 않으면 503이며 세부 설정은 반환하지 않습니다.

POST /analyze: Content-Type: application/json, 본문은 {"image":"data:image/jpeg;base64,..."} 하나의 필드만 허용합니다. jpeg/png/webp만 허용하고 선언 MIME과 매직 바이트를 대조합니다. HTTP 본문은 스트림 누적 최대 2,200,000바이트, 디코딩 이미지 최대 1,500,000바이트입니다. 완전한 이미지 디코딩/악성 파일 검사는 아니며 실제 판독 실패는 안전한 오류로 처리합니다.

성공: {"result":{"itemName":"...","isRecyclable":true,"category":"...","badgeColor":"green","steps":["...","...","..."],"caution":"..."},"model":"..."}. 모델 출력은 키·자료형·길이·단계 수·색상·기본 일관성을 엄격히 검사합니다. HTML 및 제어 문자는 거부하지만 프런트엔드는 반드시 textContent로 표시해야 합니다. 모델 결과도 신뢰할 수 없는 참고 정보입니다.

오류: {"error":"한국어 안내"}. 400 형식, 403 출처, 404 경로, 405 메서드, 413 크기, 415 콘텐츠 유형, 429 한도, 502 모델 실패, 503 준비/DB 실패, 504 시간 초과. 429에는 Retry-After 초 단위 헤더가 있습니다. 제공사 오류·응답·비밀은 사용자에게 전달하거나 코드에서 기록하지 않습니다.

## 모델과 시간 제한
OpenRouter Secret이 없을 때 Gemini 순서는 gemini-3.1-flash-lite, gemini-3-flash-preview, gemini-3.5-flash입니다. 가용성은 보장하지 않습니다. Gemini 요금제/청구 제한은 제공사 계정에서 별도 관리해야 합니다. Gemini 무료 사용을 강제하는 설정은 아닙니다.

OpenRouter 키가 있을 때만 카탈로그를 조회해 이미지 입력 및 텍스트 출력이 가능하고 prompt/completion 가격이 명시적으로 0인 모델을 최대 두 개 선택합니다. 실제 요청에도 최대 prompt/completion 가격 0 및 제공사 fallback 비활성화를 설정합니다. 유료 모델이나 고정 미검증 모델로 전환하지 않습니다. 가격 변경/부가 과금/계정 정책의 최종 관리는 제공사에서 확인하세요.

요청당 추론 호출 최대 4회, OpenRouter 카탈로그 조회 최대 1회입니다. 따라서 현재 OpenRouter Secret이 있으면 Gemini 직접 호출을 생략하고 무료 이미지 모델을 최대 2회 시도합니다. 업로드·한도 확인·카탈로그를 포함한 전체 응답 제한 85초, 각 외부 HTTP 호출은 본문 읽기를 포함하여 최대 25초입니다. OpenRouter의 출력 한도는 2,200토큰이며 낮은 추론 강도를 요청합니다. 남은 시간이 적으면 다음 호출 시간이 줄어듭니다.

## 한도와 개인정보
UTC 날짜 기준 전역 100회/일, IP 해시별 10회/일 및 10초에 1회입니다. Cloudflare가 설정하는 CF-Connecting-IP가 없으면 거부합니다. Worker 앞단이 해당 헤더를 신뢰할 수 있게 전달하는 표준 Cloudflare 배포를 전제로 합니다.

D1 serial batch 트랜잭션에서 행을 먼저 parameterized INSERT OR IGNORE하고 조건부 UPDATE ... RETURNING으로 증가시킵니다. IP 증가 바로 다음 global UPDATE의 SQLite changes() 조건으로 두 카운터를 연결합니다. 전역 한도에는 쿨다운이 없습니다. DB 장애/불일치는 보수적으로 차단합니다. 수락 후 모델이 모두 실패하거나 시간 초과되어도 한도는 환불하지 않습니다.

D1에는 날짜+SHA-256(IP+날짜) 식별자, 카운터, 마지막 시각만 저장하며 원본 IP·사진·분석 결과를 저장하지 않습니다. 원본 IP는 해시 계산 중 메모리에서만 사용합니다. 해시는 익명화 보장이 아니며 IP 공간을 추측하면 연결될 수 있습니다. 서버는 사진을 선택된 AI 제공사로 전송하므로 제공사의 데이터 보관·학습·무료 모델 정책 및 Cloudflare 플랫폼 로그 정책을 별도로 검토해야 합니다.

요청 시 및 Cron 실행 시 오늘/어제(UTC)보다 오래된 날짜 prefix 행을 삭제합니다. 매시 정각 Cron이 정상 실행되면 약 2일 이내 보관을 목표로 하며, Cron 없이 트래픽이 끊기거나 정리 실행이 실패하면 물리적 삭제가 다음 성공까지 지연됩니다. 규제상 엄격한 삭제 보장은 별도 운영 모니터링이 필요합니다.

## 한계
Origin/CORS는 브라우저 접근 제한이지 인증 또는 강력한 봇 방지가 아닙니다. 비브라우저 클라이언트는 Origin을 위조할 수 있고, 공유 IP는 한도를 공유하며 IP 회전으로 우회할 수 있습니다. topksw.github.io의 다른 프로젝트도 같은 Origin입니다. 공개 서비스에는 필요에 따라 Turnstile·인증·WAF 및 비용 알림을 추가하세요.

고정 서버 프롬프트는 사진 속 지시를 따르지 않도록 안내하고 다른 프롬프트·모델·URL을 클라이언트가 지정하지 못하게 합니다. 그렇더라도 AI 오답·프롬프트 인젝션 완전 방지는 보장하지 않습니다. 큰 단단한 과일 씨는 일반쓰레기로 안내하며, 전구·지역별 기준은 관할 지자체 문의를 요구합니다. 공식 지침을 실시간 조회하거나 정확성을 보증하는 서비스가 아닙니다.
