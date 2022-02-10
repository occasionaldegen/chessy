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
            chessboardAddress: new w3.PublicKey("DCN1KF82Pbfovs5k8zJw7rdXGRT2uhi4Ybw2FYYZvtj8"),
            chessboard: null,
            game: new Chess(),
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
            var start = 52;
            let chessboardBytes = chessAccountInfo.data.slice(start,start+73);
            let turn = {0: 'w', 1: 'b'}[chessboardBytes[72]];
            let pos = ChessBoard.objToFen(this.getPosition(chessboardBytes)) + ' ' + turn;
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
          if (move === null) return 'snapback'
        },
        onSnapEnd () {
          this.chessboard.position(this.game.fen())
        },
        getPosition(bytes) {
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
</script>

<style>
#board {
    width: 400px;
}
</style>