# 본인 담당 파트 — Side View 자세 분석 + 서버 연동

> 이 폴더는 [팀 프로젝트](../team-project-reference)에서 본인이 직접 설계·구현한 부분만 정리한 것입니다. 팀 프로젝트 전체 구성은 위 링크를 참고하세요.

## 담당 역할

- 아이디어 제시
- 컴퓨터 비전 모델 통합 (BlazePose + SpinePose)
- 측면 자세 추정 알고리즘 설계 (side_view 전담)
- 실시간 서버 및 프론트엔드 연동 (WebRTC 서버 일부 담당)

---

## 1. side_view — 측면 자세 추정 파이프라인

### 핵심 알고리즘: 2-Stage Cascade 구조

BlazePose로 상체를 먼저 크롭한 뒤, SpinePose(경량 모델)에 입력하는 방식으로 설계했습니다.

- 초기에는 SpinePose 단독으로 진행했으나, 경량 모델을 사용할 경우 정확도가 떨어지고, 고용량 모델을 사용할 경우 실시간성이 확보되지 않는 문제가 있었습니다.
- BlazePose로 상체를 크롭(정사각형 마진 적용)한 후 SpinePose에 입력하는 구조로 전환해, FPS 향상과 키포인트 추론 정확도 개선을 동시에 달성했습니다.
- 평균 **45 FPS** 달성 (실시간 처리 기준 30 FPS를 상회)

### 스레드 분리를 통한 병목 해결

AI 추론(무거운 연산)과 WebRTC 영상 수신(비동기 이벤트 루프)이 한 스레드에서 충돌하며 병목이 발생하는 문제를, 역할별로 스레드를 분리해 해결했습니다.

```
[WebRTC 콜백] process_frame_callback (비동기)
    │  프레임을 frame_q에 넣기만 함 (AI 연산 없음, non-blocking)
    │  result_q에서 이미 처리된 최신 결과를 꺼내 즉시 WebRTC로 반환 (핸드폰 화면에 표시)
    │  → result_q가 비어있으면, 직전에 처리됐던 마지막 결과를 대신 반환 (화면 끊김 방지)
    ▼
  frame_q (maxsize=1)
    ▼
[AI 추론 워커 스레드] inference_worker
    │  frame_q에서 프레임을 꺼내 무거운 AI 파이프라인(BlazePose+SpinePose) 실행
    │  처리 결과를 result_q와 display_q 양쪽에 동시에 넣음
    ├──────────────┐
    ▼              ▼
  result_q      display_q (maxsize=1)
  (WebRTC로       │
   반환용)         ▼
              [디스플레이 워커 스레드] display_worker
                로컬(노트북) 디버깅 화면에 결과 렌더링
```

**설계 의도**

- WebRTC 콜백은 "프레임을 큐에 던지고, 처리된 결과를 꺼내오기만" 하도록 만들어 AI 연산이 아무리 오래 걸려도 WebRTC 이벤트 루프가 막히지 않게 했습니다.
- AI 추론은 별도 스레드(`inference_worker`)에서 독립적으로 실행되어, 자신의 속도대로 계속 돌아갑니다.
- 결과는 `result_q`(WebRTC 응답용)와 `display_q`(로컬 디버깅 화면용) 두 개로 분리해, 최종 사용자에게 보여지는 경로와 개발자 디버깅 경로를 독립적으로 운영했습니다.
- 모든 큐는 `maxsize=1`로 설정하고, 큐가 꽉 찼을 때 새 프레임이 기존 것을 덮어쓰도록(`replace_if_full=True`) 구성했습니다. 프레임 수신 속도가 AI 연산 속도보다 빠른 상황에서, 오래된 프레임이 큐에 쌓여 지연이 누적되는 대신 **항상 가장 최신 프레임을 기준으로 처리**되도록 만든 설계입니다.

---

## 2. server-integration — WebRTC 서버 연동

`server.py`는 B팀원과 공동으로 작성한 파일이라 전체 코드는 포함하지 않았습니다. 아래는 본인이 설계·기여한 부분에 대한 설명이며, 이해를 돕기 위한 짧은 발췌만 포함했습니다.

### 담당 범위

`server.py`는 팀원과 함께 작성한 파일로, 그중에서도 AI 파이프라인과 연결되는 부분을 중점적으로 맡았습니다.

| 구성 요소 | 담당 |
|---|---|
| `offer()` (WebRTC SDP 핸드셰이크) | 팀원과 공동 작업 |
| `on_track()` / `reader()` (영상 트랙 수신) | 팀원과 공동 작업 |
| `frame_callback` 연동 구조 설계 | AI 파이프라인과의 연결 지점으로, 중점적으로 담당 |

### WebRTC 연동 흐름

핸드폰 브라우저가 카메라 영상을 획득해 WebRTC로 서버(노트북)에 전송하면, 서버는 이를 받아 AI 파이프라인에 연결합니다.

1. 핸드폰 브라우저: `getUserMedia()`로 카메라 스트림 획득 → SDP Offer 생성
2. 서버: Offer 수신 → SDP Answer 생성 → WebRTC 연결 확립
3. 핸드폰: 영상 트랙을 서버로 실시간 전송
4. 서버: `on_track()`으로 트랙 수신 → `frame_callback`을 통해 AI 추론 파이프라인과 연결
5. 처리된 결과를 WebRTC를 통해 다시 핸드폰 화면으로 실시간 반환

### 핵심 설계: `frame_callback` 인터페이스

서버(WebRTC 통신 담당)와 AI 파이프라인(본인 담당)이 서로의 내부 구현에 의존하지 않도록, 콜백 함수 형태의 인터페이스로 연결되게 설계했습니다.

```python
# server.py (발췌) — B팀원과 공동 작업한 서버 쪽 인터페이스
def set_frame_callback(callback):
    global frame_callback
    frame_callback = callback
```

```python
# side_view/run.py — 본인이 작성한 연동 부분
async def frame_callback(img):
    return await process_frame_callback(ctx, img)
server.set_frame_callback(frame_callback)
```

이렇게 분리해두면, 서버 쪽은 "콜백 함수를 등록받아 실행한다"는 것만 알면 되고, AI 파이프라인 쪽은 WebRTC 통신 세부사항을 몰라도 됩니다. 두 모듈을 독립적으로 개발·교체할 수 있게 만든 것이 이 설계의 핵심입니다.

---

## 파일 구조

```
my-contribution/
└── side_view/
    ├── __init__.py
    └── run.py              # 자세 추정 파이프라인 + 워커 스레드 구조 + 서버 연동 지점
```
