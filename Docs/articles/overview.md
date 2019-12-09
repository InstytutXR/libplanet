Libplanet overview
==================

This document describes the structure and usage of [Libplanet] designed to be easily used in [Unity].


[Libplanet]: https://libplanet.io
[Unity]: https://unity.com/


Libplanet
---------

Libplanet is a network/storage library for games that run in distributed P2P network. In many traditional multiplayer games, it was up to the game server to manage the network and the state of games.

```text
                       Traditional multiplayer game

                     +------------------------------+
                     |           Server             |
                     |                              |
                     |                              |
                     |  +------------------------+  |
                     |  |                        |  |
                     |  |       Game State       |  |
                     |  |                        |  |
                     |  +-+----------+---------+-+  |
                     +----+----------+---------+----+
                          |          |         |
                          |          |         |
                     +----+---+ +----+---+ +---+----+
                     |        | |        | |        |
                     | client | | client | | client |
                     |        | |        | |        |
                     +--------+ +--------+ +--------+
```

But with Libplanet, all game clients can manage their own game state, enabling multiplayer games to be implemented without a central server.

```text
                    Multiplayer game with Libplanet


+----------------------+ +----------------------+ +----------------------+
|                      | |                      | |                      |
|        Client        | |        Client        | |        Client        |
|                      | |                      | |                      |
| +------------------+ | | +------------------+ | | +------------------+ |
| |                  | | | |                  | | | |                  | |
| |    Game State    + +-+ +    Game State    + +-+ +    Game State    | |
| |                  | | | |                  | | | |                  | |
| +------------------+ | | +------------------+ | | +------------------+ |
+---------+------------+ +----------------------+ +-----------+----------+
          |                                                   |
          +---------------------------------------------------+

```

For the design rationale of Libplanet, refer to [this article][Libplanet Design].


[Libplanet Design]: design.md


Action and State
----------------

As mentioned above, games using Libplanet can manage their state from each client, not from a central server. The **State** of games is transitioned through an **Action**.

```text
+------------+                  +------------+                  +------------+
|            |                  |            |                  |            |
|  State #1  |                  |  State #2  |                  |  State #3  |
|            |  Battle(Action)  |            |  Battle(Action)  |            |
|            +----------------->+            |----------------->+            |
|    Lv 1    |                  |    Lv 2    |                  |    Lv 3    |
|   Novice   |                  |   Warrior  |                  |   Veteran  |
|            |                  |            |                  |            |
+------------+                  +------------+                  +------------+

```

- Because the way of transitioning the State through an Action depends on each game, Libplanet does not directly provide implementation of the Action and only provides the interface @"Libplanet.Action.IAction".
- The State is expressed in key-value pairs, and you can set appropriate values for each game.
- The State is readable at any point, but transitioning it is only possible through an Action.
- Libplanet does not directly share the transitioned State, but only shares the Action that will transition the State. Also, because State transitioned by an Action occurs on all nodes in the network, the Action must be written deterministically to return the same result from all nodes.


Transaction and Block
---------------------

In order to share an Action, Libplanet uses 2 concepts internally: **Block** and **Transaction**.


```text
+---------------------------+   +----------------------------+   +---------------------------+
|                           |   |                            |   |                           |
| Block #1                  |   | Block #2                   |   | Block #3                  |
|                           |   |                            |   |                           |
| +-----------------------+ |   | +------------------------+ |   | +-----------------------+ |
| |                       | |   | |                        | |   | |                       | |
| |  Transaction #1       | |   | |  Transaction #2        | |   | |  Transaction #4       | |
| |                       | |   | |                        | |   | |                       | |
| |  +-----------------+  | |   | |  +------------------+  | |   | |  +-----------------+  | |
| |  |  Action #1-1    |  | +-->+ |  |  Action #2-1     |  | +-->+ |  |  Action #4-1    |  | |
| |  +-----------------+  | |   | |  +------------------+  | |   | |  +-----------------+  | |
| |                       | |   | |                        | |   | |                       | |
| |  +-----------------+  | |   | +------------------------+ |   | +-----------------------+ |
| |  |  Action #1-2    |  | |   |                            |   |                           |
| |  +-----------------+  | |   | +------------------------+ |   | +-----------------------+ |
| |                       | |   | |                        | |   | |                       | |
| +-----------------------+ |   | |  Transaction #3        | |   | |  Transaction #5       | |
|                           |   | |                        | |   | |                       | |
|                           |   | |  +------------------+  | |   | +-----------------------+ |
|                           |   | |  |  Action #3-1     |  | |   |                           |
|                           |   | |  +------------------+  | |   |                           |
|                           |   | |                        | |   |                           |
|                           |   | +------------------------+ |   |                           |
|                           |   |                            |   |                           |
+---------------------------+   +----------------------------+   +---------------------------+

```

**Transaction** is a bundle of Actions and has the following characteristics:

- Transaction is signed using a private key of the creator(signer) of Transaction and Action.
- Transaction may not include an Action.
- Action in Transaction can be specified in the order of execution and ensures same order execution on all nodes.
- Transaction does not include the game code and cannot be extended. All game logic is implemented through Action.
- It is the game developer's freedom to decide how to group Action units. See [this document][transactions and actions] for more information.

**Block** is a bundle of Transaction and has the following characteristics:

- Block is the smallest unit of mutual Actions in the network. Actions in a Block are either all evaluated, or all not evaluated.
- A Block may include many Transactions or may not include any Transaction.
- A Block may contain multiple users' Transactions.
- A Block does not include game code and cannot be extended by the game developer.
- Block's Transactions are evaluated in the order of execution calculated by deterministic random number to transition the state.


[transactions and actions]: design.md#transactions-and-actions


BlockChain
----------

To manage these Blocks, Libplanet offers a class called @"Libplanet.Blockchain.BlockChain`1". This class allows game developers to:

- Create and register Transactions that contain desired Actions.
- Mine Blocks with registered Transactions.
- Inquire State obtained by evaluating Blocks.


Rendering
---------

One way for game developers to reflect the State of the game is by using [`BlockChain<T>.GetState()`].


```csharp
public class Game : MonoBehaviour
{
    private BlockChain<Action> _blockChain;

    private static void UpdateScore(long score)
    {
        // Update player's score in game.
    }

    // MonoBehaviour.Awake()
    private void Awake()
    {
        UpdateScore((long?) _blockChain.GetState(playerAddress) ?? 0);
    }
}
```

State transition occurs after a Block containing an Action is added and confirmed to the chain. The code above is suboptimal because you are polling the @"Libplanet.Blockchain.BlockChain`1" object to see if a specific Action has been reflected.

```csharp
public class Game : MonoBehaviour
{
    // ...

    private IEnumerator CoScoreUpdater()
    {
        while (true)
        {
            UpdateScore((long?) _blockChain.GetState(playerAddress) ?? 0);
            yield return new WaitForSeconds(1f);
        }
    }

    // MonoBehaviour.Awake()
    private void Awake()
    {
        StartCoroutine(CoScoreUpdater());
    }
}
```

Although this method works, there are still some problems because reflecting the State of the game depends only on time, regardless of whether the action is processed or not.

- If the wait time is long, the interval between States being reflected in the game is also extended, and if it becomes too short, frequent State checks makes the process inefficient.
- If multiple actions were executed in a short period of time, they would not be handled accurately.

Libplanet provides a rendering mechanism called [`IAction.Render()`] to solve this problem. [`IAction.Render()`] is called after a Block with the corresponding Actions is confirmed and the State is transitioned. The following code has been re-implemented using [`IAction.Render()`].

```csharp
public class Win : IAction
{
    // ...

    public override void Render(IActioncontext ctx, IAccountstatedelta nextStates)
    {
        Game.UpdateScore((long?) nextStates.GetState())
    }
}
```

[`BlockChain<T>.GetState()`]: xref:Libplanet.Blockchain.BlockChain`1.GetState(Libplanet.Address,System.Nullable{Libplanet.HashDigest{System.Security.Cryptography.SHA256}},System.Boolean)
[`IAction.Render()`]: xref:Libplanet.Action.IAction.Render(Libplanet.Action.IActionContext,Libplanet.Action.IAccountStateDelta)

Networking
----------

Libplanet has the feature to store Blocks and Transactions, as well as sharing them across other nodes in the P2P network. Game developers can use the @"Libplanet.Net.Swarm`1" class to propagate or receive Blocks from other nodes in the network.

Also, because the network is a P2P network without a central server, @"Libplanet.Net.Swarm`1" also does not need a specific server, however, it is necessary to specify a seed node with a specific address to configure the network. These seed nodes are completely the same as other nodes, except that they must have an IP address or domain accessible from the Internet.
