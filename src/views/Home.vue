<template>
  <div class="container">
    <br /><br />
    <img src="/logo.svg" width="100" /><br /><br />
    --<br /><br />
    <div v-if="!account">
      <b-button type="is-primary" v-on:click="connect()"
        >Connect to OpenSea</b-button
      >
    </div>
    <div v-if="account">
      {{ account }}<br />
      {{ balance }} ETH <br /><br />
      --
      <br /><br />
      <div v-if="order">
        <b-button type="is-primary" v-on:click="buyItem()">BUY ITEM</b-button
        ><br /><br />
        <pre
          style="
            text-align: left;
            font-size: 11px;
            border-radius: 5px;
            border: 1px solid #bbb;
          "
          >{{ order }}</pre
        >
      </div>
      <div v-if="!order">No orders found.</div>
      --
      <br /><br />
      <b-button type="is-primary" v-on:click="disconnect()"
        >Disconnect</b-button
      >
    </div>
  </div>
</template>
<script>
import Web3 from "web3";
import { OpenSeaPort, Network } from "opensea-js";
import { OrderSide } from "opensea-js/lib/types";

process.versions = { node: "", v8: "" };

export default {
  name: "Home",
  data() {
    return {
      account: "",
      web3: "",
      balance: 0,
      order: "",
    };
  },
  async mounted() {
    const app = this;
    if (window.ethereum) {
      app.web3 = await new Web3(window.ethereum);
      const accounts = await app.web3.eth.getAccounts();
      if (accounts !== undefined && accounts.length > 0) {
        if (accounts[0] === localStorage.getItem("connected")) {
          const balance = await app.web3.eth.getBalance(accounts[0]);
          app.account = accounts[0];
          app.balance = parseFloat(
            app.web3.utils.fromWei(balance, "ether")
          ).toFixed(10);
          app.getOrders();
        }
      }
    }
  },
  methods: {
    async connect() {
      const app = this;
      if (window.ethereum) {
        // Check if network is desired one
        app.selected_network = await app.web3.eth.net.getId();
        // Request accounts
        await window.ethereum.send("eth_requestAccounts");
        // Read accounts
        const accounts = await app.web3.eth.getAccounts();
        if (accounts[0] !== undefined) {
          app.account = accounts[0];
          // Take balance
          const balance = await app.web3.eth.getBalance(accounts[0]);
          app.balance = parseFloat(
            app.web3.utils.fromWei(balance, "ether")
          ).toFixed(10);
          localStorage.setItem("connected", app.account);
          this.getOrders();
        }
      } else {
        alert("Please install Metamask");
      }
    },
    async disconnect() {
      const app = this;
      localStorage.removeItem("connected");
      app.account = "";
      app.balance = 0;
      location.reload();
    },
    async getOrders() {
      const app = this;
      if (
        window.ethereum &&
        window.web3.currentProvider &&
        this.account.length > 0
      ) {
        const seaport = new OpenSeaPort(window.web3.currentProvider, {
          networkName: Network.Main,
        });
        console.log(
          "GETTING ORDERS FOR:",
          process.env.VUE_APP_CONTRACT,
          process.env.VUE_APP_TOKENID
        );
        const { orders } = await seaport.api.getOrders({
          asset_contract_address: process.env.VUE_APP_CONTRACT,
          token_id: process.env.VUE_APP_TOKENID,
          side: OrderSide.Sell,
        });
        if (orders[0] !== undefined) {
          console.log("ORDER FOUND:", orders[0]);
          app.order = orders[0];
        }
      }
    },
    async buyItem() {
      const app = this;
      const order = app.order;
      const accountAddress = app.account;
      const seaport = new OpenSeaPort(window.web3.currentProvider, {
        networkName: Network.Main,
      });
      console.log("BUYING WITH ACCOUNT:", accountAddress);
      try {
        const transactionHash = await seaport.fulfillOrder({
          order,
          accountAddress,
        });
        console.log("TRANSACTION HASH:", transactionHash);
      } catch (e) {
        alert(e.message);
      }
    },
  },
};
</script>
