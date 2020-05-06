# Generation III Trading Protocol
This is supposed to supplement the code base. You'll only get confused if you use this document as your primary way to learn the Generation III trading protocol.
## Link Entry:
TryTradeLinkup seems to be called when the user trys to trade at the Cable Club. the most important thing this function does is call `sub_80B236C`. All `sub_80B236C` does is register `sub_80B2634` as a task if it not already registered.

`sub_80B2634`, the function that actually starts the link related tasks, calls `OpenLinkedTimed`, which will basically call `OpenLink`. `sub_80B2634` only does this once. The next execution of this task will replace the task's function pointer with a pointer to `sub_80B2688`.

`sub_80B2688` will come into play later. First, let's talk about OpenLink, the function that intializes the link.

### OpenLink
OpenLink is a standard initialization function. It will essentially reset and initialize serial and set the gLinkCallback, a callback that is called in the game loop while the link is active, to `LinkCB_RequestPlayerDataExchange`. Please note that this will **only** happen if gWirelessCommType is 0, indicating that serial is being used versus wireless.

It also creates a new task with the function pointer to `Task_TriggerHandshake`. `Task_TriggerHandshake` is important because it causes `LinkMain1`, when called (we will talk about when this happens), to enable serial and set the state of the gLink to LINK_STATE_HANDSHAKE, which will trigger the communication between the games.

##### requestPlayerDataExchange
This function is the inital callback for gLinkCallback. Only the master will actually use this function. Once it is called, gLinkCallback is set to null. The purpose of this function is to let other players know what type of link is taking place. For example, it could be a trade, battle, or any other link. The code that sets this is as follows:

```c
gSendCmd[0] = LINKCMD_SEND_LINK_TYPE; // this is equal to 0x2222
gSendCmd[1] = gLinkType; // gLinkType is a u16 that defines what type of link is taking place 
```

##### sub_80B2688
This function is the task that runs while the game waits for a connection. If the following check is passed, the new task function pointer will be assigned which differs if the player is the master or slave.

```c
u32 playerCount = GetLinkPlayerCount_2();
if (sub_80B252C(taskId) == TRUE
 || sub_80B2578(taskId) == TRUE
 || playerCount < 2)
    return;
```

The first two checks simply check if B is pressed, and if it is, TRUE is returned. The third check, clearly, checks to see if there are more than two people are linked. Once these checks all pass, the new functions for the master and the slave are executed.

Here, there is a fork in the road. We can either look at the master or slave.


## The Slave
As I mentioned before, `sub_80B2688` can replace the task's function pointer with either a slave function or a master function, depending on the role of the player. If the player happens to be a slave `sub_80B2918` becomes the new function pointer that is called when this task is executed. Let's break down `sub_80B2918`.

### sub_80B2918
Take a second to read over `sub_80B2918`. 

Notice how at the beginning, two variables, local1 and local2, both are assigned data from task's internal storage. The 8 bytes in the task's storage all have a purpose, but in this case, the second and third bytes are being assigned to local1 and local2. The second and third bytes seem to be values that depend on the link type, which I discussed earlier. The second and third bytes are assigned from the arguments of `sub_80B236C`, which I mentioned earlier as the function that registers the new task. In the case of trading, these values are both 2.

Once the check if the player is holding down B or the link is disconnected finishes, a function called `sub_80B2478` is called with our local1 and local2 variables. This function gets the current status of the trade, like if it completed or if you are waiting on your partner to be ready. `sub_80B2478` does this through a function called `GetLinkPlayerDataExchangeStatusTimed`.

#### GetLinkPlayerDataExchangeStatusTimed
The current status of the trade is done with this function. local1 and local2 are passed in as the parameters of this function, which seem to do with trading with more than one person. Take a look at this code, where lower and upper are local1 and local2.

```c
cmpVal = GetLinkPlayerCount_2();
if (lower > cmpVal || cmpVal > upper)
{
    sPlayerDataExchangeStatus = EXCHANGE_STAT_6;
    return 6;
}
```

Notice how local1 and local2 always hold the value 2 in the case of trading, as I mentioned above. This means that when the player count either goes to 3 or drops to 1, the status EXCHANGE_STAT_6 becomes the new status. "lower" is used to check if the player count goes under 2, and "upper" is used to see if it goes above 2. So, EXCHANGE_STAT_6 is simply the leave/join status. The rest of the statuses returned by `GetLinkPlayerDataExchangeStatusTimed` are pretty self explainatory.

#### sub_80B2918's Trainer Card Creation
If `GetLinkPlayerDataExchangeStatusTimed` returns a non-error status, `sub_80B2688` will begin to start sending the trainer card. This is done by casting the send buffer as a trainer card, and filling it with the appropriate data. Once it does this, `sub_80B2C30` becomes the new function pointer of the task.

### sub_80B2C30
