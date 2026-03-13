# macOS 앱 사용 기록 파악 가이드

이 문서는 `ps aux` 명령어와 `AppleScript`를 사용하여 macOS에서 현재 사용 중인 앱 정보를 파악하는 방법과 그 효율성에 대해 정리한 가이드입니다.

## 1. 사용 중인 앱 파악 방법

### 방법 A: `ps aux` 활용 (시스템 프로세스 중심)
시스템 전체 프로세스 목록을 보여주며, 특정 경로(`/Applications`)를 필터링하여 실행 중인 앱을 찾을 수 있습니다.
- **장점**: 실행 중인 모든 백그라운드 앱 확인 가능
- **단점**: 현재 화면에 떠 있는 '활성화된 앱'이 무엇인지 바로 알기는 어려움

### 방법 B: `AppleScript` 활용 (활성 창 중심) - **추천**
현재 사용자가 실제로 보고 있는(Frontmost) 앱의 이름과 창 제목을 정확히 가져옵니다.

**실행 명령어:**
```bash
osascript -e 'tell application "System Events" to set frontApp to first process whose frontmost is true' \
          -e 'tell application "System Events" to {displayed name, title} of frontApp'
```

## 2. 결과 해석 예시
*   **출력**: `Visual Studio Code, app_tracking_guide.md`
    *   **앱 이름**: Visual Studio Code
    *   **창 제목**: app_tracking_guide.md (현재 편집 중인 파일명 등)

## 3. 리소스 사용량 분석 (성능)
명령어를 주기적으로(예: 5초마다) 실행할 때 시스템에 미치는 영향은 다음과 같습니다.

| 리소스 | 사용량 및 영향 | 비고 |
| :--- | :--- | :--- |
| **RAM** | 약 10MB ~ 30MB | 실행 시에만 잠시 점유 후 즉시 해제됨 |
| **CPU** | 0.1% 미만 | 시스템 전체 성능에 영향 없음 |
| **안정성** | 매우 높음 | 메모리 누수 없이 안전하게 반복 실행 가능 |

## 4. 실시간 모니터링 스크립트
아래 코드를 터미널에 복사하여 붙여넣으면 5초 간격으로 현재 사용 중인 앱을 로그로 출력합니다.

```bash
while true; do
  echo "--- $(date +%H:%M:%S) ---"
  osascript -e 'tell application "System Events" to set frontApp to first process whose frontmost is true' \
            -e 'tell application "System Events" to {displayed name, title} of frontApp'
  sleep 5
done
```

## 5. 데이터 추출 및 저장 전략 (JSON / YAML)
프로그래밍적으로 데이터를 처리하기 위해 **JSON** 또는 **YAML** 형식으로 저장하는 방법입니다.

### 5.1. JSON 형식으로 저장하기
매번 새로운 로그를 JSON 배열의 객체 형태로 파일에 추가합니다. (간단한 구현을 위해 각 줄에 JSON 객체를 하나씩 쓰는 'JSON Lines' 방식을 추천합니다.)

```bash
LOG_FILE="app_usage_log.json"

while true; do
  # AppleScript 결과 추출
  APP_INFO=$(osascript -e 'tell application "System Events" to set frontApp to first process whose frontmost is true' \
                       -e 'tell application "System Events" to {displayed name, title} of frontApp')
  
  APP_NAME=$(echo "$APP_INFO" | awk -F', ' '{print $1}')
  WINDOW_TITLE=$(echo "$APP_INFO" | awk -F', ' '{$1=""; print $0}' | sed 's/^, //')

  # JSON 객체 생성 및 파일 저장
  printf '{"timestamp": "%s", "app": "%s", "title": "%s"}\n' \
    "$(date '+%Y-%m-%d %H:%M:%S')" "$APP_NAME" "$WINDOW_TITLE" >> "$LOG_FILE"
  
  sleep 5
done
```

### 5.2. YAML 형식으로 저장하기
가독성이 중요한 경우 아래와 같이 리스트 형태로 저장할 수 있습니다.

```bash
LOG_FILE="app_usage_log.yaml"

while true; do
  APP_INFO=$(osascript -e 'tell application "System Events" to set frontApp to first process whose frontmost is true' \
                       -e 'tell application "System Events" to {displayed name, title} of frontApp')
  
  APP_NAME=$(echo "$APP_INFO" | awk -F', ' '{print $1}')
  WINDOW_TITLE=$(echo "$APP_INFO" | awk -F', ' '{$1=""; print $0}' | sed 's/^, //')

  {
    echo "- timestamp: $(date '+%Y-%m-%d %H:%M:%S')"
    echo "  app: \"$APP_NAME\""
    echo "  title: \"$WINDOW_TITLE\""
  } >> "$LOG_FILE"
  
  sleep 5
done
```

## 6. 팀원 배포 시 주의사항 (비서명 앱 대응)

Apple 개발자 유료 플랜을 사용하지 않고 배포할 경우, macOS의 **Gatekeeper**와 **TCC(보안 정책)**에 의해 앱이 차단될 수 있습니다. 이를 해결하기 위한 팀원용 가이드입니다.

### 5.1. 앱 실행 차단 해제 (Gatekeeper)
서명되지 않은 앱을 처음 실행하면 "확인되지 않은 개발자" 경고가 뜹니다.
- **해결법:** 앱 아이콘에서 `우클릭(Control + 클릭)` -> `열기`를 선택하면 '열기' 버튼이 활성화됩니다. 이후에는 정상 실행됩니다.

### 5.2. 스크립트 실행을 위한 권한 허용 (필수)
macOS 보안 정책상, 스크립트가 다른 앱의 이름이나 창 제목을 읽어오려면 반드시 **'손쉬운 사용'** 권한이 필요합니다. 이 설정을 하지 않으면 스크립트 실행 시 오류가 발생하거나 빈 값이 출력됩니다.

- **권한이 필요한 이유:** 스크립트가 현재 활성화된 앱의 상태를 관찰할 수 있도록 허용하기 위함입니다.
- **설정 경로:** `시스템 설정` > `개인정보 보호 및 보안` > `손쉬운 사용(Accessibility)`
- **허용 방법:** 목록에서 스크립트를 실행하는 도구(예: `터미널`, `iTerm2` 또는 제작한 앱)를 찾아 스위치를 **'켬'**으로 변경하세요.

---
*작성일: 2026. 3. 11.*
