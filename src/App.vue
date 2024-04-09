<template>
  <div id="app">
    <div class="body">
      <p class="ibc-title">IBC Token Bridge</p>
      <div class="container">
        <button class="connect-btn btn" v-show="!metamaskConnected" @click="connectWallet">
          Connect metamask
        </button>
        <span v-show="metamaskConnected">
          {{ currentAccount }}
        </span>
      </div>
      <div class="body-content-container">
        <div class="body-balance">
          <span class="balance-common">
            Optimism balance: {{ balanceOfOp }} {{ symbolOfOp }}
          </span>
          <span class="balance-common">
            Base balance: {{ balanceOfBase }} {{ symbolOfBase }}
          </span>
        </div>
        <input v-show="metamaskIsInstalled" type="text" v-model="ibcAmount" class="bridge-input"
          placeholder="Your IBC token amount">
        <div class="btn-container" v-show="metamaskConnected">
          <button style="margin-right:20px;" class="transfer-btn btn" @click="transfer('op')">Transfer to Base</button>
          <button class="transfer-btn btn" @click="transfer('base')">Transfer to
            Optimism</button>
        </div>
        <button class="btn" style="height:38px;width:100%;" v-show="!metamaskConnected" @click="connectWallet">
          Connect metamask
        </button>
        <div class="loading-cls" v-show="isLoading">
          Loading...
        </div>
      </div>
      <p class="notice">Cross-chain transactions will be relatively slow, please be patient and wait for about 40s.</p>


    </div>
  </div>
</template>

<script>
import Web3 from 'web3';
import BigNumber from "bignumber.js";
import { CONTRACT_ABI } from './abi.js';

const OP_PORT_ADDRESS = '0xE525FAAA769d40506fc0261dAe775a0a41a7B257'
const BASE_PORT_ADDRESS = '0xC29d78Bd9db700Ad2dc5cABc7e410cF28bc2e8BB'

export default {
  name: 'App',
  data() {
    return {
      currentAccount: '',
      metamaskIsInstalled: false,
      metamaskConnected: false,
      ibcAmount: '',
      isLoading: false,
      balanceOfOp: '',
      balanceOfBase: '',
      symbolOfOp: '',
      symbolOfBase: '',
    }
  },
  async mounted() {
    this.checkMetamaskInstalled();
  },
  methods: {
    async checkMetamaskInstalled() {
      if (typeof window.ethereum !== 'undefined') {
        this.metamaskIsInstalled = true;
        if (window.ethereum.isConnected()) {
          const accounts = await ethereum.request({ method: 'eth_accounts' })
          if (accounts && accounts.length > 0) {
            this.currentAccount = accounts[0];
            this.metamaskConnected = true;
            this.getBalance();
          }
        }
      }
    },
    async transfer(chain) {
      try {
        const chainId = await ethereum.request({ method: 'eth_chainId' });
        if (chain === 'op' && chainId !== '0xaa37dc') {
          await window.ethereum.request({
            method: 'wallet_switchEthereumChain',
            params: [{ chainId: '0xaa37dc' }],
          });
        } else if (chain === 'base' && chainId !== '0x14a34') {
          await window.ethereum.request({
            method: 'wallet_switchEthereumChain',
            params: [{ chainId: '0x14a34' }],
          });
        }
        const web3 = new Web3(window.ethereum);
        await window.ethereum.enable();
        this.isLoading = true
        const receipt = await new web3.eth.Contract(
          CONTRACT_ABI, 
          chain === 'op' ? OP_PORT_ADDRESS : BASE_PORT_ADDRESS
          ).methods.crossChainTransfeer(
            chain === 'op' ? BASE_PORT_ADDRESS : OP_PORT_ADDRESS,
          web3.utils.padRight(web3.utils.asciiToHex(chain === 'op' ? 'channel-10' : 'channel-11'), 64),
          this.transfer2Erc20Decimal(this.ibcAmount)
        ).send({ from: this.currentAccount });
        console.log(receipt);
        setTimeout(() => {
          this.isLoading = false;
          this.getBalance()
        }, 40000)
      } catch (error) {
        this.isLoading = false;
        console.error(error);
      }
    },

    transfer2Erc20Decimal(number) {
      const result = new BigNumber(number).multipliedBy(new BigNumber(10).pow(18));
      return result.toNumber();
    },
    async connectWallet() {
      try {
        await window.ethereum.enable();
        const accountList = await new Web3(window.ethereum).eth.getAccounts();
        this.currentAccount = accountList[0]
        this.metamaskConnected = true;
        this.getBalance();
      } catch (error) {
        console.error(error);
      }

    },
    async getBalance() {
      try {
        const opContract = new (new Web3('https://sepolia.optimism.io')).eth.Contract(CONTRACT_ABI, OP_PORT_ADDRESS);
        const opBalance = await opContract.methods.balanceOf(
          this.currentAccount,
        ).call();
        const opSymbol = await opContract.methods.symbol().call();
        let opNum = new BigNumber(Number(opBalance));
        this.balanceOfOp = opNum.dividedBy('1e18');
        this.symbolOfOp = opSymbol
      } catch (error) {
        console.error(error);
      }
      try {
        const baseContract = new (new Web3('https://sepolia.base.org')).eth.Contract(CONTRACT_ABI, BASE_PORT_ADDRESS);
        const baseBalance = await baseContract.methods.balanceOf(
          this.currentAccount,
        ).call();
        const baseSymbol = await baseContract.methods.symbol().call();
        let baseBigNumber = new BigNumber(Number(baseBalance));
        this.balanceOfBase = baseBigNumber.dividedBy('1e18');
        this.symbolOfBase = baseSymbol
      } catch (error) {
        console.error(error);
      }
      this.ibcAmount = '';
    },
  }

}
</script>

<style lang="scss">
html,
body {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
}

#app {
  background: linear-gradient(139.73deg, #e5fdff, #f3efff);
  width: 100%;

  height: 100%;

  .body {
    width: 100%;
    height: 100%;
    padding-top: 20px;
    display: flex;
    flex-direction: column;
    align-items: center;
    color: #101112;

    .btn {
      background-color: #409eff;
      color: #fff;
      border: none;
      border-radius: 8px;
      cursor: pointer;
      outline: none;
      min-height: 28px;

      &:hover {
        background-color: #66b1ff;
      }
    }

    .ibc-title {
      font-size: 28px;

    }

    .container {
      margin-bottom: 10px;
      width: 100%;
      display: flex;
      justify-content: flex-end;
      padding-right: 40px;
      margin-bottom: 30px;
      height: 40px;

      .connect-btn {
        padding: 3px 6px;

      }

    }

    .body-content-container {
      display: flex;
      flex-direction: column;
      justify-content: space-between;
      align-items: center;
      height: 280px;
      width: 400px;
      color: #101112;
      background: #ffffff;
      border-radius: 20px;
      padding: 40px;
      position: relative;
      overflow: hidden;
      box-sizing: border-box;
      align-items: flex-start;

      .bridge-input {
        width: 100%;
        height: 22px;
        border: none;
        background: #f2f4f5;
        border-radius: 8px;
        height: 36px;
        text-indent: 10px;
      }

      .btn-container {
        display: flex;
        justify-content: space-between;
        width: 100%;

        .transfer-btn {
          flex: 1;
          width: 140px;
          height: 38px;
        }
      }

      .body-balance {
        display: flex;
        flex-direction: column;
      }

      .loading-cls {
        width: 100%;
        height: 100%;
        z-index: 100;
        position: absolute;
        background: rgba($color: #000000, $alpha: 0.5);
        color: #fff;
        font-size: 40px;
        top: 0;
        left: 0;
        display: flex;
        justify-content: center;
        align-items: center;
      }
    }

    .notice {
      font-size: 12px;
      margin-top: 20px;
      color: #FF0420;
    }




  }
}
</style>
