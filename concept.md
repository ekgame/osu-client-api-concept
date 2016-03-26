# About
This is a proposal for osu! client side API for third-party applications. The API is designed to provide information as well as to allow interaction with the osu! client. 

osu! client here is refered as a Server, third party applications are refered to as Client.

Integration options mockup (by oliebol):

![mockup](http://omk.pics/1/U0Zvx.png)

# TCP packet structure:  
[ULEB128 7-bit encoded integer][UTF-8 String with of length defined by the integer]...

# API life-cycle

1. Client->Server - Try to connect via TCP over port [TBD]
2. Client->Server - Send authentication request (application name, it's purpose, description, etc.), optionally a previously generated token.

    ```js
    {
        "type": "auth",
        "application": "application name",
        "description": "application description",
        "permissions": "read|notify|interact", // or "full"
        "scope": "user", // or client
        "token": "HHWnH0XpdWzej7Ir" // Optional
    }
    ```
    
    ***
    
    osu! should store a hash of the token for later authentication.
    
    If scope `client`:
    `sha512("HHWnH0XpdWzej7Ir") = "0652e041caf727f4f1a6127fe012c3f0"`
    
    If scope: `user`:
    `sha512("HHWnH0XpdWzej7Ir", username) = "0652e041caf727f4f1a6127fe012c3f0"`
    
    ***

3. osu! client should pop up a message to approve the application (for every new user if `"scope": "client"`), unless a valid token is provided.
4. Server->Client - Send authentication response (ok, user denied, banned, etc.) together with a token for later authentication. The token should then be hashed and stored in memory.

    ```js
    {
        "type": "auth",
        "state": "ok",
        "token": "Hz32jEJmgzHgXzVc"
    }
    ```
    
    ***
    
    Possible states:
    * *whitelisted* - authentified and whitelisted.
    * *ok* - authentified because the application was whitelisted.
    * *denied* - user denied connection.
    * *invalid_state* - the user can't approve the connection because theyr in-game.
    * *blacklisted* - the application is blacklisted.
    * *protocol_error* - the client sent a malformed packet.
    
    ***

5. osu! client should pop-out a notification, that the application is now connected.
6. Server<->Client - Various traffic between server and client.
7. Client<->Server - Close connection with a reason.

    ```js
    {
        "type": "disconnect",
        "reason": "shutdown"
    }
    ```
    
    ***
    
    Possible reasons:
    * *shutdown* - the client is being closed by the user.
    * *restart* - the client is performing an update and is restarting.
    * *blacklisted* - the application has been blacklisted by the user.
    
    ***

# Permission levels:
* **read** - can only request and receive information from the client.
* **notify** - can pop up standard (bottom-right) notification to the osu! client.
* **interact** - can force a client to perform an action.
* **full** - all of the above.

# Packets
### Generic error from server
```js
{
    "type": "error",
    "error": "invalid_request"
}
```

***

Error types:
* *not_authentified* - the client has not yet authentified.
* *invalid_request* - the server has received an invalid request from the client
* *permission_error* - the client tried to do something it wasn't permitted to.
* *unsupported_feature* - the client tried to do something that is not yet implemented.

***

### Read permission requests
#### Status requests
*Request*:
```js
{
    "type": "status"
}
```
*Response* (should also be sent when a change happens):
```js
{
    "type": "status",
    "status": "idle", // or "playing", "spectating", "afk", "editing", "modding", "choosing", etc.
    "gamemode": 0,    // 0 - osu!, 1 - taiko, 2 - ctb, 3 - mania
    "mods": 12, // binary encoded mod flags
    "beatmap": {
        "approved": 1,   // 3 = qualified, 2 = approved, 1 = ranked, 0 = pending, -1 = WIP, -2 = graveyard
        "approved_date": "2013-07-02 01:01:12", 
        "last_update": "2013-07-06 16:51:22",
        "artist": "Luxion",
        "beatmap_id": 252002, // or null           
        "beatmapset_id": 93398, // or null            
        "bpm": 196,
        "creator": "RikiH_",
        "difficultyrating": 5.59516,  // Accounting for the currently enabled mods
        "diff_size" : 4,              // Accounting for the currently enabled mods     
        "diff_overall": 6,            // Accounting for the currently enabled mods        
        "diff_approach": 7,           // Accounting for the currently enabled mods         
        "diff_drain": 6,              // Accounting for the currently enabled mods    
        "hit_length": 113,                
        "title": "High-Priestess",   
        "total_length": 145,            
        "version": "Overkill",       
        "file_md5": "c8f08438204abfcdd1a748ebfae67421",            
        "gamemode": 0,               
        "tags": "melodious long",   
        "max_combo": 2101  
    }
}
```
#### Spectator status requests
*Request*:
```js
{
    "type": "spectating_status"
}
```
*Response* (should also be sent when a change happens)::
```js
{
    "type": "spectating_status",
    "is_spectating": true,
    "host_status": "playing", // or "selecting", "paused", "failed", "passed", etc
    "other_spectators": [
        "username_1",
        "username_2",
        ...
    ],
    "beatmap": { // or null if osu! doesn't have the map.
        // see: Status request
    }
}
```

#### Multiplayer status requests
*Request*:
```js
{
    "type": "multiplayer_status"
}
```
*Response* (should also be sent when a change happens):
```js
{
    "type": "multiplayer_status",
    "multiplayer_enabled": true,
    "room": { // may be null if not in room
        "matchtype": "standard", // or "powerplay"
        "scoring_type": "score", // or "accuracy", "combo"
        "team_type": "head_to_head", // or "tag_coop", "team_vs", "tag_team_vs"
        "name": "Room name", 
        "password": "asdfgh", // or empty string for no password 
        "max_slots": 16,
        "mods": 12, // binary encoded mod flags
        "specials": "freemods", // or null
        "users": [
            {
                "slot": 0,
                "username": "username_1",
                "user_id": 123456,
                "status": "idle", // also "ready", "no_map", "playing"
                "mods": 12, // optional, not present when specials is not "freemods"
                "team": "red" // or "blue". Optional, not present when match type is not teambased
            },
            ...
        ]
    }
}
```

### Notify permission requests
#### Notification request
*Request*:
```js
{
    "type": "notification",
    "message": "Notification message."
}
```
*Response*: none expected. May send `error` if the client doesn't have sufficient permissions.
