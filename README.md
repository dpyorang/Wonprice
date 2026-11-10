# WonPrice

중고 상품을 경매나 즉시구매로 거래하고, 거래가 성사되면 판매자와 구매자가 채팅할 수 있는 팀 프로젝트입니다.

- **개발 기간** · 2023.08.24–2023.09.19
- **팀 구성** · 프론트엔드 3명, 백엔드 3명
- **담당 역할** · 백엔드 팀장. 정기 코드 리뷰와 팀 통합 테스트를 진행하고, 경매 종료·즉시구매를 채팅과 연결하는 기능과 메시지·읽음 처리를 개발했습니다.
- **백엔드 기술** · Java 11, Spring Boot 2.7, Spring Data JPA, MySQL, Spring Security, WebSocket·STOMP

## 담당한 백엔드 기능

### 경매 종료와 즉시구매를 채팅으로 연결

경매가 끝나거나 즉시구매가 성사되면, 판매자와 구매자가 거래 방법을 협의할 수 있도록 채팅방을 생성했습니다.

- **시간 만료** · Spring Scheduler가 작업 종료 후 5초 간격으로 종료 시각이 지난 경매를 조회합니다. 구매자가 있으면 채팅방을 만들고 상품을 거래 중 상태로 변경합니다. 구매자가 없으면 채팅방 없이 경매를 종료합니다.
- **즉시구매** · 상품 가격을 즉시구매가로 확정하고 구매자와 거래 상태를 갱신한 뒤, 판매자와 구매자를 채팅방에 등록합니다.
- **입찰가가 즉시구매가에 도달한 경우** · 입찰 요청을 처리하는 과정에서 경매를 종료하고 채팅방을 생성하도록 연결했습니다.

관련 코드 · [경매 종료 스케줄러](server/wonprice/src/main/java/main/wonprice/scheduler/ProductScheduler.java), [상품 서비스](server/wonprice/src/main/java/main/wonprice/domain/product/service/ProductServiceImpl.java), [채팅 서비스](server/wonprice/src/main/java/main/wonprice/domain/chat/service/ChatService.java)  
기여 기록 · [경매 종료 후 채팅방 자동 생성](https://github.com/dpyorang/Wonprice/commit/2e6122bc481031fe049bb3a0edde30e976852d8f), [즉시구매 구현](https://github.com/dpyorang/Wonprice/commit/39d1abde7aa0ca3bca615fd7fe196191fb094c3a), [즉시구매가 도달 시 경매 종료](https://github.com/dpyorang/Wonprice/commit/90f1f74506d3ca38e0e7f20abe2c03a4a14c15b2)

### 실시간 메시지와 읽음 상태

채팅방마다 STOMP 목적지를 나눴습니다. 클라이언트가 `/app/chat/{room-id}`로 메시지를 보내면 서버는 메시지를 저장하고, 같은 방의 `/topic/chat/{room-id}` 구독자에게 결과를 전달합니다.

읽음 상태는 **참여자별 마지막으로 읽은 메시지 ID**로 관리합니다. 방에 들어오면 마지막 메시지 ID를 기록하고, 새 메시지가 들어오면 현재 접속 중인 참여자의 읽음 위치를 갱신합니다. 이 위치보다 뒤에 저장된 메시지로 읽지 않은 개수를 계산합니다. WebSocket 접속과 종료 이벤트에 맞춰 채팅 세션을 등록하고 삭제하며, 현재 코드에서는 해당 세션을 JPA로 관리합니다.

관련 코드 · [메시지 컨트롤러](server/wonprice/src/main/java/main/wonprice/domain/chat/controller/MessageController.java), [채팅 참여자](server/wonprice/src/main/java/main/wonprice/domain/chat/entity/ChatParticipant.java), [WebSocket 연결 이벤트](server/wonprice/src/main/java/main/wonprice/websocket/WebSocketConnectListener.java)  
기여 기록 · [접속 상태를 반영한 읽음 처리](https://github.com/dpyorang/Wonprice/commit/4ed40f1ea2f3286099d0900aa04b3b7b0827104e)

### 입찰과 채팅의 메시지 경로 분리

입찰 결과는 상품별 `/topic/bid/{productId}`로, 채팅 메시지는 방별 `/topic/chat/{room-id}`로 전달하도록 목적지를 구분했습니다. 입찰 상세 화면은 가격이 바뀌면 다음 입찰 판단에 바로 반영해야 하지만, 목록은 조회가 중심이라 필요한 갱신 수준이 달랐습니다. 현재 저장소에서 확인한 참여 경매 목록은 HTTP API를 30초마다 다시 조회합니다.

관련 코드 · [입찰 컨트롤러](server/wonprice/src/main/java/main/wonprice/domain/product/controller/BidController.java), [참여 경매 목록](client/src/components/auctionContent/AuctionContent.tsx)  
기여 기록 · [경매·채팅 WebSocket 목적지 분리](https://github.com/dpyorang/Wonprice/commit/3bc84e5e92edaff80245b47fd7ee247f340a7529)

### 상품·프로필 이미지 연결

클라이언트가 이미지를 S3에 업로드하면, 백엔드는 전달받은 URL을 상품이나 회원 프로필에 연결해 저장하고 조회 응답에 포함합니다. 이 중 이미지 정보의 저장·조회와 프로필 이미지 변경 처리를 담당했습니다.

관련 코드 · [상품 컨트롤러](server/wonprice/src/main/java/main/wonprice/domain/product/controller/ProductController.java), [회원 컨트롤러](server/wonprice/src/main/java/main/wonprice/domain/member/controller/MemberController.java)  
기여 기록 · [이미지 등록 및 조회 구현](https://github.com/dpyorang/Wonprice/commit/ef0e8b3ed42e1ffda983e153343b00385dd90a31)

## 저장소 안내

백엔드 소스는 [`server/wonprice`](server/wonprice), 프론트엔드 소스는 [`client`](client)에 있습니다. 팀 구성과 전체 기능은 [당시 팀 README](https://github.com/KongDaePyo/Wonprice)를 참고할 수 있습니다. 위 구현 설명은 현재 `main` 브랜치의 코드와 커밋 이력을 기준으로 정리했습니다.
