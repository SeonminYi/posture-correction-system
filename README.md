# Posture Correction System

노트북 카메라(정면)와 측면 스마트폰 카메라를 결합해 사용자의 올바른 자세를 실시간으로 유도하는 시스템입니다.

- **진행 기간**: 2025.07.03 ~ 2025.11.14
- **진행 형태**: 3인 팀 프로젝트
- **본인 담당**: 아이디어 제시, 컴퓨터 비전 모델 통합, 측면 자세 추정 알고리즘 설계, 실시간 서버 및 프론트엔드 연동
- **기술 스택**: Python, OpenCV, MediaPipe BlazePose, SpinePose, WebRTC

## 이 레포의 구성

이 레포는 팀 프로젝트 중 **본인이 직접 설계·구현한 부분만** 정리한 것입니다.

| 폴더 | 내용 |
|---|---|
| [`team-project-reference/`](./team-project-reference) | 팀 프로젝트 전체 구성 및 파트별 담당자 안내, 원본 레포 링크 |
| [`my-contribution/`](./my-contribution) | 본인이 담당한 side_view 파이프라인 코드 및 서버 연동 설계 상세 설명 |

## 핵심 성과

- 2-stage 캐스케이드 구조(BlazePose + SpinePose)로 평균 **45 FPS** 달성 (실시간 처리 기준 30 FPS 상회)
- WebRTC 비동기 이벤트 루프와 AI 추론 연산 간 병목을, 스레드 분리 및 큐 기반 설계로 해결
- 정량적 정확도 검증은 진행하지 않았으나, 여러 각도·인원 대상 자체 테스트를 통해 육안상 자세 이상 유형(거북목, 허리 굽힘)을 안정적으로 구분해내는 것을 확인

자세한 기술 설명은 [`my-contribution/README.md`](./my-contribution/README.md)를 참고하세요.
