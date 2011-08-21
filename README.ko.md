# 39DLL 간편스크립트

게임메이커 8용 네트워크 스크립트 라이브러리다. 39DLL의 소켓 함수를 감싸서 서버 개설, 접속, 플레이어 관리, 메시지 송수신을 31개의 `sk_` 스크립트로 정리했다. 소켓 버퍼를 직접 다루지 않고도 멀티플레이 게임을 만들 수 있게 하는 것이 목표였다. 스크립트 리소스 파일과 스크립트 설명, 접속부터 간단한 게임까지 이어지는 예제 5종을 묶어 배포했다.

<p>
  <img src="docs/screenshots/screenshot-1.png" width="306" alt="예제의 대기실 화면">
</p>


## 사용 방법

Releases에서 받은 압축 파일에는 스크립트 리소스(`.gmres`), 39DLL 확장(`.gex`), 스크립트 설명, 예제가 들어 있다. 게임메이커 8에 39DLL 확장을 설치한 뒤 `File > Import Resources` 로 스크립트 리소스를 불러오면 된다.

접속은 세 줄이면 된다. 서버를 연 쪽이 방장이 되고, 나머지는 방장의 IP로 참가한다.

```gml
sk_set_39dll("127.0.0.1", 12345)
if sk_socket_make("방장") { ... }        // 서버 개설
if sk_socket_join("접속자", 1) { ... }   // 참가, 응답을 1초 기다린다
```

메시지는 모아뒀다가 한 번에 보낸다. 메시지 id는 게임에서 정하는 번호이고, 받을 플레이어는 플레이어 id 외에 `0`(방장), `-1`(나를 뺀 모두), `-2`(나를 포함한 모두)를 쓸 수 있다.

```gml
sk_message_save(1, -1)                    // 메시지 id 1, 나를 뺀 모두에게
sk_message_sadd(simple_short, 2, x, y)    // short 두 개
sk_message_sadd(simple_string, 1, name)   // 문자열 하나
sk_message_send()
```

받는 쪽은 Step 이벤트에서 `sk_message_check` 를 돌리면서 메시지 id로 분기한다. 변수는 타입과 순서로 꺼내는데, 순서는 타입별로 0부터 따로 센다.

```gml
while(sk_message_check())
{
  if sk_get_message_id()=1
  {
    x = sk_get_message_var(simple_short, 0)
    y = sk_get_message_var(simple_short, 1)
    name = sk_get_message_var(simple_string, 0)
  }
}
```

인스턴스 단위로 주고받는 공간 메시지도 있다. 인스턴스마다 다른 공간 번호를 두고 `sk_space_message_save(id, 받을 플레이어, 오브젝트, 공간)` 으로 보내면, 받는 쪽 인스턴스의 User Defined 이벤트가 호출된다. 거기서 `sk_space_message_check(공간)` 으로 자기 것인지 확인하고 나머지는 일반 메시지와 같이 읽는다.

스크립트 전체 목록과 인자 설명은 [docs/reference.ko.md](docs/reference.ko.md) 에 있다.


## 구현 원리

**서버가 중계하는 스타 구조다.** 서버를 연 쪽이 플레이어 id 0을 갖고, 접속한 클라이언트는 서버가 accept 할 때 받은 소켓 번호를 그대로 플레이어 id로 받는다. 클라이언트끼리는 직접 연결하지 않고 모든 메시지가 서버를 거친다. 서버는 받은 메시지의 수신자 값을 보고 자기가 처리할지, 특정 클라이언트에게 넘길지, 전부에게 뿌릴지 정한다.

**타입별 버퍼로 직렬화한다.** 패킷은 수신자, 종류, 메시지 id, 보낸 사람 id, 보낸 사람 이름 순의 헤더 뒤에 여섯 타입(byte, short, ushort, int, uint, string)의 개수가 오고, 그다음 타입 순서대로 값이 이어진다. `sk_message_sadd` 는 값을 타입별 임시 버퍼 여섯 개에 쌓아두기만 하고, `sk_message_send` 가 이를 한 패킷으로 합친다. 받는 쪽은 개수만큼 읽어 `[순서, 타입]` 2차원 배열에 넣기 때문에 꺼낼 때도 타입과 순서로 지정한다.

```gml
// sk_message_sadd: 타입별 버퍼에 쌓는다
case 1: dll39_write_short(argument[__i+2], global._39_send_var_buffer_[1]); break;

// sk_message_check: 개수만큼 읽어 배열에 담는다
global._39_message_st_max_[1]=dll39_read_uint(global._39_saved_buffer_)
for(__i=0; __i!=global._39_message_st_max_[1]; __i+=1){global._39_message_st_sts_[__i, 1]=dll39_read_short(global._39_saved_buffer_)}
```

**보내기 전에 목록에 모은다.** `sk_message_save` 는 실제로 보내지 않고 목록에 항목만 추가한다. `sk_message_send` 가 목록을 돌며 패킷을 만들고, 자기 자신이 수신자에 포함되면 네트워크를 거치지 않고 자기 수신 버퍼에 바로 복사한다. 그래서 방장이 방장에게 보내는 메시지도 같은 코드로 처리된다.

**공간 메시지는 이벤트로 배달한다.** `sk_message_check` 가 공간 메시지를 만나면 해당 오브젝트의 모든 인스턴스에 User Defined 이벤트를 띄운다. 각 인스턴스는 `sk_space_message_check` 로 자기 공간인지 확인하고, 맞는 인스턴스가 나오면 나머지 인스턴스는 건너뛴다. 인스턴스가 많을 때 메시지를 효율적으로 나눠 받으려고 만든 방식이다.

**부가 기능.** 비밀번호를 설정하면 39DLL의 버퍼 암호화로 패킷을 감싸서 보낸다. 전송량 제한을 걸면 정해진 개수를 보낼 때마다 잠시 쉬어서 업로드 대역폭이 낮은 서버에서 패킷이 깨지는 것을 막는다. 클라이언트 연결이 끊기면 서버가 그 소켓을 닫고 플레이어 id를 큐에 넣어두므로 `sk_get_sleep_player_id` 로 확인할 수 있다.


## 파일

| 경로 | 내용 |
|---|---|
| `source/39dll-simple-scripts.gmk` | 스크립트를 담은 프로젝트 파일 |
| `source/split/` | GmkSplitter로 분해한 텍스트 트리. 스크립트 31개가 `.gml` 로 들어 있다 |
| `source/39dll_Ext.gex` | 빌드에 필요한 39DLL 확장 |
| `docs/reference.ko.md` | 스크립트 설명 |
| `docs/screenshots/` | 스크린샷 |
| Releases | 스크립트 리소스, 39DLL 확장, 스크립트 설명, 예제 |


## 크레딧

39DLL은 게임메이커 커뮤니티에서 널리 쓰인 파일·네트워크 DLL로 39ster가 만들었다.


## 라이선스

zlib 라이선스다. 자세한 내용은 [LICENSE](LICENSE) 에 있다. 함께 들어 있는 것 중 다른 사람이 만든 라이브러리는 각자의 라이선스를 따른다.
