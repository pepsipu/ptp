# Generation III Trading Protocol
This is supposed to supplement the code base. You'll only get confused if you use this document as your primary way to learn the Generation III trading protocol.
## Link Entry:
TryTradeLinkup seems to be called when the user trys to trade at the Cable Club. the most important thing this function does is call sub_80B236C. All sub_80B236C does is register sub_80B2634 as a task if it not already registered.

sub_80B2634, the function that actually starts the link related tasks, calls OpenLinkedTimed, which will basically call OpenLink. It only does this once. The next execution of this task will replace the task's function pointer with a pointer to sub_80B2688

sub_80B2688 will come into play later. First, let's talk about OpenLink, the function that intializes the link.

### OpenLink:
OpenLink is a standard initialization function. It will essentially reset and initialize serial and set the gLinkCallback (we will talk about this soon) to LinkCB_RequestPlayerDataExchange. Please note that this will **only** happen if gWirelessCommType is 0, indicating that serial is being used versus wireless.

It also creates a new task with the function pointer to Task_TriggerHandshake.
##### requestPlayerDataExchange
This function is the inital callback for gLinkCallback. Only the master will actually use this function. Once it is called, gLinkCallback is set to null. The purpose of this function is to let other players know what type of link is taking place. For example, it could be a trade, battle, or any other link. The code that sets this is as follows:

```c
gSendCmd[0] = LINKCMD_SEND_LINK_TYPE; // this is equal to 0x2222
gSendCmd[1] = gLinkType; // gLinkType is a u16 that defines what type of link is taking place 
```
