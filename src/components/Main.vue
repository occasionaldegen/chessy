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
    <div v-else>
        {{chessboard}}
    </div>
</div>
</template>

<script>
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
    data() {
        return {
            connection: null,
            chessboard: null,
            wallet: shallowRef(null),
            walletConnected: false,
            solBalance: null,
            selectedNetwork: "devnet",
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
            this.chessboard = 'hi';
        },
    }
}
</script>
