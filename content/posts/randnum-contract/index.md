---
title: "Creating a Decentralized Lottery Game Contract on the Algorand Blockchain"
date: 2023-04-06T16:25:40+01:00
draft: true
description: "RandNum, a guess the number lottery game"
topics: ["Algorand", "Lottery", "Blockchain"]
---

Since the beginning of time, lottery games have been a well-liked means for people to test their luck and possibly win significant sums of money. However, the traditional game necessitates a certain amount of trust in a third party. Players must trust that the lucky number is chosen randomly, rather than after looking at their tickets. Participants must also be confident that the lottery administrators will honor winners’ claims and won’t secretly change players’ numbers to produce fake tickets. To do this, it is necessary to ensure that the lottery’s lucky winning number is chosen randomly, that users’ guesses are acceptable to all players, and that it is free from manipulation.

Blockchain technology has made it possible to design decentralized lotteries that are safe, open, and impartial. The Algorand blockchain solves these problems with its following properties

1.Algorand is a highly secure blockchain platform that employs advanced cryptographic techniques and consensus algorithms to ensure the integrity and confidentiality of transactions. Building your lottery game on Algorand can provide a secure environment for players to participate in the game and trust the fairness of the outcome.

2.Scalability: Algorand is designed to be scalable, capable of processing thousands of transactions per second with low transaction fees. This makes it a suitable platform for building a lottery game with potentially high transaction volume and frequent interactions with the smart contract.

3.Programmability: Algorand supports smart contracts written in pyteal, a powerful and expressive smart contract language that allows you to implement complex logic and rules for your lottery game. This gives you flexibility in designing and implementing the game's logic and rules according to your specific requirements.

4.Transparency: Algorand's blockchain is transparent, meaning that all transactions and smart contract executions are publicly recorded and verifiable. This transparency can foster trust among players, as they can verify the fairness and integrity of the lottery game. It takes

5.Verifiable Random Function (VRF): Algorand incorporates a built-in Verifiable Random Function (VRF) that enables secure and transparent generation of random numbers on-chain. This VRF can be used to fairly and securely determine the winning number in your lottery game, ensuring that the outcome is random and cannot be manipulated by any party, including the contract creator or participants. The use of Algorand's VRF can enhance the trustworthiness of your lottery game and provide players with confidence in the fairness of the game's outcome.

These properties are used to create RandNum, a "guess the number" lottery game built on the Algorand blockchain.
The game’s rules are straightforward: if players accurately predict the winning number, they win a prize. Winners receive their awards in the form of Algo, the native cryptocurrency of the Algorand network.

This article would consists of two parts.The current one describes the smart contract while the [next one]({{< ref "randnum-server" >}}) discusses the backend server interacting with it.The complete code can be found [here](https://github.com/Jaybee020/Randnum)

## Design and Functionality

The picture below gives a general idea of the contract's implementation.
![EditorImages/2023/02/10 14:38/RandNum.jpg](https://algorand-devloper-portal-app.s3.amazonaws.com/static/EditorImages/2023/02/10%2014%3A38/RandNum.jpg)

The immutable contract, written in pyteal, is designed to be as safe and easy to use as possible for users.Its lifecycle is as follows:

1. A user serving as the game master for the current lottery round invokes the **initiliaze_game_params** method of the lottery-smart contract. The contract validates specific requirements and establishes the rules for the current lottery round. In the contract section, we go into greater detail.

2. A player participates in the lottery round by paying the ticket fee and calling the **enter_game** method with their guess number.

3. A client invokes the **generate_lucky_number** method and passes the randomness-beacon-contract application id as input. The random number for the most recent round is then globally stored.

4. Calls are made to read the current game parameters and the guessed number of players by calling the respective methods.

5. The **check_user_win_lottery** method, which takes a player's address as an input argument, is used after the random number generation to determine whether a player is a legitimate winner in the current game round. The contract gives the player their prize if their prediction matches the winning number.

6. After paying every winner, the **reset_game_params** method resets the game parameters to their initial state.

7. The cycle restarts and a new round is created.

This is the overview of the lottery contract's overall structure. Now to dive into the technical part, starting with the PyTEAL smart contract.

## The Lottery Smart Contract

The application contract is an ARC-4 application, meaning it has an abi and a router that manages the various methods. We begin by defining the global variables stored in the contract's storage and setting their initial values.

```py
from pyteal import *
current_ticketing_start = App.globalGet(Bytes("Ticketing_Start"))
current_ticketing_duration = App.globalGet(Bytes("Ticketing_Duration"))
current_withdrawal_start = App.globalGet(Bytes("Withdrawal_Start"))
current_lucky_number = App.globalGet(Bytes("Lucky_Number"))
current_ticket_fee = App.globalGet(Bytes("Ticket_Fee"))
current_win_multiplier = App.globalGet(Bytes("Win_Multiplier"))
current_max_players_allowed = App.globalGet(Bytes("Max_Players_Allowed"))
current_max_guess_number = App.globalGet(Bytes("Max_Guess_Number"))
current_game_master = App.globalGet(Bytes("Game_Master"))
current_game_master_deposit = App.globalGet(Bytes("Game_Master_Deposit"))
current_players_ticket_bought = App.globalGet(Bytes("Players_Ticket_Bought"))
current_players_ticket_checked = App.globalGet(Bytes("Players_Ticket_Checked"))
current_players_won = App.globalGet(Bytes("Players_Won"))
current_total_game_played = App.globalGet(Bytes("Total_Game_Count"))

'''Ticketing_Start-timestamp when tickets can be sold to players
   Ticketing_Duration-amount of time in seconds the ticketing phase should last
   Withdrawal_Start-timestamp when users can check their winning status and receive rewards for winning.
   Lucky_Number-lucky number generated for the lottery
   Ticket_Fee-amount of algos to send to contract to participate in lottery
   Win_Multiplier-reward multiplier for winner
   Max_Players_Allowed-max no of players allowed to participate in lottery round
   Max_Guess_Number-max value for guess number and lucky number for game round
   Players_Ticket_Bought-no of purchased tickets to participate in the current lottery
   Players_Ticket_Checked-no of tickets whose win status have been checked
   Players_Won-keeps track of how many players have won
   Game_Master-initiator of current game round
   Game_Master_Deposit-keeps track of the deposit amount the game master used to intialize the game
   Total_Game_Count-total no of rounds played on this contract.
'''
handle_Creation = Seq(
    App.globalPut(Bytes("Ticketing_Start"), Int(0)),
    App.globalPut(Bytes("Ticketing_Duration"), Int(0)),
    App.globalPut(Bytes("Withdrawal_Start"), Int(0)),
    App.globalPut(Bytes("Lucky_Number"), Int(0)),
    App.globalPut(Bytes("Ticket_Fee"), Int(0)),
    App.globalPut(Bytes("Win_Multiplier"), Int(0)),
    App.globalPut(Bytes("Max_Players_Allowed"), Int(0)),
    App.globalPut(Bytes("Max_Guess_Number"), Int(0)),
    App.globalPut(Bytes("Players_Ticket_Bought"), Int(0)),
    App.globalPut(Bytes("Players_Ticket_Checked"), Int(0)),
    App.globalPut(Bytes("Players_Won"), Int(0)),
    App.globalPut(Bytes("Game_Master"), Global.zero_address()),
    App.globalPut(Bytes("Game_Master_Deposit"), Int(0)),
    Approve()
)

```

The `router` handles the contract's initialization and other bare app calls. The contract is immutable.

```py
router = Router(
    name="Lotto",
    bare_calls=BareCallActions(
        no_op=OnCompleteAction(
            action=Seq(App.globalPut(Bytes("Total_Game_Count"), Int(0)), handle_Creation), call_config=CallConfig.CREATE
        ),
        opt_in=OnCompleteAction(
            action=Approve(), call_config=CallConfig.CALL
        ),
        clear_state=OnCompleteAction(
            action=Reject(), call_config=CallConfig.CALL
        ),
        close_out=OnCompleteAction(
            action=Reject(), call_config=CallConfig.CALL
        ),
        # Prevent updating and deleting of this application
        update_application=OnCompleteAction(
            action=Reject(), call_config=CallConfig.CALL
        ),
        delete_application=OnCompleteAction(
            action=Reject(), call_config=CallConfig.CALL
        ),

    )
)

```

The lottery game's `initialize game params` method initiates a new round. Validations performed by the contract for this method include:

1.  No active game is hosted on the contract; this is to prevent a game from being reset after players have already bought tickets.

2.  If all participants win the lottery, the contract's account balance is sufficient to pay the winners. The game master accomplishes this by sending algos to the application contract via an atomic transfer that raises its balance to make this possible.

3.  Restrictions that ensure there is a suitable time gap between buying tickets, producing the random number, and paying out winners.

```py
@ABIReturnSubroutine
def initiliaze_game_params(ticketing_start: abi.Uint64, ticketing_duration: abi.Uint64, ticket_fee: abi.Uint64, withdrawal_start: abi.Uint64, win_multiplier: abi.Uint64, max_guess_number: abi.Uint64, max_players_allowed: abi.Uint64, lottery_account: abi.Account, create_txn: abi.PaymentTransaction):
    return Seq(
        Assert(
            And(
                # Make sure lottery contract has been reset
                current_ticketing_start == Int(0),
                current_ticketing_duration == Int(0),
                current_withdrawal_start == Int(0),
                current_lucky_number == Int(0),
                current_ticket_fee == Int(0),
                current_max_players_allowed == Int(0),
                current_win_multiplier == Int(0),
                current_max_guess_number == Int(0),
                current_game_master == Global.zero_address(),
                current_game_master_deposit == Int(0),
                current_players_won == Int(0),
                lottery_account.address() == Global.current_application_address(),
                create_txn.get().receiver() == Global.current_application_address(),
                create_txn.get().sender() == Txn.sender(),
                create_txn.get().type_enum() == TxnType.Payment,
                # need to deposit at least 1 algo to create game
                create_txn.get().amount() >= Int(1000000),
                # balance after this transaction is sufficient to pay out a scenario where all participants win the lottery
                create_txn.get().amount()+Balance(lottery_account.address()) -
                MinBalance(lottery_account.address())
                > max_players_allowed.get() *
                (win_multiplier.get()-Int(1))*ticket_fee.get(),
                # Txn Note must be fixed
                create_txn.get().note() == Bytes("init_game"),
                # ticketing phase must start at least 3 minutes into the future
                ticketing_start.get() > Global.latest_timestamp() + Int(180),
                # ticketing phase must be for at least 15 minutes
                ticketing_duration.get() > Int(900),
                # ticketing_fee is greater than 1 algo
                ticket_fee.get() >= Int(1000000),
                # max guess number should be at least 100
                max_guess_number.get() > Int(99),
                max_players_allowed.get() > Int(0),
                # win multiplier is greater than 1,
                win_multiplier.get() > Int(1),
                # withdrawal starts at least 15 mins after ticketing closed
                withdrawal_start.get() > ticketing_start.get() + \
                ticketing_duration.get() + Int(900)
            )

        ),
        App.globalPut(Bytes("Ticketing_Start"), ticketing_start.get()),
        App.globalPut(Bytes("Ticketing_Duration"), ticketing_duration.get()),
        App.globalPut(Bytes("Withdrawal_Start"), withdrawal_start.get()),
        App.globalPut(Bytes("Ticket_Fee"), ticket_fee.get()),
        App.globalPut(Bytes("Win_Multiplier"), win_multiplier.get()),
        App.globalPut(Bytes("Max_Players_Allowed"), max_players_allowed.get()),
        App.globalPut(Bytes("Max_Guess_Number"), max_guess_number.get()),
        App.globalPut(Bytes("Game_Master"), Txn.sender()),
        App.globalPut(Bytes("Game_Master_Deposit"), create_txn.get().amount()),
    )

```

The view method `get_game_params` is used to obtain the current round game parameters for the lottery. We begin by creating a class for our game parameters to define it as an ABItuple.

```py
class Game_Params(abi.NamedTuple):
    ticketing_start: abi.Field[abi.Uint64]
    ticketing_duration: abi.Field[abi.Uint64]
    withdrawal_start: abi.Field[abi.Uint64]
    ticket_fee: abi.Field[abi.Uint64]
    lucky_number: abi.Field[abi.Uint64]
    players_ticket_bought: abi.Field[abi.Uint64]
    win_multiplier: abi.Field[abi.Uint64]
    max_guess_number: abi.Field[abi.Uint64]
    max_players_allowed: abi.Field[abi.Uint64]
    game_master: abi.Field[abi.Address]
    players_ticket_checked: abi.Field[abi.Uint64]
    total_game_played: abi.Field[abi.Uint64]

@ABIReturnSubroutine
def get_game_params(*, output: Game_Params):
    ticketing_start = abi.make(abi.Uint64)
    ticketing_duration = abi.make(abi.Uint64)
    withdrawal_start = abi.make(abi.Uint64)
    ticket_fee = abi.make(abi.Uint64)
    lucky_number = abi.make(abi.Uint64)
    players_ticket_bought = abi.make(abi.Uint64)
    players_ticket_checked = abi.make(abi.Uint64)
    game_master = abi.make(abi.Address)
    win_multiplier = abi.make(abi.Uint64)
    max_guess_number = abi.make(abi.Uint64)
    max_players_allowed = abi.make(abi.Uint64)
    total_game_played = abi.make(abi.Uint64)

    return Seq(
        ticketing_start.set(current_ticketing_start),
        ticketing_duration.set(current_ticketing_duration),
        withdrawal_start.set(current_withdrawal_start),
        ticket_fee.set(current_ticket_fee),
        lucky_number.set(current_lucky_number),
        players_ticket_bought.set(current_players_ticket_bought),
        players_ticket_checked.set(current_players_ticket_checked),
        total_game_played.set(current_total_game_played),
        game_master.set(current_game_master),
        win_multiplier.set(current_win_multiplier),
        max_guess_number.set(current_max_guess_number),
        max_players_allowed.set(current_max_players_allowed),
        output.set(ticketing_start, ticketing_duration,
                   withdrawal_start, ticket_fee, lucky_number, players_ticket_bought,
                   win_multiplier, max_guess_number, max_players_allowed, game_master,
                   players_ticket_checked, total_game_played)
    )
```

The function `enter_game` enables a player to participate in the current lottery game on the smart contract. The contract checks that it is the ticketing period and demands a payment transaction for the ticket fee in addition to your guess number, which is encoded into the application args and saved to the player's local storage. The contract ensures that the ticketing phase is indeed the current phase, that the ticket payment transaction is legitimate, and that the maximum number of allowable tickets is not exceeded.

```py
@ABIReturnSubroutine
def enter_game(guess_number: abi.Uint64, ticket_txn: abi.PaymentTransaction):

    return Seq(
        Assert(

            # Assert that the receiver of the transaction is the smart contract and amount paid is ticket fee and contract is in ticketing phase
            And(
                current_players_ticket_bought < current_max_players_allowed,
                guess_number.get() > Int(0),
                guess_number.get() <= current_max_guess_number,
                ticket_txn.get().receiver() == Global.current_application_address(),
                ticket_txn.get().sender() == Txn.sender(),
                ticket_txn.get().type_enum() == TxnType.Payment,
                ticket_txn.get().note() == Bytes("enter_game"),
                ticket_txn.get().amount() == current_ticket_fee,
                ticket_txn.get().close_remainder_to() == Global.zero_address(),
                Global.latest_timestamp() <= current_ticketing_start+current_ticketing_duration,
            )
        ),

        # If player has checked from previous game,reset value back to 0
        If(App.localGet(Txn.sender(), Bytes("checked"))).Then(
            App.localPut(Txn.sender(), Bytes("checked"), Int(0)),
            App.localPut(Txn.sender(), Bytes("guess_number"), Int(0))
        ),
        # If player buys another ticket do not increase global var
        If(Not(App.localGet(Txn.sender(), Bytes("guess_number")))).Then(
            App.globalPut(Bytes("Players_Ticket_Bought"),
                          current_players_ticket_bought+Int(1))
        ),
        App.localPut(Txn.sender(), Bytes("guess_number"),
                     guess_number.get())
    )
```

`change_guess_number` is used to change a player's guess number. The player making the call must have previously acquired a valid ticket, and the current phase must be the ticketing phase,

```py
@ABIReturnSubroutine
def change_guess_number(new_guess_number: abi.Uint64):
    return Seq(
        Assert(
            # Assert we are still in ticketing phase and user has not been checked to prevent reusing previous ticket numbers
            And(
                new_guess_number.get() > Int(0),
                new_guess_number.get() <= current_max_guess_number,
                Global.latest_timestamp() <= current_ticketing_start+current_ticketing_duration,
                App.localGet(Txn.sender(), Bytes(
                    "guess_number")),
                Not(App.localGet(Txn.sender(), Bytes("checked")))
            )
        ),
        App.localPut(Txn.sender(), Bytes(
            "guess_number"), new_guess_number.get())
    )
```

`get_user_guess_number` is a view method that returns the guess number of the player address passed as input.

```py
@ABIReturnSubroutine
def get_user_guess_number(player: abi.Account, *, output: abi.Uint64):
    return Seq(
        Assert(
            And(
                App.localGet(player.address(), Bytes(
                    "guess_number")),
            )
        ),
        output.set(App.localGet(player.address(), Bytes("guess_number")))
    )
```

The 'generate lucky number' method generates a lucky number for the current lottery round. A reference block number **24908202** is used in this procedure to determine the most recent block to retrieve the random bytes. This method calls the randomness beacon contract to obtain a 32-byte value, extracts the 12th to 20th byte, converts it to a uint, and performs a modulo operation to enable the resultant integer to fall within a valid range. Validations performed by the contract for this method include:

1. Ticketing phase is over

2. The application Id passed as a parameter is the randomness beacon contract

3. No lucky number has been generated before.

4. The application id passed is the

```py
@ABIReturnSubroutine
def generate_lucky_number(application_Id: abi.Application):
    # That block number is the reference point in order to get a valid block round to retrieve randomness from

    most_recent_saved_block_difference = Global.round()-Int(24908202)
    most_recent_saved_block_modulo = most_recent_saved_block_difference % Int(
        8)
    most_recent_saved_block = Int(
        24908202) + most_recent_saved_block_difference-most_recent_saved_block_modulo-Int(16)
    return Seq(
        Assert(
            And(
                # make sure we are calling the right randomness beacon,#change value for mainnet/testnet
                application_Id.application_id() == Int(110096026),
                Global.latest_timestamp() >= current_ticketing_start+current_ticketing_duration,
                current_lucky_number == Int(0)
            )
        ),
        InnerTxnBuilder.Begin(),
        InnerTxnBuilder.SetFields(
            {
                TxnField.type_enum: TxnType.ApplicationCall,
                TxnField.application_id:  application_Id.application_id(),
                TxnField.on_completion: OnComplete.NoOp,
                TxnField.fee: Int(0),
                TxnField.application_args: [MethodSignature(
                    "get(uint64,byte[])byte[]"), Itob(most_recent_saved_block), Txn.sender()]  # adds the sender of the transaction has entropy
            }
        ),
        InnerTxnBuilder.Submit(),
        App.globalPut(Bytes("Lucky_Number"), (Btoi(
            Extract(InnerTxn.last_log(), Int(12), Int(8))) % current_max_guess_number) + Int(1)),
    )
```

The `check_user_win_lottery` method determines whether a player correctly predicted the lucky number. The output is logged if a user has previously checked his win status. If not, the **checked** key in the player's local state is updated and if the player is a genuine winner, the prize is paid to the player.

```py
@ABIReturnSubroutine
def check_user_win_lottery(player: abi.Account, *, output: abi.Bool):
    user_has_checked = App.localGet(player.address(), Bytes("checked"))
    user_guess_correctly = App.localGet(
        player.address(), Bytes("guess_number")) == current_lucky_number
    return Seq(
        Assert(
            And(
                Global.latest_timestamp() >= current_withdrawal_start,
                App.localGet(player.address(), Bytes("guess_number")),
                current_lucky_number
            )
        ),
        If(user_has_checked).Then(
            output.set(user_guess_correctly)
        ).Else(
            If(user_guess_correctly).Then(
                InnerTxnBuilder.Begin(),
                InnerTxnBuilder.SetFields({
                    TxnField.type_enum: TxnType.Payment,
                    TxnField.receiver: player.address(),
                    TxnField.note: Bytes("pay_winner"),
                    TxnField.fee: Int(0),
                    TxnField.amount: current_ticket_fee*current_win_multiplier}),
                InnerTxnBuilder.Submit(),
                App.globalPut(Bytes("Players_Won"), current_players_won+Int(1))
            ),
            App.localPut(player.address(), Bytes("checked"), Int(1)),
            App.globalPut(Bytes("Players_Ticket_Checked"),
                          current_players_ticket_checked+Int(1)),
            output.set(user_guess_correctly)
        )
    )

```

The view methods `get_lucky_number` and `get_total_game_played` simply read the lucky number and total games played from the contract.

```py
@ABIReturnSubroutine
def get_lucky_number(*, output: abi.Uint64):
    return Seq(
        output.set(current_lucky_number)
    )

@ABIReturnSubroutine
def get_total_game_played(*, output: abi.Uint64):
    return Seq(
        output.set(App.globalGet(Bytes("Total_Game_Count")))
    )
```

The `reset_game_params` method signifies the end of a lotto game round. It involves resetting game parameter variables stored in the global state. It can only be called after the win status of all tickets bought has been checked and by the deployer of this contract(The reason is to enable the server discussed in the next part keep track of all lotto game). This method sends 5% of the contract's balance capped at 250 algo to the creator as a protocol fee. It also calculates the amount to refund the game master based on the number of players won and the game master deposit.

```py
@ ABIReturnSubroutine
def reset_game_params(lottery_account: abi.Account, game_master_account: abi.Account, protocol_account: abi.Account):
    lottery_balance = Balance(lottery_account.address()) - \
        MinBalance(lottery_account.address())
    protocol_fee_var = ScratchVar(TealType.uint64)
    game_master_fee_var = ScratchVar(TealType.uint64)
    game_master_reward_var = ScratchVar(TealType.uint64)
    refundable_game_master_deposit = ScratchVar(TealType.uint64)
    return Seq(
        # Make sure every ticket has been checked and winners have been credited
        Assert(
            And(
                is_creator,
                lottery_account.address() == Global.current_application_address(),
                protocol_account.address() == Global.creator_address(),
                game_master_account.address() == current_game_master,
                current_players_ticket_bought == current_players_ticket_checked
            )
        ),
        # handle paying protocol fee to creator and game Master fee
        If(lottery_balance >= Int(1000000)).Then(
            protocol_fee_var.store(lottery_balance/Int(20)),
            # calculate how much players game_master deposit allows
            game_master_reward_var.store(
                current_game_master_deposit/(current_ticket_fee*(current_win_multiplier-Int(1))*current_max_players_allowed)),
            # If it is more than players participating
            If(game_master_reward_var.load() > current_players_ticket_bought).Then(
                game_master_reward_var.store(current_players_ticket_bought)
            ),
            If(protocol_fee_var.load() > Int(250000000)).Then(
                protocol_fee_var.store(Int(250000000)),
            ),
            # every game master receives a minimum of 1000 micro algos
            If((current_players_won*current_win_multiplier*current_ticket_fee) >= current_game_master_deposit).Then(
                refundable_game_master_deposit.store(Int(1000))
            ).Else(
                refundable_game_master_deposit.store(
                    current_game_master_deposit-(current_players_won*current_win_multiplier*current_ticket_fee))
            ),
            # game master usually gets 25% on every ticket bought out of his rewardable players
            game_master_fee_var.store(
                refundable_game_master_deposit.load()+((game_master_reward_var.load()-current_players_won)*current_ticket_fee/Int(4))),
            # Handle the case where the contract can't pay
            If(game_master_fee_var.load() > (lottery_balance-protocol_fee_var.load())).Then(
                game_master_fee_var.store(
                    lottery_balance-protocol_fee_var.load())
            ),
            InnerTxnBuilder.Begin(),
            InnerTxnBuilder.SetFields({
                TxnField.type_enum: TxnType.Payment,
                TxnField.receiver: protocol_account.address(),
                TxnField.note: Bytes("pay_protocol"),
                TxnField.fee: Int(0),
                TxnField.amount: protocol_fee_var.load()}),
            InnerTxnBuilder.Next(),
            InnerTxnBuilder.SetFields({
                TxnField.type_enum: TxnType.Payment,
                TxnField.receiver: game_master_account.address(),
                TxnField.note: Bytes("pay_game_master"),
                TxnField.fee: Int(0),
                TxnField.amount: game_master_fee_var.load()}),
            InnerTxnBuilder.Submit()
        ),
        App.globalPut(Bytes("Total_Game_Count"),
                      App.globalGet(Bytes("Total_Game_Count"))+Int(1)),
        handle_Creation
    )
```

The routing of calls of the contract and generation of approval and clear program is handled as follows

```py

router.add_method_handler(initiliaze_game_params)
router.add_method_handler(get_game_params)
router.add_method_handler(
    enter_game, method_config=MethodConfig(opt_in=CallConfig.CALL, no_op=CallConfig.CALL))
router.add_method_handler(change_guess_number)
router.add_method_handler(get_user_guess_number)
router.add_method_handler(generate_lucky_number)
router.add_method_handler(get_lucky_number)
router.add_method_handler(check_user_win_lottery)
router.add_method_handler(reset_game_params)
router.add_method_handler(get_total_game_played)
approval_program, clear_state_program, contract = router.compile_program(
    version=7, optimize=OptimizeOptions(scratch_slots=True)
)


with open("contracts/app.teal", "w") as f:
    f.write(approval_program)

with open("contracts/clear.teal", "w") as f:
    f.write(clear_state_program)


with open("contracts/contract.json", "w") as f:
    f.write(json.dumps(contract.dictify(), indent=4))

if __name__ == "__main__":
    print(approval_program)

```

### Limitations

One limitation of the contract is that it only allows a fixed number of players to participate. This is to prevent the prize pool from getting too large making it impossible to distribute the prizes fairly.

Another limitation is that the contract allows players to make only one guess per game. This is to prevent players from gaming the system by making multiple guesses and increasing their chances of winning.

### Future Improvements

Future improvements involve allowing the use of ASA to also initialise lottery round. It would also making the contract fully permissionless by making the reset_game_params function open to all instead of just the creator.
