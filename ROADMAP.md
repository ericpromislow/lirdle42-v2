Let's build a web-based tic-tac-toe game. Players need to register
with a username and can either log in with a password or via a
3rd-party auth thing. The game has the following parts:

    A lobby that lists everyone logged in.  Some users are "friends"
    of other users, and those names are highlighted. Users can
    specify that they want to see only logged-in friends or everyone,
    but if everyone they only see who's looking for a game.

    A game room. Each game instance takes place in a game room.
    When the game ends and all players and spectators leave, the
    game room can be destroyed

    A game. Obviously -- it's an instance of tic-tac-toe and is
    turn-based, so until the game ends, a player is either about
    to make a move, is making a move, or is waiting for the other
    player to move. Until the game ends, and then the only move the
    player can make is to leave.

    The main room is called the lobby, and there's a lobby-based
    chat where people can find other players to play with. To avoid
    flame wars the users can only use pre-canned sentences, like
    "Anyone want to play?", "<USER> do you want to play?", "where
    is the FAQ?" ,and any other pre-canned sentences you think might
    be useful. The purpose of this is to prevent harsh speech and
    abuse of the chat functionality

    Lobby-based chat needs to be saved for up to a year

    Each game room also has a chat box. All spectators can read it,
    but only the two players can type in it. Chat records are saved
    for up to a year, along with the game details.


The client-side is standard HTML+JavaScript+CSS

Not sure what the server-side should be. The game-play details can
be a Rails app, but I might need something better for the communication
side
…

I'd keep Rails as the main application, including the game logic,
and use WebSockets for the real-time communication. You probably
don't need a separate communications server initially.

The important distinction is between durable application data
(accounts, friendships, games, chat history) and ephemeral state
(who is online, who's currently looking for a game, whose turn it
is).

My recommendation would be a Rails application with PostgreSQL, Redis and Action Cable.
1. Suggested architecture

Browser client

HTML · CSS · JavaScript · WebSocket connection

HTTP + WebSocket

Rails application

Application

Accounts, friends, lobby, game rules, room lifecycle

Action Cable

Lobby updates, game events, chat delivery

PostgreSQL

Persistent accounts, friendships, games, moves, chat messages

Redis

Presence, pub/sub, temporary matchmaking state, connection coordination
Why these technologies?

    Rails: Handles the application rules, database access, authentication, and authorization. It's perfectly capable of managing tic-tac-toe.

    Action Cable: Rails' built-in WebSocket framework. It can broadcast lobby changes, game moves and chat messages to connected browsers.

    PostgreSQL: Stores everything that needs to survive a restart.

    Redis: Handles information that changes frequently and doesn't need to be permanently stored.

I would start with Action Cable. If you eventually need to scale
WebSocket connections independently of Rails, look at AnyCable. It
can replace Action Cable's WebSocket server while retaining much
of the Rails integration.
2. Divide the application into functional areas

I'd organize the application around these areas:

Area
	

Responsibilities

Accounts
	

Registration, login, third-party authentication

Friends
	

Friend requests, acceptance, removal

Presence
	

Online/offline status, availability

Lobby
	

Player listing, filters, invitations

Lobby chat
	

Pre-canned messages, history

Game rooms
	

Players, spectators, room lifecycle

Game engine
	

Board, turns, legal moves, win/draw detection

Game chat
	

Player-only posting, spectator-readable messages

History
	

Games, moves and chat retention

These can all live in one Rails application. They don't need to be separate services.
3. Model the data

Here's a starting point for the relational database.

Table
	

Important fields

users
	

username, password_digest, created_at

authentications
	

user_id, provider, provider_uid

friendships
	

requester_id, recipient_id, status

games
	

status, player_x_id, player_o_id, current_turn, board, winner_id

game_moves
	

game_id, user_id, position, move_number

game_participants
	

game_id, user_id, role

lobby_messages
	

user_id, message_type, target_user_id, created_at

game_messages
	

game_id, user_id, message_type, created_at

A few points:

    authentications allows a user to have multiple login methods, such as a password and Google authentication.

    game_participants records both players and spectators.

    game_moves gives you a complete record of how a game unfolded.

    Store a machine-readable message type rather than arbitrary
    chat text. The client can render the corresponding pre-canned
    sentence.

You could also store the board as nine cells in a compact representation,
or reconstruct it from the moves. I'd probably store the current
board on games for convenient access and retain game_moves as the
historical record.

4. Real-time communication

This is where Action Cable comes in.

I'd use three kinds of channels:

Channel
	

Subscribers
	

Events

LobbyChannel
	

Logged-in users
	

Presence, availability, invitations

LobbyChatChannel
	

Logged-in users
	

New lobby messages

GameChannel
	

Players and spectators in one game
	

Moves, game status, room chat

For example, when a player makes a move:

    Browser sends a move request.

    Rails verifies that the game is active and it's that player's turn.

    Rails validates the selected square.

    Rails updates the board and records the move in PostgreSQL.

    Rails determines whether the game has ended.

    Action Cable broadcasts the updated game state to everyone subscribed to that room.

The server must be authoritative. Never rely on the browser to
determine whether a move is legal or whether someone has won. The
browser displays the state; Rails decides it.

Presence is different from game history

I'd keep online status in Redis rather than PostgreSQL.

A user's presence record might contain:

    User ID

    Last heartbeat

    Current availability (looking, not_looking)

    Current game room, if any

A heartbeat or connection timeout can detect disconnected users. A
user's lobby listing can then be derived from this information and
their friendship relationships.

One subtlety: a user might have several browser tabs open. Presence
should remain online until their last active connection disappears.

5. Lobby behaviour

I would implement your two display modes as follows:

Friends only

    Show online friends.

    Highlight them.

    Show whether they're looking for a game.

    Allow invitations.

Everyone

    Show only online users who are looking for a game.

    Highlight friends among those results.

    Allow invitations.

This distinction is useful: the second mode doesn't expose every logged-in user to strangers.

For the pre-canned chat, I would use message templates with optional parameters:

Message type
	

Displayed sentence

looking_for_game
	

Anyone want to play?

invite_player
	

@username, do you want to play?

ask_faq
	

Where is the FAQ?

ask_rules
	

Can someone explain the rules?

ask_spectators
	

Anyone interested in watching a game?

congratulate
	

Good game!

thank_you
	

Thanks for playing!

goodbye
	

See you later!

The client sends the message type and parameters, not arbitrary
text. The server checks that the message type is permitted and that
any referenced username is valid.

Pre-canned messages reduce the opportunity for abusive language,
but don't eliminate harassment or spam. I'd also add rate limits,
a block feature and a way to report inappropriate usernames.

6. Game room lifecycle

I would make the room lifecycle explicit:

Waiting for players

One or both player slots may be vacant

In progress

Two players; spectators may join

Finished

No more moves; players and spectators can read the result and chat

Archived / eligible for cleanup

All participants have left; persistent game record remains

I'd distinguish between the game room and the game record.

The room is the live environment: subscriptions, current participants,
temporary state. It can be destroyed when everyone leaves.

The game record is persistent: players, moves, result and chat. It
remains in PostgreSQL for the retention period.

You can implement cleanup with a background job that checks for
finished games with no active participants. It can remove ephemeral
room state without deleting the historical record.

7. Chat retention

You specified up to one year for both lobby and game chat.

I'd implement this with timestamps and a scheduled cleanup job:

    Lobby messages older than 365 days are deleted.

    Game chat older than 365 days is deleted.

    Game records and moves are retained for the same period, if that is your intended policy.

    Redis presence data expires automatically.

One design decision worth making: should game details and moves
also disappear after one year, or should only chat messages expire?
You said game details are saved along with chat, but it's worth
specifying whether they share the same retention period.

8. Authentication

I'd use Rails' standard password authentication and add third-party login through OAuth/OIDC.

For example:

    Username and password

    Google sign-in

    Possibly GitHub sign-in

An important design choice: third-party authentication should attach
to an existing user account, not automatically create a second
account every time someone tries another provider. The username
remains the identity used in the game.

9. What I'd actually build first

I'd avoid overengineering the communications layer. This is a
relatively modest real-time application, not a high-frequency
multiplayer game.

My proposed initial stack:

Component
	

Technology

Browser
	

HTML, CSS, JavaScript

Application
	

Rails

Database
	

PostgreSQL

Real-time
	

Rails Action Cable

Pub/sub and presence
	

Redis

Background jobs
	

Solid Queue or Sidekiq

Authentication
	

Rails authentication + OAuth/OIDC

Deployment
	

One Rails application, PostgreSQL and Redis

I'd also make the JavaScript client fairly thin. It should render
the lobby, chat and board, and send user actions. Rails should own
all the business rules.

10. A few decisions before we start coding

There are a handful of details that affect the database and communication design.

    Invitations: Can a user invite someone who isn't currently
    looking for a game, or only users who have advertised availability?

    Spectators: Can anyone watch a public game, or must they be invited by one of the players?

    Disconnects: If a player loses their connection mid-game, should
    the game remain active so they can reconnect?

    Multiple games: Can one user participate in multiple games simultaneously, or only one?

    Retention: Should the game record and moves also be deleted after a year, or just the chat?

My preference would be to allow reconnection, allow spectators to
join public games, and restrict each user to one active game at a
time. Those are reasonable starting assumptions, not requirements.

I'd start by writing the application specification and database
schema, then build the Rails application and Action Cable channels.
There's no compelling reason to introduce a separate communications
server until actual load demonstrates that Action Cable is insufficient.

