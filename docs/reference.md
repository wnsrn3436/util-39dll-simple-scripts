# Script reference

Networking scripts for GameMaker 8 that run on top of 39DLL. Whoever opens the server is the host, and everyone else joins with the host's IP. Every message goes through the host.

## Connection

- `sk_set_39dll(ip, port)` : Sets the IP and port to connect to. Must be called before any other script. `ip` is a string such as `"127.0.0.1"`, and `port` is an integer up to 65535.
- `sk_socket_make(name)` : Opens a new server. `name` is your player name. The host's player id is 0. Returns `false` if a server is already listening on that port.
- `sk_socket_join(name, seconds)` : Joins the server set with `sk_set_39dll`. `name` is your player name, and `seconds` is how long to wait for the server to answer. Returns `false` if the server is closed or does not answer in time.
- `sk_socket_close(seconds)` : Closes the connection to the server. `seconds` is how long to wait for the leave message to be sent. To join again, start over from `sk_set_39dll`.
- `sk_socket_player_close(player)` : Closes the connection of the given player. Can only be used on the server.
- `sk_get_server_check()` : Returns whether the server is connected.

## Sending messages

Messages are collected first and sent all at once with `sk_message_send`. Besides a player id, the receiver can be `0` (the host), `-1` (everyone but me), or `-2` (everyone including me).

- `sk_message_save(id, player)` : Queues a message with message id `id` for `player`. `id` is a number the game defines. Can be called any number of times.
- `sk_message_sadd(type, count, value1, value2, ...)` : Attaches variables to the most recently queued message. `type` is one of the packet types below and `count` is the number of values that follow. Can be called any number of times.
- `sk_message_send()` : Sends every queued message.
- `sk_get_save_message_max()` : Returns the number of queued messages.

| Packet type | Value | Size |
|---|---|---|
| `simple_byte` | integer 0 to 255 | 1 byte |
| `simple_short` | integer -32768 to 32767 | 2 bytes |
| `simple_ushort` | integer 0 to 65535 | 2 bytes |
| `simple_int` | integer -2147483648 to 2147483647 | 4 bytes |
| `simple_uint` | integer 0 to 4294967295 | 4 bytes |
| `simple_string` | string | 1 byte per character |

## Receiving messages

- `sk_message_check()` : Checks whether a message has arrived. It also accepts new players and delivers space messages to their objects. Returns `false` if there is no message. Wrap it in a `while` loop to receive several messages in one step.
- `sk_get_message_id()` : Returns the id of the received message.
- `sk_get_message_send()` : Returns the player id of whoever sent the received message.
- `sk_get_message_send_name()` : Returns the player name of whoever sent the received message.
- `sk_get_message_var(type, index)` : Returns a variable attached to the received message. `index` counts from 0 within each type.
- `sk_get_message_var_max(type)` : Returns how many variables of `type` are attached to the received message.

## Space messages

A space message is a message addressed to a particular instance. Each instance gets its own space number, and the receiving instance checks in its User Defined event with `sk_space_message_check` whether the message is for it.

- `sk_space_message_save(id, player, object, space)` : Queues a message for the instance of `object` whose space number is `space`. Attach variables with `sk_message_sadd`.
- `sk_set_space_event(number)` : Sets which User Defined event runs when a space message arrives. It applies to every instance. The default is 0.
- `sk_space_message_check(space)` : Call it inside the User Defined event. Returns `true` if the space message that arrived is for `space`. Variables are read the same way as normal messages. Do not wrap it in a `while` loop.

## Players

- `sk_set_my_player_name(name)` : Changes your player name.
- `sk_get_my_player_id()` : Returns your player id.
- `sk_get_my_player_name()` : Returns your player name.
- `sk_get_my_ip(kind)` : Returns your IP. `kind` 0 is the external IP and 1 is the local IP. The external IP is looked up at whatismyip.org, so it may no longer work.
- `sk_get_server_ip()` : Returns the IP of the server.
- `sk_set_player_accept(accept)` : Sets whether new players are accepted. `false` blocks connections entirely. Returns `true` if reopening succeeded. Can only be used on the server.
- `sk_set_player_accept_max(max)` : Sets the maximum number of players the server accepts. 0 means no limit and is the default. Can only be used on the server.
- `sk_get_player_max_number()` : Returns how many players have joined the server so far. Can only be used on the server.
- `sk_get_player_number()` : Returns how many players are on the server now. Can only be used on the server.
- `sk_get_sleep_player_id()` : Returns the ids of players whose connection dropped, one per call. Returns 0 when there are none, so call it repeatedly while the result is not 0. Must be called after `sk_message_check`. Can only be used on the server.

## Other

- `sk_set_39dll_packet_password(password)` : Encrypts messages before sending. The server and the clients must use the same password. The default is `""`.
- `sk_set_message_send_max_packet(count, seconds)` : Pauses for `seconds` after every `count` messages sent. It keeps messages from breaking when the server has a slow upload. The default is 0, 0.

## Notes

The host's player id is 0, and joining players get ids from 2 upward in join order. The id of a player who left is reused by the next player who joins.

Players and variables are held in arrays, so there can be at most 32000 players and 32000 variables per type in one message.

## Changes

v4.0.7

- Messages go only to the specified player instead of being broadcast to everyone.
- Packets can be encrypted with `sk_set_39dll_packet_password`.
- `sk_set_message_send_max_packet` prevents sending too many messages.
- Indexes of received variables start at 0 within each type.
