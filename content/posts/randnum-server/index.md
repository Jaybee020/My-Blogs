---
title: "Creating a Server Interacting with the Lottery Contract"
date: 2023-04-06T16:25:40+01:00
---

Previously,we discussed the lottery contract of [Randnum]("url") in the [previous article]({{< ref "randnum-contract" >}}).We are now ready to describe a server that interacts with the lottery contract, that uses the Algorand Javascript SDK and Indexer to track transactions sent to the smart contract.

## The Express Server

Once the contract has been deployed,the application Id is kept in a .env file.The second piece of the code is the server. The server is written in javascript but can be adapted to any other language.

It is built using express and the Algorand Javascript SDK.The server helps to fetch players previous interactions with the contracts, make view calls to the contract and prepares transactions that need to be signed by the user. These information are exposed via several endpoint.

### Database Model

To make the server more efficient, the lottery game details is stored in a database every new round created by the server.Ideally,this means the server should be responsible for creating every new round of the game.The starting and ending round of each game is stored in the db.A transaction hash is also stored to show that the correct method was called before the result is stored in the database.

```js
import { Schema, model, Document, Model, ObjectId, Types } from "mongoose";
export interface GameParams {
  ticketingStart: number;
  ticketingDuration: number;
  withdrawalStart: number;
  ticketFee: number;
  luckyNumber: number;
  winMultiplier: number;
  maxPlayersAllowed: number;
  maxGuessNumber: number;
  gameMaster: string;
  playersTicketBought: number;
  playersTicketChecked: number;
  totalGamePlayed: number;
}

export interface Lotto extends Document {
  lottoId: number;
  gameParams: GameParams;
  roundStart: number;
  roundEnd: number;
  txReference: string;
}

interface LottoModel extends Model<Lotto> {}

const LottoSchema = new Schema<Lotto>({
  lottoId: {
    required: true,
    type: Number,
  },
  gameParams: { type: Object },
  roundStart: {
    required: true,
    type: Number,
  },
  roundEnd: {
    type: Number,
  },
  txReference: {},
});

export const LottoModel = model<Lotto, LottoModel>("LottoModel", LottoSchema);
```

### Decoder

The decoder is used to decode a player's application calls fetched via the Algorand Indexer.This involves hardcoding each method signature and method name into separate arrays. The decoding process simply involves getting the index of the method signature from the encoded array and returning the data in that particular index of the decoded methods array.

```js
import { ABIMethod } from "algosdk";

export interface LottoGameArgsDecoder {
  decodedMethods: string[];
  encodedMethods: string[];
}

export class LottoGameArgsDecoder {
  constructor() {
    this.encodedMethods = [];
    this.decodedMethods = [];
    const initializeGameABI = new ABIMethod({
      name: "initiliaze_game_params",
      args: [
        {
          type: "uint64",
          name: "ticketing_start",
        },
        {
          type: "uint64",
          name: "ticketing_duration",
        },
        {
          type: "uint64",
          name: "ticket_fee",
        },
        {
          type: "uint64",
          name: "withdrawal_start",
        },
        {
          type: "uint64",
          name: "win_multiplier",
        },
        {
          type: "uint64",
          name: "max_guess_number",
        },
        {
          type: "uint64",
          name: "max_players_allowed",
        },
        {
          type: "account",
          name: "lottery_account",
        },
        {
          type: "pay",
          name: "create_txn",
        },
      ],
      returns: {
        type: "void",
      },
    });
    this.encodedMethods.push(
      Buffer.from(initializeGameABI.getSelector()).toString("base64")
    );
    this.decodedMethods.push(initializeGameABI.name);
    const enterGameABI = new ABIMethod({
      name: "enter_game",
      args: [
        {
          type: "uint64",
          name: "guess_number",
        },
        {
          type: "pay",
          name: "ticket_txn",
        },
      ],
      returns: {
        type: "void",
      },
    });

    this.encodedMethods.push(
      Buffer.from(enterGameABI.getSelector()).toString("base64")
    );
    this.decodedMethods.push(enterGameABI.name);
    const changegNumberABI = new ABIMethod({
      name: "change_guess_number",
      args: [
        {
          type: "uint64",
          name: "new_guess_number",
        },
      ],
      returns: {
        type: "void",
      },
    });

    this.encodedMethods.push(
      Buffer.from(changegNumberABI.getSelector()).toString("base64")
    );
    this.decodedMethods.push(changegNumberABI.name);

    const generateLuckyNumberABI = new ABIMethod({
      name: "generate_lucky_number",
      args: [
        {
          type: "application",
          name: "application_Id",
        },
      ],
      returns: {
        type: "void",
      },
    });

    this.encodedMethods.push(
      Buffer.from(generateLuckyNumberABI.getSelector()).toString("base64")
    );
    this.decodedMethods.push(generateLuckyNumberABI.name);

    const checkUserWinLotteryABI = new ABIMethod({
      name: "check_user_win_lottery",
      args: [
        {
          type: "account",
          name: "player",
        },
      ],
      returns: {
        type: "bool",
      },
    });

    this.encodedMethods.push(
      Buffer.from(checkUserWinLotteryABI.getSelector()).toString("base64")
    );
    this.decodedMethods.push(checkUserWinLotteryABI.name);
  }

  decodeMethod(encodedMethod: string) {
    const index = this.encodedMethods.findIndex(
      (method) => method == encodedMethod
    );
    if (index == -1) {
      return null;
    }
    return this.decodedMethods[index];
  }
}
```

### Utility Functions

The functions below are utility functions used by our server. The functions involve using the Algorand Indexer to fetch interactions between an address and the lottery contract,filtering these transactions by the note value,sending algos,getting the log data from a transaction using its hash among others. They can be studied in the github repo.

### LottoCall Functions

These functions create transactions that interact with the lottery smart contract. It uses the atomic transaction composer for most methods, while methods that require the player's action are returned as transactions to be signed by the player.

This function calculates the minimum amount of algo to be sent to the lottery contract to initialize a new lottery round.

```js
export async function getMinAmountToStartGame(
  ticketFee: number,
  win_multiplier: number,
  max_players_allowed: number | bigint
) {
  const appAccountInfo = await algodClient.accountInformation(appAddr).do();
  const appSpendableBalance =
    appAccountInfo.amount - appAccountInfo["min-balance"];

  return (
    (BigInt(win_multiplier) - BigInt(1)) *
      BigInt(max_players_allowed) *
      BigInt(ticketFee) -
    BigInt(appSpendableBalance)
  );
}
```

The **initializeGameParams** function creates a new game on the lottery contract. The inputs to this method are the necessary game parameters.It constructs an atomic transaction that consists of the payment transaction to fund the contract for a new lottery round and the method call to create a new round. It assigns a group id to the transactions and returns the grouped transactions to the gameMaster to sign and publish to the network.

```js
export async function initializeGameParams(
  gameMasterAddr: string,
  ticketingStart: number | bigint,
  ticketingDuration: number,
  ticketFee: number,
  win_multiplier: number,
  max_guess_number: number | bigint,
  max_players_allowed: number | bigint,
  lotteryContractAddr: string,
  withdrawalStart: number | bigint
) {
  try {
    const params = await algodClient.getTransactionParams().do();
    const minAmountToTransfer = await getMinAmountToStartGame(
      ticketFee,
      win_multiplier,
      max_players_allowed
    );
    const enc = new TextEncoder();
    const newGameTxn = makePaymentTxnWithSuggestedParamsFromObject({
      suggestedParams: params,
      from: gameMasterAddr,
      to: appAddr,
      amount: minAmountToTransfer <= BigInt(1e6) ? 1e6 : minAmountToTransfer,,
      note: enc.encode("init_game"), //for decoding when trying to do profile(exclude init game txns)
    });
    const abi = new ABIMethod({
      name: "initiliaze_game_params",
      args: [
        {
          type: "uint64",
          name: "ticketing_start",
        },
        {
          type: "uint64",
          name: "ticketing_duration",
        },
        {
          type: "uint64",
          name: "ticket_fee",
        },
        {
          type: "uint64",
          name: "withdrawal_start",
        },
        {
          type: "uint64",
          name: "win_multiplier",
        },
        {
          type: "uint64",
          name: "max_guess_number",
        },
        {
          type: "uint64",
          name: "max_players_allowed",
        },
        {
          type: "account",
          name: "lottery_account",
        },
        {
          type: "pay",
          name: "create_txn",
        },
      ],
      returns: {
        type: "void",
      },
    });
    var applCallTxn = makeApplicationNoOpTxn(
      gameMasterAddr,
      params,
      appId,
      [
        abi.getSelector(),
        encodeUint64(ticketingStart),
        encodeUint64(ticketingDuration),
        encodeUint64(ticketFee),
        encodeUint64(withdrawalStart),
        encodeUint64(win_multiplier),
        encodeUint64(max_guess_number),
        encodeUint64(max_players_allowed),
        encodeUint64(1).subarray(7, 8),
      ],
      [lotteryContractAddr]
    );
    return {
      status: true,
      txns: assignGroupID([newGameTxn, applCallTxn]),
    };
  } catch (error) {
    console.log(error);
    return { status: false };
  }
}
```

The **enterCurrentGame** function accepts inputs of the player's address, the player's guess number, and the ticket fee for the current lottery round. This method creates an atomic transaction consisting of the ticket fee payment as a payment transaction and a method call to the lottery contract address. It assigns a group id to these transactions and returns the array of transactions to be signed by the player.

```js
export async function enterCurrentGame(
  playerAddr: string,
  guessNumber: number,
  ticketFee: number | bigint
) {
  // string parameter
  const params = await algodClient.getTransactionParams().do();
  const enc = new TextEncoder();
  const ticketTXn = makePaymentTxnWithSuggestedParamsFromObject({
    suggestedParams: params,
    from: playerAddr,
    to: appAddr,
    amount: ticketFee,
    note: enc.encode("enter_game"), //txn note as specified by the contract
  });
  const abi = new ABIMethod({
    name: "enter_game",
    args: [
      {
        type: "uint64",
        name: "guess_number",
      },
      {
        type: "pay",
        name: "ticket_txn",
      },
    ],
    returns: {
      type: "void",
    },
  });
  const encodedLuckyNumber = encodeUint64(guessNumber);
  var applCallTxn = makeApplicationOptInTxn(playerAddr, params, appId, [
    abi.getSelector(),
    encodedLuckyNumber,
  ]);
  if (await checkUserOptedIn(playerAddr, appId)) {
    applCallTxn = makeApplicationNoOpTxn(playerAddr, params, appId, [
      abi.getSelector(),
      encodedLuckyNumber,
    ]);
  }
  return assignGroupID([ticketTXn, applCallTxn]);
}
```

The **changeGuessNumber** is used to modify the choie of a player's guess number. It creates an application call transaction with the input as the new guess number for the player to sign.

```js
export async function changeCurrentGameNumber(
  playerAddr: string,
  newGuessNumber: number
) {
  const params = await algodClient.getTransactionParams().do();

  const abi = new ABIMethod({
    name: "change_guess_number",
    args: [
      {
        type: "uint64",
        name: "new_guess_number",
      },
    ],
    returns: {
      type: "void",
    },
  });
  const encodedLuckyNumber = encodeUint64(newGuessNumber);
  const applCallTxn = makeApplicationNoOpTxn(playerAddr, params, appId, [
    abi.getSelector(),
    encodedLuckyNumber,
  ]);
  return [applCallTxn];
}
```

The rest of the method calls are made using the atomic transaction composer. This is because they are made by the server and do not neccessarily need to be signed by the user(as the server signs them already).

```js
export async function call(
  user: Account,
  appId: number,
  method: string,
  methodArgs: any[],
  fee?: number,
  OnComplete?: OnApplicationComplete
) {
  const params = await algodClient.getTransactionParams().do();
  params.flatFee = true;
  params.fee = fee == undefined ? ALGORAND_MIN_TX_FEE : fee;

  const commonParams = {
    appID: appId,
    sender: user.addr,
    suggestedParams: params,
    signer: makeBasicAccountTransactionSigner(user),
  };

  let atc = new AtomicTransactionComposer();
  atc.addMethodCall({
    method: getMethodByName(method),
    methodArgs: methodArgs,
    ...commonParams,
    onComplete: OnComplete,
  });
  const result = await atc.execute(algodClient, 1000);
  for (const idx in result.methodResults) {
    // console.log(result.methodResults[idx]);
  }
  return result;
}

export async function getTotalGamesPlayed() {
  try {
    const data = await call(user, appId, "get_total_game_played ", []);
    if (data && data.methodResults[0].returnValue) {
      return parseInt(data.methodResults[0].returnValue.toString());
    }
  } catch (error) {
    return { staus: false };
  }
}

export async function getGameParams() {
  try {
    const data = await call(user, appId, "get_game_params", []);

    if (data && data.methodResults[0].returnValue) {
      return {
        data: data.methodResults[0].returnValue,
        txId: data.txIDs[0],
        status: true,
      };
    }
    return {
      status: false,
    };
  } catch (error) {
    return { status: false };
  }
}

export async function checkUserWinLottery(userAddr: string) {
  try {
    const data = await call(
      user,
      appId,
      "check_user_win_lottery",
      [userAddr],
      2 * ALGORAND_MIN_TX_FEE
    );

    if (data && data.methodResults[0].returnValue) {
      return {
        status: true,
        data: data.methodResults[0].returnValue,
      };
    }
  } catch (error: any) {
    console.log(error.message);
    return { status: false };
  }
}

export async function getUserGuessNumber(userAddr: string) {
  try {
    const data = await call(user, appId, "get_user_guess_number", [userAddr]);
    if (data && data.methodResults[0].returnValue) {
      return {
        data: data.methodResults[0].returnValue.toString(),
      };
    }
  } catch (error) {
    console.log(error);
    return { status: false };
  }
}

export async function generateRandomNumber() {
  try {
    await call(
      user,
      appId,
      "generate_lucky_number",
      [randomnessBeaconContract],
      2 * ALGORAND_MIN_TX_FEE
    );
    return { status: true };
  } catch (error) {
    console.log(error);
    return { status: false };
  }
}

export async function getGeneratedLuckyNumber() {
  try {
    const data = await call(user, appId, "get_lucky_number", []);
    if (data && data.methodResults[0].returnValue) {
      return {
        data: data.methodResults[0].returnValue.toString(),
      };
    }
  } catch (error) {
    return { status: false };
  }
}




export async function resetGameParams(
  lotteryContractAddr: string,
  gameMasterAddr: string,
  protocolAddr: string
) {
  try {
    const data = await call(
      user,
      appId,
      "reset_game_params",
      [lotteryContractAddr, gameMasterAddr, protocolAddr],
      3 * ALGORAND_MIN_TX_FEE
    );
    return {
      status: true,
      confirmedRound: data.confirmedRound,
    };
  } catch (error) {
    console.log(error);
    return { status: false };
  }
}

```

### Helper Functions

These functions are called by our server for its several routes. They call the several methods in the lottery smart contract.

First, we define the needed interfaces to ensure type safety.

1. The **UserBetDetailValue** is a structure used to define a player's interaction with the lottery contract for a particular game round. It contains the player's address, the transaction id for that transaction, the decoded action or method call for that transaction, the value passed as input for the method call, and the confirmation round of the transaction
2. The **UserBetDetail** indexes a player's interactions by lottoId. It is an object whose key is the round of the game played and the value is the userBetDetailValue for that round.
3. The **Transaction** interface is a structure for transactions fetched using the algod indexer API.

```js
interface UserBetDetailValue {
  userInteractions: {
    userAddr: string,
    action: string | null,
    value: any,
    round: number,
    txId: string,
  }[];
  lottoParams?: GameParams;
  id?: string;
}

interface UserBetDetail {
  [key: string]: UserBetDetailValue;
}

interface Transaction {
  sender: string;
  id: string;
  group?: string;
  "confirmed-round": number;
  "application-transaction": {
    "application-args": string[],
    accounts: string[],
  };
  "payment-transaction": {
    receiver: string,
  };
}
```

The **parseLottoTxn** helps to extract necessary information from a list of the user's application call transactions to the lottery contract. This is achieved by decoding the method signature and the method arguments in the application arguments.

```js
function parseLottoTxn(userTxns: Transaction[]) {
  const decoder = new LottoGameArgsDecoder();
  const filteredAndParsed = userTxns
    .filter((userTxn) =>
      decoder.encodedMethods.includes(
        userTxn["application-transaction"]["application-args"][0]
      )
    )
    .map((userTxn) => {
      const action = decoder.decodeMethod(
        userTxn["application-transaction"]["application-args"][0]
      );
      var value;
      if (action == "check_user_win_lottery") {
        value =
          userTxn["application-transaction"]["accounts"][0] ||
          userTxn["sender"];
      } else if (action == "initiliaze_game_params") {
        const initParams = userTxn["application-transaction"][
          "application-args"
        ]
          .slice(1, 8)
          .map((arg) => decodeUint64(Buffer.from(arg, "base64"), "mixed"));
        value = {
          ticketingStart: initParams[0],
          ticketingDuration: initParams[1],
          ticketFee: initParams[2],
          withdrawalStart: initParams[3],
          win_multiplier: initParams[4],
          max_guess_number: initParams[5],
          max_players_allowed: initParams[6],
        };
      } else if (action == "change_guess_number" || action == "enter_game") {
        value = decodeUint64(
          Buffer.from(
            userTxn["application-transaction"]["application-args"][1],
            "base64"
          ),
          "mixed"
        );
      }
      return {
        userAddr: userTxn["sender"],
        action: action,
        value: value,
        txId: userTxn["id"],
        round: userTxn["confirmed-round"],
      };
    });

  return filteredAndParsed;
}
```

The **getUserLottoHistory** and **getUserLottoHistoryByLottoId** is used to get the interactions of the input user address. The getUserLottoHistory fetches this interaction across all games played on the contract. The getUserLottoHistoryByLottoId takes in an extra lottoId parameter that signifies the lottery round we want to fetch the interactions for.This is achieved by querying the database for the starting and ending rounds for that particular transaction round and filtering for interactions between that round

```js
export async function getUserLottoHistory(
  userAddr: string
): Promise<UserBetDetail> {
  try {
    const userTxns: Transaction[] = await getUserTransactionstoApp(
      userAddr,
      appId
    );
    const userInteractions = parseLottoTxn(userTxns).filter(
      (parsedTxns) => parsedTxns.value != undefined
    );

    const data: Record<string, UserBetDetailValue> = {};
    const filtered = await Promise.all(
      userInteractions.map(async (userInteraction) => {
        const lottoDetails = await LottoModel.findOne({
          roundEnd: { $gte: userInteraction.round },
          roundStart: { $lte: userInteraction.round },
        });

        const lottoId = lottoDetails?.lottoId || -1;
        const lottoParams = lottoDetails?.gameParams;
        if (!data[String(lottoId)]) {
          data[String(lottoId)] = {
            userInteractions: [userInteraction],
            lottoParams: lottoParams,
            id: lottoDetails?.id,
          };
        } else {
          data[String(lottoId)]["userInteractions"].push(userInteraction);
        }
      })
    );

    return data;
  } catch (error) {
    console.log(error);
    return {};
  }
}

export async function getUserHistoryByLottoId(
  lottoId: number,
  userAddr: string
): Promise<UserBetDetail> {
  try {
    const betHistoryDetails = await LottoModel.findOne({ lottoId: lottoId });
    var lottoMinRound;
    var lottoMaxRound;
    var userTxns: Transaction[];
    if (betHistoryDetails) {
      lottoMinRound = betHistoryDetails.roundStart;
      lottoMaxRound = betHistoryDetails.roundEnd;
      userTxns = await getUserTransactionstoAppBetweenRounds(
        userAddr,
        appId,
        lottoMinRound,
        lottoMaxRound
      );
    } else {
      return {};
    }
    const userInteractions = parseLottoTxn(userTxns).filter(
      (parsedTxns) => parsedTxns.value != undefined
    );
    return {
      [String(lottoId)]: {
        userInteractions: userInteractions,
        lottoParams: betHistoryDetails?.gameParams,
        id: betHistoryDetails?.id,
      },
    };
  } catch (error) {
    console.log(error);
    return {};
  }
}
```

The **getLottoCallsById** works similar to the **getUserLottoHistoryByLottoId**. The major difference is that it does not take in a user address and instead fetches for all user interactions for that particular lottery round.

```js
export async function getLottoCallsById(lottoId: number) {
  try {
    const betHistoryDetails = await LottoModel.findOne({ lottoId: lottoId });
    if (betHistoryDetails) {
      const lottoMinRound = betHistoryDetails.roundStart;
      const lottoMaxRound = betHistoryDetails.roundEnd;
      const lottoTxns = await getAppCallTransactionsBetweenRounds(
        appId,
        lottoMinRound,
        lottoMaxRound
      );

      const lottoInteractions = parseLottoTxn(lottoTxns).filter(
        (parsedTxns) => parsedTxns.value
      );

      return {
        [String(lottoId)]: {
          userInteractions: lottoInteractions.filter(
            (interaction) =>
              interaction.action == "initiliaze_game_params" ||
              interaction.action == "generate_lucky_number"
          ),
          lottoParams: betHistoryDetails?.gameParams,
        },
      };
    } else {
      return [];
    }
  } catch (error) {
    return [];
  }
}
```

The **getLottoPayTxnById** and **getLottoPayTxn** are used to fetch ticket fee transactions and winner payment transactions for the corresponding game round and all game rounds respectively.

```js
export async function getLottoPayTxnById(lottoId: number) {
  try {
    const betHistoryDetails = await LottoModel.findOne({ lottoId: lottoId });
    if (betHistoryDetails) {
      const lottoMinRound = betHistoryDetails.roundStart;
      const lottoMaxRound = betHistoryDetails.roundEnd;

      const receivedTxns = await getAppEnterGameTransactionsBetweenRounds(
        appAddr,
        lottoMinRound,
        lottoMaxRound
      );
      const sentTxns = await getAppPayWinnerTransactionsBetweenRounds(
        appAddr,
        lottoMinRound,
        lottoMaxRound
      );
      return { receivedTxns: receivedTxns, sentTxns: sentTxns };
    } else {
      return { receivedTxns: [], sentTxns: [] };
    }
  } catch (error) {
    console.log(error);
    return { receivedTxns: [], sentTxns: [] };
  }
}

export async function getLottoPayTxn() {
  try {
    const receivedTxns = await getAppEnterGameTransactions(appAddr);
    const sentTxns = await getAppPayWinnerTransactions(appAddr);
    return { receivedTxns: receivedTxns, sentTxns: sentTxns };
  } catch (error) {
    console.log(error);
    return { receivedTxns: [], sentTxns: [] };
  }
}
```

**generateLuckyNumber** function simply checks if the condition to generate the lucky numbers are met before calling the **generateRandomNumber** method.

```js
export async function generateLuckyNumber() {
  const gameParams = await getCurrentGameParam();

  //only when there is a current game being played and we are in the random number generation stage
  if (
    gameParams.ticketingStart != 0 &&
    gameParams.luckyNumber == 0 &&
    gameParams.ticketingStart + gameParams.ticketingDuration <
      Math.round(Date.now() / 1000)
  ) {
    const result = await generateRandomNumber();
    return result;
  } else {
    console.log("Can not generate random number in this phase of game");
    return { status: false };
  }
}
```

The **getCurrentGameParam** simply fetches the current game parameters and returns it as an object instead of an array.

```js
export async function getCurrentGameParam() {
  const data = await getGameParams();
  const gameParams: GameParams = {
    ticketingStart: 0,
    ticketingDuration: 0,
    withdrawalStart: 0,
    ticketFee: 0,
    luckyNumber: 0,
    playersTicketBought: 0,
    winMultiplier: 0,
    maxGuessNumber: 0,
    maxPlayersAllowed: 0,
    gameMaster: "",
    playersTicketChecked: 0,
    totalGamePlayed: 0,
  };
  const gameParamsKey = [
    "ticketingStart",
    "ticketingDuration",
    "withdrawalStart",
    "ticketFee",
    "luckyNumber",
    "playersTicketBought",
    "winMultiplier",
    "maxGuessNumber",
    "maxPlayersAllowed",
    "gameMaster",
    "playersTicketChecked",
    "totalGamePlayed",
  ];
  gameParamsKey.forEach(
    //@ts-ignore
    (gameParamKey, i) =>
      //@ts-ignore
      (gameParams[gameParamKey] = Number.isNaN(Number(data.data[i]))
        ? //@ts-ignore
          String(data.data[i])
        : //@ts-ignore
          Number(data.data[i]))
  );
  return gameParams;
}
```

The **endCurrentAndCreateNewGame** ends the current game played on the contract and creates a new one. It ensures that we are in the right conditions to end the game played before starting a new game. It also updates the database for every new game to keep track of historical data on the contract. It takes in the necessary game parameters as inputs(or uses the default values specified if they are absent) and returns transactions to be signed by the game Master.

```js
export async function endCurrentAndCreateNewGame(
  ticketingStart = Math.round(Date.now() / 1000 + 200),
  ticketingDuration = 3600,
  withdrawalStart = ticketingStart + 4600,
  ticketFee = 2e6,
  winMultiplier = 2,
  maxPlayersAllowed = 2,
  maxGuessNumber = 100000,
  gameMasterAddr = user.addr
) {
  const data = await getGameParams();
  if (!data?.status || !data.data) {
    return { newLottoDetails: {}, newGame: { status: false, txns: [] } };
  }

  //@ts-ignore
  const gameParams: GameParams = {};
  const gameParamsKey = [
    "ticketingStart",
    "ticketingDuration",
    "withdrawalStart",
    "ticketFee",
    "luckyNumber",
    "playersTicketBought",
    "winMultiplier",
    "maxGuessNumber",
    "maxPlayersAllowed",
    "gameMaster",
    "playersTicketChecked",
    "totalGamePlayed",
  ];
  gameParamsKey.forEach(
    (gameParamKey, i) =>
      //@ts-ignore
      (gameParams[gameParamKey] = Number.isNaN(Number(data.data[i]))
        ? //@ts-ignore
          String(data.data[i])
        : //@ts-ignore
          Number(data.data[i]))
  );

  const lottoId = Number(gameParams.totalGamePlayed);

  // Only reset game when there has been a game played(just initialize game)
  if (gameParams.ticketingStart == 0) {
    console.log("No new game was played on contract");
    const success = await initializeGameParams(
      gameMasterAddr,
      BigInt(ticketingStart),
      ticketingDuration,
      ticketFee,
      winMultiplier,
      maxGuessNumber,
      maxPlayersAllowed,
      appAddr,
      BigInt(withdrawalStart)
    );
    return { newLottoDetails: {}, newGame: success };
  }

  //if game is not yet over do not restart
  if (
    gameParams.withdrawalStart > Math.round(Date.now() / 1000) ||
    gameParams.playersTicketBought != gameParams.playersTicketChecked
  ) {
    console.log("Current Game not finished");
    return { newLottoDetails: {}, newGame: { status: false, txns: [] } };
  }

  const resetStatus = await resetGameParams(
    appAddr,
    gameParams.gameMaster,
    user.addr
  );
  if (!resetStatus.status || !resetStatus.confirmedRound) {
    return { newLottoDetails: {}, newGame: { status: false, txns: [] } };
  }

  const betHistoryDetails = await LottoModel.findOne({ lottoId: lottoId });
  const prevbetHistoryDetails = await LottoModel.findOne({
    lottoId: lottoId - 1,
  });
  if (!betHistoryDetails) {
    await LottoModel.create({
      lottoId: lottoId,
      roundStart: prevbetHistoryDetails?.roundEnd || 0,
      roundEnd: resetStatus.confirmedRound,
      gameParams: gameParams,
      txReference: data.txId,
    });
  } else {
    betHistoryDetails.gameParams = gameParams;
    betHistoryDetails.roundEnd = resetStatus.confirmedRound;
    betHistoryDetails.txReference = data.txId;
    await betHistoryDetails.save();
  }
  var newLotto = await LottoModel.findOne({ lottoId: lottoId + 1 });
  if (!newLotto) {
    newLotto = await LottoModel.create({
      lottoId: lottoId + 1,
      roundStart: resetStatus.confirmedRound,
    });
  }
  const success = await initializeGameParams(
    gameMasterAddr,
    BigInt(ticketingStart),
    ticketingDuration,
    ticketFee,
    winMultiplier,
    maxGuessNumber,
    maxPlayersAllowed,
    appAddr,
    BigInt(withdrawalStart)
  );

  return { newLottoDetails: newLotto, newGame: success };
}
```

The **checkAllPlayersWin** method is used to check the winning status of every player that participated in the current lottery game round. It ensures we are in the withdrawal phase of the lottery game and then fetches every player that interacted with the lottery contract between the starting and ending rounds of the current lottery game round. It then proceeds to check the winning status of unchecked addresses.

```js
export async function checkAllPlayersWin() {
  try {
    const gameParams = await getCurrentGameParam();
    const lottoId = gameParams.totalGamePlayed;
    if (
      gameParams.withdrawalStart != 0 &&
      gameParams.withdrawalStart < Math.round(Date.now() / 1000)
    ) {
      const lotto = await LottoModel.findOne({ lottoId: lottoId });
      const minRound = lotto?.roundStart || 0;
      const playerPayTxns: Transaction[] =
        await getAppEnterGameTransactionsFromRound(appAddr, minRound);

      const potentialPlayers = playerPayTxns.map((txn) => txn.sender);

      const playerCallTxns = await getAppCallTransactionsFromRound(
        appId,
        minRound
      );
      const checkedAddresses = parseLottoTxn(playerCallTxns)
        .filter((parsedTxns) => parsedTxns.action == "check_user_win_lottery")
        .map((parsedTxn) => parsedTxn.value);
      var uncheckedAddresses = potentialPlayers.filter(
        (player) => !checkedAddresses.includes(player)
      );

      uncheckedAddresses = [...new Set(uncheckedAddresses)];
      const chunkSize = 25;
      for (let i = 0; i < uncheckedAddresses.length; i += chunkSize) {
        const chunk = uncheckedAddresses.slice(i, i + chunkSize);
        await Promise.all(chunk.map((player) => checkUserWinLottery(player)));
        console.log(
          `Checked win status for ${i} out of ${uncheckedAddresses.length}`
        );
      }
      return { status: true };
    } else {
      console.log("Not in withdrawal phase.");
      return { status: false };
    }
  } catch (error: any) {
    console.error(error.message);
    return { status: false };
  }
}
```

### Server Routers

The endpoints exposed by the server that responds to requests made is discussed in the repo's server readme.

### Server Workers

There are multiple workers running in the background of the server via cronjobs. They help to run certain functions incase there is not any client to run them. This simply means that the server just simply acts as an open relayer that calls certain methods. These methods include

1. Generating a new game
2. Generating the lucky number
3. Checking the win status of every player that interacted with the contract for that round.

```js
import Queue from "bull";
import { CronJob } from "cron";
import {
  checkAllPlayersWin,
  endCurrentAndCreateNewGame,
  generateLuckyNumber,
  getCurrentGameParam,
} from "../server/helpers";
import {
  MODE,
  REDIS_HOST,
  REDIS_PASSWORD,
  REDIS_PORT,
  initRedis,
  user,
} from "../scripts/config";
import { algodClient, cache } from "../scripts/utils";
import { waitForConfirmation } from "algosdk";
import { GameParams, LottoModel } from "../server/models/lottoHistory";

var newGameQueue: Queue.Queue;
var generateNumberQueue: Queue.Queue;
var checkUserWinQueue: Queue.Queue;
//The least time a game lasts for is 30 mins
if (MODE == "PRODUCTION") {
  newGameQueue = new Queue("newGame", {
    redis: {
      port: REDIS_PORT,
      host: REDIS_HOST,
      password: REDIS_PASSWORD,
    },
  });
  generateNumberQueue = new Queue("generateNumber", {
    redis: {
      port: REDIS_PORT,
      host: REDIS_HOST,
      password: REDIS_PASSWORD,
    },
  });
  checkUserWinQueue = new Queue("checkUserWin", {
    redis: {
      port: REDIS_PORT,
      host: REDIS_HOST,
      password: REDIS_PASSWORD,
    },
  });
} else {
  newGameQueue = new Queue("newGame", "redis://127.0.0.1:6379");
  generateNumberQueue = new Queue("generateNumber", "redis://127.0.0.1:6379");
  checkUserWinQueue = new Queue("checkUserWin", "redis://127.0.0.1:6379");
}

newGameQueue.process(async function (job, done) {
  try {
    const data = await endCurrentAndCreateNewGame();
    console.log(
      `New Game status:${data.newGame.status}. New Game Txn Length:${data.newGame.txns?.length}`
    );
    if (data.newGame.status) {
      const initGameTxns = data.newGame.txns;
      if (initGameTxns && initGameTxns.length > 0) {
        try {
          const signed = initGameTxns.map((txn) => txn.signTxn(user.sk));
          const { txId } = await algodClient.sendRawTransaction(signed).do();
          await waitForConfirmation(algodClient, txId, 1000);
          console.log("Created new Game");
        } catch (error: any) {
          console.log(error.body);
          console.error("Could not create a new game because txn failed");
        }
      }
      const client = await initRedis();
      const key = "Current Game Parameter";
      await cache<GameParams>(key, [], 2, getCurrentGameParam, client);
    }
    done();
  } catch (error: any) {
    console.error(
      "Resetting game failed.Check if current game is still running"
    );
    done(error);
  }
});

generateNumberQueue.process(async function (job, done) {
  try {
    const success = await generateLuckyNumber();
    console.log(`generate number status ${success?.status}`);
    done();
  } catch (error: any) {
    console.error("Error generating number");
    done(error);
  }
});

checkUserWinQueue.process(async function (job, done) {
  try {
    const success = await checkAllPlayersWin();
    console.log(`check user win status ${success.status}`);
    done();
  } catch (error: any) {
    console.error("Error checking user win");
    done(error);
  }
});

export var restartGame = new CronJob(
  "*/60 * * * *",
  function () {
    console.log("Starting to restart game");
    newGameQueue.add(
      {},
      {
        attempts: 3,
        backoff: 3000,
      }
    );
  },
  null,
  true
);

export var generateNumber = new CronJob(
  "*/15 * * * *",
  function () {
    console.log("Starting to generate number");
    generateNumberQueue.add(
      {},
      {
        attempts: 3,
        backoff: 3000,
      }
    );
  },
  null,
  true
);

export var checkUserWin = new CronJob(
  "*/15 * * * *",
  function () {
    console.log("Starting to check users");
    checkUserWinQueue.add(
      {},
      {
        attempts: 3,
        backoff: 3000,
      }
    );
  },
  null,
  true
);

```

Now we have our server and contract. **Goodluck Playing**. You can play a live version of the game [here]("link")
