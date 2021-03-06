# Steam for Node.js

This is a Node.js port of [SteamKit2](https://bitbucket.org/VoiDeD/steamre/wiki/Home). It lets you interface with Steam without running an actual Steam client. Could be used to run an autonomous chat/trade bot.

The API is scarce compared to SteamKit2 - however, most chat and trade functions are implemented.


# Installation

```
npm install steam
```

Note: starting from version 0.5.5, node-steam does not support Node.js versions below 0.10.0.

# Usage
First, `require` this module.
```js
var Steam = require('steam');
```
`Steam` is now a namespace (implemented as an object) containing the `SteamClient` class, `servers` property, and a huge collection of enums (implemented as objects). More on those below.

Then you'll want to create an instance of `SteamClient`, call its `logOn` method and assign event listeners.

```js
var bot = new Steam.SteamClient();
bot.logOn('username', 'password');
bot.on('loggedOn', function() { /* ... */});
```

See example.js for the usage of some of the available API.

# Servers

`Steam.servers` contains the list of CM servers node-steam will attempt to connect to. The bootstrapped list (see [servers.js](https://github.com/seishun/node-steam/blob/master/lib/servers.js)) is not always up-to-date and might contain dead servers. To avoid ETIMEDOUT errors, replace it with your own list before logging in if you have one (see ['servers' event](#servers-1)).

# SteamID

Since JavaScript's `Number` type does not have enough precision to store 64-bit integers, SteamIDs are represented as decimal strings. (Just wrap the number in quotes)

# Enums

Whenever a method accepts (or an event provides) an `ESomething`, it's a `Number` that represents some enum value. See [steam_language.js](https://github.com/seishun/node-steam/tree/master/lib/generated/steam_language.js) for the whole list of them.

Note that you can't easily get the string value from the number, but you probably don't need to. You can still use them in conditions (e.g. `if (type == Steam.EChatEntryType.Emote) ...`) or switch statements.

# Properties

## steamID

Your own SteamID.

## users

Information about users you have encountered. It's an object with the following structure:

```js
{
  "steamID of the user": {
    playerName: "the user's current profile name",
    gameName: "the title of the game the user is currently playing"
    // ...and other properties that come directly from Steam
  }
  // ...other users
}
```

## chatRooms

Information about chat rooms you have joined. It's an object with the following structure:
```js
{
  "steamID of the chat": {
    "steamID of one of the chat's current members": {
      rank: "EClanPermission",
      permissions: "a bitset of values from EChatPermission"
    }
    // other members
  }
  // other chats
}
```

For example, `Object.keys(steamClient.chatRooms[chatID])` will return an array of the chat's current members, and `steamClient.chatRooms[chatID][memberID].permissions & Steam.EChatPermission.Kick` will evaluate to a nonzero value if the specified user is allowed to kick from the specified chat.

# Methods

## logOn(username, password, [sentry], [code])

Connects to Steam and logs you on upon connecting. If your account has Steam Guard enabled, you should provide at least one of the below:

* `sentry` - your sentry file hash (see ['sentry' event](#sentry)).
* `code` - the Steam Guard code you'll receive by email. If you have previously logged into another account using node-steam, providing the old hash along with the code will allow you to reuse the same hash for multiple accounts.

If you provide neither, the logon will fail and you'll receive an email with the code.

## webLogOn(callback)

Logs into Steam Community. You only need this if you know you do. `callback` will be called with your new cookie (as a string).

Do not call this before the first [`webSessionID` event](#websessionid), or you'll get a broken cookie. Feel free to call this whenever you need to refresh your web session - for example, if you log into the same account from a browser on another computer.

## gamesPlayed(appIDs)

Tells Steam you are playing game(s). `appIDs` is an array of AppIDs, for example `[570]`. Multiple AppIDs can (used to?) be used for multi-game idling.

## setPersonaName(name)

Changes your Steam profile name.

## setPersonaState(EPersonaState)

You'll want to call this with `EPersonaState.Online` upon logon, otherwise you'll show up as offline.

## sendMessage(steamID, message, [EChatEntryType])

Last parameter defaults to `EChatEntryType.ChatMsg`. Another type you might want to use is `EChatEntryType.Emote`.

## addFriend(steamID)

Sends a friend request.

## removeFriend(steamID)

Removes a friend.

## joinChat(steamID)

Joins the chat room of the specified group. Go to the group's Community page, press Ctrl+U and search for "joinchat". Will silently fail if you are not allowed to join.

## lockChat(steamID)
## unlockChat(steamID)

Locks and unlocks a chat room respectively.

## setModerated(steamID)
## setUnmoderated(steamID)

Enables and disables officers-only chat respectively.

## kick(chatSteamID, memberSteamID)
## ban(chatSteamID, memberSteamID)
## unban(chatSteamID, memberSteamID)

Self-explanatory.

## trade(steamID)

Sends a trade request to the specified user.

## respondToTrade(tradeID, acceptTrade)

Same `tradeID` as the one passed through the [`tradeProposed` event](#tradeproposed). `acceptTrade` should be `true` or `false`.

## cancelTrade(steamID)

Cancels your proposed trade to the specified user.

## toGC(appID, type, body, [callback])

Sends a message to Game Coordinator. `body` must be a serialized message without the header (it will be added by node-steam). `type` must be masked accordingly if it's a protobuf message. If `callback` is provided and this message receives a response (jobID based), the ['fromGC' event](#fromgc) will not be emitted, but instead `callback` will be called with `type` and `body` of the response.

# Events

## 'error'
* `e` - an `Error` object

Something preventing continued operation of node-steam has occurred. `e.cause` is a string containing one of these values:
* 'logonFail' - can't log into Steam. `e.eresult` is an `EResult`, the logon response. Some values you might want to handle are `InvalidPassword`, `AlreadyLoggedInElsewhere` and `AccountLogonDenied` (Steam Guard code required).
* 'loggedOff' - you were logged off for a reason other than Steam going down. `e.eresult` is an `EResult`, most likely `LoggedInElsewhere`.

## 'loggedOn'

You can now safely use all API.

## 'webSessionID'
* your new sessionID

If you are using Steam Community (including trading), you should call [`webLogOn`](#weblogoncallback) again, since your current cookie is no longer valid.

## 'sentry'
* a Buffer containing your Steam Guard sentry file hash

If you didn't provide a hash when logging in, Steam will send you one through this event. If you have Steam Guard enabled, you should save this and use it for your further logons. It will not expire unlike the code.

## 'servers'
* an Array containing the up-to-date server list

node-steam will use this new list when reconnecting, but it will be lost when your application restarts. You might want to save it to a file or a database and assign it to [`Steam.servers`](#servers) before logging in next time.

Note that `Steam.servers` will be automatically updated _after_ this event is emitted. This will be useful if you want to compare the old list with the new one for some reason - otherwise it shouldn't matter.

## 'loggedOff'

You were logged off from Steam due to it going down. ['disconnected'](#disconnected) should follow immediately afterwards. Wait until it reconnects.

## 'disconnected'

Either you were disconnected from Steam, or a connection attempt failed. Don't use any API now - wait until it reconnects.

## 'chatInvite'
* SteamID of the chat you were invited to
* name of the chat
* SteamID of the user who invited you

## 'personaState'
* Object with new user data

Someone has gone offline/online, started a game, changed their nickname or something else. The provided object has the same structure as in the `users` property, and its `friendid` property contains the user's SteamID. Note that the `users` property is not yet updated when this event is fired, so you can compare the new state with the old one to see what changed.

## 'relationship'
* SteamID of the user
* `EFriendRelationship`

Some activity in your friend list. For example, `EFriendRelationship.PendingInvitee` means you got a friend invite, `EFriendRelationship.None` means you got removed.

## 'friendMsg'
* SteamID of the user
* the message
* `EChatEntryType`

## 'chatMsg'
* SteamID of the chat room
* the message
* `EChatEntryType`
* SteamID of the chatter

## 'message'
Same arguments as the above two, captures both events. In case of a friend message, the fourth argument will be undefined.

## 'chatStateChange'
* `EChatMemberStateChange`
* SteamID of the user who entered or left the chat room, disconnected, or was kicked or banned
* SteamID of the chat where it happened
* SteamID of the user who kicked or banned

Something happened in a chat you are in. For example, if the first argument equals `Steam.EChatMemberStateChange.Kicked`, then someone got kicked.

## 'tradeProposed'
* Trade ID
* SteamID of the user who proposed the trade

You were offered a trade.

## 'tradeResult'
* Trade ID
* `EEconTradeResponse`
* SteamID of the user you sent a trade request

Listen for this event if you are the one sending a trade request.

## 'sessionStart'
* SteamID of the other party

The trade is now available at http://steamcommunity.com/trade/{SteamID}. You need a cookie as described in `webLoggedOn`. You can use [node-steam-trade](https://github.com/seishun/node-steam-trade) to automate the trade itself.

## 'announcement'
* SteamID of the group
* headline

Use the group's RSS feed to get the body of the announcement if you want it.

## 'fromGC'
* appID
* `type` - masked accordingly for protobuf
* the message body

A message has been received from GC. See also [`toGC`](#togcappid-type-body-callback).
