# 스크립트 설명

39DLL 위에서 동작하는 게임메이커 8용 네트워크 스크립트다. 서버를 연 사람이 방장이 되고, 나머지는 방장의 IP 로 참가한다. 모든 메시지는 방장을 거쳐 전달된다.

## 연결

- `sk_set_39dll(ip, port)` : 접속할 IP 와 포트를 정한다. 다른 스크립트보다 먼저 호출해야 한다. `ip` 는 `"127.0.0.1"` 같은 문자열이고, `port` 는 65535 까지의 정수다.
- `sk_socket_make(name)` : 새 서버를 연다. `name` 은 자신의 플레이어 이름이다. 방장의 플레이어 id 는 0 이다. 같은 포트에 이미 서버가 있으면 `false` 를 돌려준다.
- `sk_socket_join(name, seconds)` : `sk_set_39dll` 로 정한 서버에 참가한다. `name` 은 자신의 플레이어 이름이고, `seconds` 는 서버의 응답을 기다리는 시간(초)이다. 서버가 닫혀 있거나 시간 안에 응답하지 않으면 `false` 를 돌려준다.
- `sk_socket_close(seconds)` : 서버와의 연결을 끊는다. `seconds` 는 퇴장 메시지가 전송되기를 기다리는 시간(초)이다. 다시 참가하려면 `sk_set_39dll` 부터 다시 호출해야 한다.
- `sk_socket_player_close(player)` : 지정한 플레이어의 연결을 끊는다. 서버에서만 사용할 수 있다.
- `sk_get_server_check()` : 서버와 연결되어 있는지 돌려준다.

## 메시지 보내기

메시지는 모아 두었다가 `sk_message_send` 로 한 번에 보낸다. 받을 플레이어는 플레이어 id 외에 `0`(방장), `-1`(나를 뺀 모두), `-2`(나를 포함한 모두)를 쓸 수 있다.

- `sk_message_save(id, player)` : 메시지 id 인 메시지를 `player` 에게 보내도록 모아 둔다. `id` 는 게임에서 정하는 번호다. 여러 번 호출할 수 있다.
- `sk_message_sadd(type, count, value1, value2, ...)` : 마지막으로 모아 둔 메시지에 변수를 붙인다. `type` 은 아래 패킷 형식이고 `count` 는 뒤에 오는 변수의 개수다. 여러 번 호출할 수 있다.
- `sk_message_send()` : 모아 둔 메시지를 전부 보낸다.
- `sk_get_save_message_max()` : 모아 둔 메시지의 개수를 돌려준다.

| 패킷 형식 | 값 | 크기 |
|---|---|---|
| `simple_byte` | 정수 0 부터 255 | 1바이트 |
| `simple_short` | 정수 -32768 부터 32767 | 2바이트 |
| `simple_ushort` | 정수 0 부터 65535 | 2바이트 |
| `simple_int` | 정수 -2147483648 부터 2147483647 | 4바이트 |
| `simple_uint` | 정수 0 부터 4294967295 | 4바이트 |
| `simple_string` | 문자열 | 한 글자에 1바이트 |

## 메시지 받기

- `sk_message_check()` : 메시지가 왔는지 확인한다. 새 플레이어를 받아들이고 공간 메시지를 해당 오브젝트에 전달하는 일도 여기서 한다. 메시지가 없으면 `false` 를 돌려준다. `while` 로 감싸면 한 스텝에 여러 메시지를 받는다.
- `sk_get_message_id()` : 받은 메시지의 id 를 돌려준다.
- `sk_get_message_send()` : 받은 메시지를 보낸 플레이어의 id 를 돌려준다.
- `sk_get_message_send_name()` : 받은 메시지를 보낸 플레이어의 이름을 돌려준다.
- `sk_get_message_var(type, index)` : 받은 메시지에 붙은 변수를 돌려준다. `index` 는 같은 형식끼리 0 부터 센다.
- `sk_get_message_var_max(type)` : 받은 메시지에 붙은 변수 중 `type` 형식의 개수를 돌려준다.

## 공간 메시지

공간 메시지는 특정 인스턴스에게 보내는 메시지다. 인스턴스마다 서로 다른 공간 번호를 두고, 받는 인스턴스의 User Defined 이벤트에서 `sk_space_message_check` 로 자기 것인지 확인한다.

- `sk_space_message_save(id, player, object, space)` : `object` 의 인스턴스 중 공간 번호가 `space` 인 인스턴스에게 보낼 메시지를 모아 둔다. 변수는 `sk_message_sadd` 로 붙인다.
- `sk_set_space_event(number)` : 공간 메시지가 도착했을 때 실행할 User Defined 이벤트 번호를 정한다. 모든 인스턴스에 같이 적용된다. 기본값은 0 이다.
- `sk_space_message_check(space)` : User Defined 이벤트 안에서 호출한다. 도착한 공간 메시지가 `space` 에게 온 것이면 `true` 를 돌려준다. 변수는 일반 메시지와 같은 방법으로 읽는다. `while` 로 감싸지 않는다.

## 플레이어

- `sk_set_my_player_name(name)` : 자신의 플레이어 이름을 바꾼다.
- `sk_get_my_player_id()` : 자신의 플레이어 id 를 돌려준다.
- `sk_get_my_player_name()` : 자신의 플레이어 이름을 돌려준다.
- `sk_get_my_ip(kind)` : 자신의 IP 를 돌려준다. `kind` 가 0 이면 외부 IP, 1 이면 내부 IP 다. 외부 IP 는 whatismyip.org 에 물어보는 방식이라 지금은 동작하지 않을 수 있다.
- `sk_get_server_ip()` : 서버의 IP 를 돌려준다.
- `sk_set_player_accept(accept)` : 새 플레이어를 받아들일지 정한다. `false` 면 접속 자체를 막는다. 다시 열 때 성공하면 `true` 를 돌려준다. 서버에서만 사용할 수 있다.
- `sk_set_player_accept_max(max)` : 서버에 접속할 수 있는 플레이어 수의 상한을 정한다. 0 이면 제한하지 않으며 기본값이다. 서버에서만 사용할 수 있다.
- `sk_get_player_max_number()` : 지금까지 서버에 들어온 플레이어 수를 돌려준다. 서버에서만 사용할 수 있다.
- `sk_get_player_number()` : 현재 서버에 있는 플레이어 수를 돌려준다. 서버에서만 사용할 수 있다.
- `sk_get_sleep_player_id()` : 연결이 끊긴 플레이어의 id 를 하나씩 돌려준다. 없으면 0 을 돌려주므로 0 이 아닌 동안 반복해서 호출한다. `sk_message_check` 뒤에 호출해야 한다. 서버에서만 사용할 수 있다.

## 기타

- `sk_set_39dll_packet_password(password)` : 메시지를 암호화해서 보낸다. 서버와 클라이언트가 같은 비밀번호를 써야 한다. 기본값은 `""` 이다.
- `sk_set_message_send_max_packet(count, seconds)` : 메시지를 `count` 개 보낼 때마다 `seconds` 초 쉰다. 업로드 속도가 낮은 쪽이 서버일 때 메시지가 깨지는 것을 막는다. 기본값은 0, 0 이다.

## 주의

플레이어 id 는 방장이 0 이고, 참가한 순서대로 2 이상의 값을 받는다. 나간 플레이어의 id 는 다음에 들어온 플레이어가 이어받는다.

플레이어와 변수는 배열로 다루므로 플레이어는 최대 32000 명, 변수는 메시지 하나에 형식마다 32000 개까지다.

## 변경 내역

v4.0.7

- 메시지를 모두에게 뿌리지 않고 지정한 플레이어에게만 보낸다.
- `sk_set_39dll_packet_password` 로 패킷을 암호화할 수 있다.
- `sk_set_message_send_max_packet` 으로 과다 전송을 막는다.
- 받은 변수의 인덱스는 형식마다 0 부터 시작한다.
