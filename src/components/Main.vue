<template>
<div class="right">
    <select :disabled="walletConnected" v-model="selectedNetwork">
        <option>mainnet-beta</option>
        <option>testnet</option>
        <option>devnet</option>
        <option>localhost</option>
    </select>

    <span v-if="walletConnected">
        <button @click="wallet.disconnect()">Disconnect</button>
    </span>
    <span v-else>
        <button @click="connectWallet()">Connect</button>
    </span>

    <div v-if="walletConnected">
        <br/>
        <span> {{ solBalance == null ? "--" : solBalance }} SOL </span>
        <br/>
    </div>
</div>

<div>
    <p>
        play chess with extend
    </p>

    <div class="center" v-if="!chessboard">
        <button :disabled="!walletConnected" @click="setChessboard()">lets do it</button>
    </div>
    <div id="board"></div>
</div>
</template>

<script>
import Chess from "chess.js";
import ChessBoard from "chessboardjs-vue";
import * as w3 from "@solana/web3.js";
import { PhantomWalletAdapter } from "@solana/wallet-adapter-phantom";
import { shallowRef } from 'vue';

const NETWORKS = {
    "mainnet-beta": "https://solana-api.projectserum.com",
    testnet: "https://api.testnet.solana.com",
    devnet: "https://api.devnet.solana.com",
    localhost: "http://127.0.0.1:8899",
};

export const BASE = "XBSEKuwLduP8A95EDxFYxmQ9whW6eXqtJGC6ayk2Uee";
export const CHESSBOARD = "4cuXq7BFBJh1eb1vJv3K9Y4iFLznaQ67UzL6UgYDs1HV";
export const CHESS_PROGRAM_ID = new PublicKey("CH5d8PWucjbVfpGQaEYn4AFN6o7qFLWfB8p4LwNMhCY3");
export const WALLET_W = "[16,247,81,151,78,116,109,56,154,200,27,69,82,194,125,120,182,136,162,167,81,99,15,89,35,175,81,47,165,96,104,242,249,161,238,52,175,61,240,220,100,207,207,244,172,42,105,196,171,242,237,22,41,113,177,43,39,102,78,86,208,187,66,173]";
export const WALLET_B = "[47,133,83,206,214,50,96,2,23,173,225,3,255,91,35,5,125,15,207,155,45,129,132,37,164,172,54,169,255,168,91,105,5,55,101,139,77,18,229,212,15,54,64,149,214,27,28,234,156,139,199,199,4,150,127,228,203,15,103,76,237,248,200,115]";
export const ENV = "devnet";

export default {
    name: "Main",
    components: {},
    mounted() {},
    data() {
        return {
            connection: null,
            wallet: shallowRef(null),
            walletConnected: false,
            solBalance: null,
            selectedNetwork: "devnet",
            chessboardAddress: new w3.PublicKey(CHESSBOARD),
            chessboard: null,
            game: new Chess(),
            ply: 0,
        }
    },
    watch: {
        selectedNetwork(curr, prev) {
            if(curr != prev) {
                this.connectChain();
            }
        }
    },
    methods: {
        async connectWallet() {
            let viewmodel = this;
            viewmodel.wallet = new PhantomWalletAdapter();

            viewmodel.wallet.on("connect", () => {
                viewmodel.walletConnected = true;
                viewmodel.fetchBalance();
            });

            viewmodel.wallet.on("disconnect", () => {
                viewmodel.wallet = null;
                viewmodel.walletConnected = false;
                viewmodel.solBalance = null;
            });

            await viewmodel.wallet.connect();
        },
        connectChain() {
            this.connection = new w3.Connection(NETWORKS[this.selectedNetwork]);
        },
        async fetchBalance() {
            if (!this.connection) this.connectChain();
            this.solBalance = await
                this.connection.getBalance(this.wallet.publicKey)
                .then(bal => (bal / 10 ** 9).toFixed(3))
                .catch(() => null);
        },
        async setChessboard() {
            let chessAccountInfo = await this.connection.getAccountInfo(
                this.chessboardAddress,
                'confirmed'
            );
            var start = 53; // 32 + 8 * 2 + 1 + 4
            let chessboardBytes = chessAccountInfo.data.slice(start,start+73);
            let turn = {0: 'w', 1: 'b'}[chessboardBytes[72]];
            this.ply = (chessboardBytes[71]-1) * 2;  // legal_chess starts at 1
            if (turn == 'b') this.ply += 1;
            let pos = ChessBoard.objToFen(this.getPosition(chessboardBytes)) + ' ' + turn + ' KQkq - 0 1';
            console.log('Loading pos', pos);
            this.game.load(pos);
            console.log(this.game.fen());
            var config = {
              draggable: true,
              position: pos,
              onDragStart: this.onDragStart,
              onDrop: this.onDrop,
              onSnapEnd: this.onSnapEnd
            }
            this.chessboard = ChessBoard('board', config);
        },
        onDragStart (source, piece, position, orientation) {
          if (this.game.game_over()) return false
          if ((this.game.turn() === 'w' && piece.search(/^b/) !== -1) ||
              (this.game.turn() === 'b' && piece.search(/^w/) !== -1)) {
            return false
          }
        },
        onDrop (source, target) {
          var move = this.game.move({
            from: source,
            to: target,
            promotion: 'q' // NOTE: always promote to a queen for example simplicity
          })
          if (move === null) return 'snapback';

          this.ply += 1;
          let cmap = {
            'a':0,'b':1,'c':2,'d':3,'e':4,'f':5,'g':6,'h':7,
            '1':0,'2':1,'3':2,'4':3,'5':4,'6':5,'7':6,'8':7,
          };
          let nSource = cmap[source[1]]*8 + cmap[source[0]];
          let nTarget = cmap[target[1]]*8 + cmap[target[0]];
          console.log("from", source, "to", target);
          voteMove(nSource, nTarget, this.ply-1, {'b':'w','w':'b'}[this.game.turn()]);
        },
        onSnapEnd () {
          this.chessboard.position(this.game.fen())
        },
        getPosition(bytes) {
            console.log(bytes);
            let pos = {};
            let files = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h'];
            let pieces = {
                0: null,
                1: 'wP',
                11: 'bP',
                2: 'wR',
                12: 'bR',
                3: 'wN',
                13: 'bN',
                4: 'wB',
                14: 'bB',
                5: 'wQ',
                15: 'bQ',
                6: 'wK',
                16: 'bK',
            }
            for (let f = 0; f < 8; f++) {
                for (let r = 0; r < 8; r++) {
                    let piece = pieces[bytes[f*8 + r]];
                    if (piece !== null) {
                        pos[files[f] + (r+1)] = piece;
                    }
                }
            }
            console.log(pos);
            return pos;
        },
    }
}

// JUNK BELOW THIS LINE

import {clusterApiUrl, Connection, Keypair, PublicKey, Transaction, TransactionInstruction} from "@solana/web3.js";
import {serialize} from "borsh";
import * as fs from "fs";

export class ChessVoteInstructionData  {
  constructor(args) {
    this.instruction = 3;
    this.s_x = args.s_x;
    this.s_y = args.s_y;
    this.ply = args.ply;
    this.m_f = args.m_f;
    this.m_t = args.m_t;
    this.m_p = args.m_p;
  }
}

export const ChessVoteInstructionDataSchema = new Map([
  [
    ChessVoteInstructionData,
    {
      kind: "struct",
      fields: [
        ["instruction", "u8"],
        ["n_x", "u64"],
        ["n_y", "u64"],
        ["ply", "u16"],
        ["m_f", "u8"],
        ["m_t", "u8"],
        ["m_p", "u8"],
      ],
    },
  ],
]);

export const ChessVoteInstruction = async (
    connection,
    wallet,
    base,
    s_x,
    s_y,
    ply,
    m_f,
    m_t,
    m_p,
    chessBoard,
) => {
  const keys = [
    {
      pubkey: base,
      isSigner: false,
      isWritable: false,
    },
    {
      pubkey: wallet.publicKey,
      isSigner: true,
      isWritable: false,
    },
    {
      pubkey: wallet.publicKey,
      isSigner: false,
      isWritable: false,
    },
    {
      pubkey: wallet.publicKey,
      isSigner: false,
      isWritable: false,
    },
    {
      pubkey: chessBoard,
      isSigner: false,
      isWritable: true,
    },
  ];

  const args = new ChessVoteInstructionData({
    s_x,
    s_y,
    ply,
    m_f,
    m_t,
    m_p,
  });

  let data = Buffer.from(serialize(ChessVoteInstructionDataSchema, args));

  let Ix =
      [new TransactionInstruction({
        keys,
        programId: CHESS_PROGRAM_ID,
        data,
      })];

  return Ix;
};

export const sendTransactionWithRetryWithKeypair = async (
    connection,//: Connection,
    wallet,//: Keypair,
    instructions,//: TransactionInstruction[],
    signers,//: Keypair[],
    commitment,//: Commitment = "singleGossip",
    includesFeePayer,//: boolean = false,
    block,//?: BlockhashAndFeeCalculator,
    beforeSend,//?: () => void,
) => {
  const transaction = new Transaction({feePayer : includesFeePayer ? signers[0].publicKey : wallet.publicKey });

  instructions.forEach((instruction) => transaction.add(instruction));
  transaction.recentBlockhash = (
      block || (await connection.getRecentBlockhash(commitment))
  ).blockhash;

  transaction.partialSign(wallet);

  if (signers.length > 0) {
    transaction.partialSign(...signers);
  }

  if (beforeSend) {
    beforeSend();
  }

  const { txid, slot } = await sendSignedTransaction({
    connection,
    signedTransaction: transaction,
  });

  return { txid, slot };
};

export async function sendSignedTransaction({
                                              signedTransaction,
                                              connection,
                                              timeout = 30000,
                                            }) {
  const rawTransaction = signedTransaction.serialize();
  const startTime = getUnixTs();
  let slot = 0;
  const txid = await connection.sendRawTransaction(
      rawTransaction,
      {
        skipPreflight: true,
      }
  );

  console.log("Started awaiting confirmation for", txid);

  let done = false;
  await (async () => {
    while (!done && getUnixTs() - startTime < timeout) {
      await connection.sendRawTransaction(rawTransaction, {
        skipPreflight: true,
      });
      await sleep(500);
    }
  })();
  try {
    const confirmation = await awaitTransactionSignatureConfirmation(
        txid,
        timeout,
        connection,
        "recent",
        true
    );

    if (!confirmation)
      throw new Error("Timed out awaiting confirmation on transaction");

    if (confirmation.err) {
      console.log(confirmation.err);
      throw new Error("Transaction failed: Custom instruction error");
    }

    slot = confirmation.slot || 0;
  } catch (err) {
    console.log("Timeout Error caught", err);
    if (err.timeout) {
      throw new Error("Timed out awaiting confirmation on transaction");
    }
    let simulateResult = null;
    try {
      simulateResult = (
          await simulateTransaction(connection, signedTransaction, "single")
      ).value;
    } catch (e) {
      console.log("Simulate Transaction error", e);
    }
    if (simulateResult && simulateResult.err) {
      if (simulateResult.logs) {
        for (let i = simulateResult.logs.length - 1; i >= 0; --i) {
          const line = simulateResult.logs[i];
          if (line.startsWith("Program log: ")) {
            throw new Error(
                "Transaction failed: " + line.slice("Program log: ".length)
            );
          }
        }
      }
      throw new Error(JSON.stringify(simulateResult.err));
    }
  } finally {
    done = true;
  }

  console.log("Latency", txid, getUnixTs() - startTime);
  return { txid, slot };
}

export const getUnixTs = () => {
  return new Date().getTime() / 1000;
};

export function sleep(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function simulateTransaction(
    connection,
    transaction,
    commitment
) {
  transaction.recentBlockhash = await connection._recentBlockhash(
      connection._disableBlockhashCaching
  );

  const signData = transaction.serializeMessage();
  const wireTransaction = transaction._serialize(signData);
  const encodedTransaction = wireTransaction.toString("base64");
  const config = { encoding: "base64", commitment };
  const args = [encodedTransaction, config];

  const res = await connection._rpcRequest("simulateTransaction", args);
  if (res.error) {
    throw new Error("failed to simulate transaction: " + res.error.message);
  }
  return res.result;
}

async function awaitTransactionSignatureConfirmation(
    txid,//: TransactionSignature,
    timeout,//: number,
    connection,//: Connection,
    commitment,//: Commitment = "recent",
    queryStatus = false
) {
  let done = false;
  let status = {
    slot: 0,
    confirmations: 0,
    err: null,
  };
  let subId = 0;
  await(async() => {
    setTimeout(() => {
      if (done) {
        return;
      }
      done = true;
      console.log("Rejecting for timeout...");
    }, timeout);
    try {
      subId = connection.onSignature(
          txid,
          (result, context) => {
            done = true;
            status = {
              err: result.err,
              slot: context.slot,
              confirmations: 0,
            };
            if (result.err) {
              console.log("Rejected via websocket", result.err);
            }
          },
          commitment
      );
    } catch (e) {
      done = true;
      console.log("WS error in setup", txid, e);
    }
    while (!done && queryStatus) {
      await (async () => {
        try {
          const signatureStatuses = await connection.getSignatureStatuses([
            txid,
          ]);
          status = signatureStatuses && signatureStatuses.value[0];
          if (!done) {
            if (status.err) {
              console.log("REST error for", txid, status);
              done = true;
            } else if (!status.confirmations) {
              console.log("REST no confirmations for", txid, status);
            } else {
              done = true;
            }
          }
        } catch (e) {
          if (!done) {
            console.log("REST connection error: txid", txid, e);
          }
        }
      })();
      await sleep(2000);
    }
  })();

  if (connection._signatureSubscriptions[subId])
    await connection.removeSignatureListener(subId);
  done = true;
  return status;
}

async function voteMove(from, to, ply, turn) {
  console.log("VOTING MOVE");
  console.log("from", from, "to", to, "ply", ply, "turn", turn);
  const base_address = new PublicKey(BASE);
  const keypairString = (turn == 'b') ? WALLET_B : WALLET_W;
  const walletKeyPair = Keypair.fromSecretKey(
      new Uint8Array(JSON.parse(keypairString))
  );
  const solConnection = new Connection(clusterApiUrl(ENV));

  const ixs = await ChessVoteInstruction(
      solConnection,
      walletKeyPair,
      base_address,
      0,
      0,
      ply,
      from,
      to,
      0,
      new PublicKey(CHESSBOARD),
  );

  console.log(ixs);
  await sendTransactionWithRetryWithKeypair(
      solConnection,
      walletKeyPair,
      [...ixs],
      [walletKeyPair]
  );
  console.log("MOVE VOTED");
}
</script>

<style>
#board {
    width: 400px;
}
</style>