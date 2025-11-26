# AutoSchedule - 실시간 협업 스케줄 자동 최적화 시스템

### Repository : https://github.com/sowon2222/autoschedule

### 프로젝트 소개
AutoSchedule은 팀 단위의 작업 / 일정 관리와 자동 스케줄링 기능을 제공하는 프로젝트입니다.  
Spring Boot + WebSocket + FullCalendar.js 기반으로 구현되었습니다.

### Troubleshooting Log
이 프로젝트에서 겪은 에러들과 해결 과정입니다:

- [LazyInitializationException (CalendarEvent 조회)](issues/01_lazy_loading_event_query.md)
- [Websocket Secure Handshake Error](issues/02_websocket_handshake.md)
