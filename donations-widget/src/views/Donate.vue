<template>
  <div>
    <p>
      This is <strong>the</strong> donation app. <em>Donate and/or bequest</em> here for the software and the free
      market to choose the best donation recepient.
    </p>
    <p style="color: red">
      This is demo version for a testnet. Contracts are not enough tested and audited yet.
      <a target="_blank" href="https://gitcoin.co/grants/1591/science-of-the-future-the-100-years-forward-plan">Donate</a>
      for contract audit!
    </p>
    <NetworkInfo :chainid="chainid" :networkname="networkname" :web3="web3"/>
    <p>
      <label>
        <input
          type="radio"
          name="paymentKind"
          @click="setPaymentKind('donate')"
          checked=""
        />
        &nbsp;Donate
      </label>
      <label>
        <input
          type="radio"
          name="paymentKind"
          @click="setPaymentKind('bequestSafe')"
          checked=""
        />
        &nbsp;Bequest all funds on a smart wallet
      </label>
    </p>
    <p :style="{display: tokenDisplayBlock}">
      Donation in:
      <label>
        <input
          type="radio"
          name="tokenKind"
          @click="setTokenKind('eth')"
        />
        &nbsp;ETH
      </label>
      <small>(recommended)</small>
      {{' '}}
      <label>
        <input
          type="radio"
          name="tokenKind"
          @click="setTokenKind('erc1155')"
        />
        &nbsp;ERC-1155
      </label>
      <small>(recommended)</small>
      {{' '}}
      <label>
        <input type="radio" name="tokenKind" @click="setTokenKind('erc20')" />
        &nbsp;ERC-20
      </label>
      <br />
      <small>(Don't use stablecoins for long-time funding.)</small>
    </p>
    <p :style="{display: tokenDisplayInline}">
      <span>Token address:</span>
      {{' '}}
      <EthAddress v-model="tokenEthAddress"/>
    </p>
    <p :style="{display: walletDisplayInline}">
      Gnosis Safe address:
      {{' '}}
      <EthAddress v-model="safeAddress"/>
      <br/>
      <button
        target="_blank"
        rel="noopener noreferrer"
        @click="bequest"
        :disabled="bequestDisabled"
        class="donateButton"
      >
        Bequest all funds on a Gnosis Safe
      </button>
    </p>
    <p :style="{display: paymentKind !== 'bequestSafe' && tokenKind === 'erc1155' ? 'block' : 'none'}">
      Token ID:
      <Uint256 v-model="tokenId"/>
    </p>
    <div :style="{display: tokenDisplayBlock}">
      <p>
        Donation amount:
        <Amount v-model="amount"/>
        {{' '}}
        <button @click="donate()" :disabled="donateButtonDisabled">Donate</button>
      </p>
    </div>
  </div>
</template>

<script>
import Web3 from 'web3';
// MEWConnect does not work on Firefox 84.0 for Ubuntu.
// import Web3Modal from "web3modal";
// import MewConnect from '@myetherwallet/mewconnect-web-client';

import validators from '../utils/validators'

const { toBN, toWei } = Web3.utils;

import EthAddress from '@/components/EthAddress.vue'
import Uint256 from '@/components/Uint256.vue'
import Amount from '@/components/Amount.vue'
import NetworkInfo from '@/components/NetworkInfo.vue'
import { mySend, getABIs, getAccounts, getAddresses } from '../utils/AppLib'

import erc1155Abi from '../utils/ERC1155Abi';
import erc20Abi from '../utils/ERC20Abi';

export default {
  name: 'Donate',
  props: [
    'prefix',
    'chainid',
    'networkname',
    'providerurl',
    'web3Getter',
  ],
  components: {
    EthAddress,
    Uint256,
    Amount,
    NetworkInfo,
  },
  watch: {
    amount() {
      this.setDonateButtonDisabled();
    },
    paymentKind() {
      this.setDonateButtonDisabled();
      this.updateWalletTokenDisplay();
    },
    tokenKind() {
      this.setDonateButtonDisabled();
      this.updateWalletTokenDisplay();
    },
    tokenEthAddress() {
      this.setDonateButtonDisabled();
    },
    tokenId() {
      this.setDonateButtonDisabled();
    },
    safeAddress() {
      this.setBequestURI();
    },
    gnosisBequestApp() {
      this.setBequestURI();
    },
    networkname() {
      const self = this
      async function doIt() {
        self.web3 = self.web3Getter ? await self.web3Getter() : window.web3 // Duplicate code
        const abis = await self.myGetAddresses(self.prefix);
        self.oracleId = abis ? abis.oracleId : null;
        self.gnosisBequestApp = abis ? abis.gnosisBequestApp : null; // FIXME: Use the same app for all networks.
      }
      doIt();
    },
  },
  data() {
    return {
      web3: null,

      oracleId: null, // TODO: should be a property instead

      paymentKind: 'bequestSafe',
      tokenKind: '',
      tokenEthAddress: '',
      tokenId: '',
      amount: '',
      safeAddress: '',
      gnosisBequestApp: null,
      gnosisBequestAppInSafe: '',
      donateButtonDisabled: true,
      bequestDisabled: true,
      
      // TODO: too many:
      walletDisplayInline: 'inline',
      walletDisplayBlock: 'block',
      tokenDisplayInline: 'none',
      tokenDisplayBlock: 'none',
    }
  },
  created() {
    const self = this
    self.myGetAddresses(self.prefix)
      .then(function(abis) {
        if (abis) {
          self.oracleId = abis.oracleId
        }
      })
    window.donateComponent = self // bug workaround used in GitCoin
  },
  methods: {
    async myGetAddresses(PREFIX) {
      return await getAddresses(PREFIX, this.networkname)
    },
    async obtainERC1155Token() {
      let collateralContractEthAddress, collateralTokenId;
      switch(this.tokenKind) {
        case 'erc1155':
          collateralContractEthAddress = this.tokenEthAddress;
          collateralTokenId = this.tokenId;
          break;
        case 'erc20':
          collateralContractEthAddress = (await this.myGetAddresses(this.prefix)).ERC1155OverERC20.address;
          collateralTokenId = Web3.utils.toHex(this.tokenEthAddress);

          {
            const web3 = await this.getWeb3();
            // if (web3 === null) return;
            const account = (await getAccounts(web3))[0];
            // if(!account) return;

            // Approve ERC-20 spent
            const erc20 = new web3.eth.Contract(erc20Abi, this.tokenEthAddress);
            const allowanceStr = await erc20.methods.allowance(account, collateralContractEthAddress).call();
            const allowance = toBN(allowanceStr);
            const halfBig = toBN(2).pow(toBN(128));
            if(allowance.lt(halfBig)) {
              const big = toBN(2).pow(toBN(256)).sub(toBN(1));
              const tx = await mySend(await this.getWeb3(), erc20, erc20.methods.approve, [collateralContractEthAddress, big.toString()], {from: account}, null)
                // .catch(e => alert(e.message));
              await tx;
            }
          }
          break;
      }
      return [collateralContractEthAddress, collateralTokenId];
    },
    async lockContract() {
      const addresses = await this.myGetAddresses(this.prefix);
      return addresses.SalaryWithDAO.address;
    },
    setBequestURI() {
      this.bequestDisabled = !validators.isEthAddressValid(this.safeAddress);
      const safeUI = `https://rinkeby.gnosis-safe.io/app/#/safes/${this.safeAddress}`;
      this.gnosisBequestAppInSafe =
        `${safeUI}/apps?appUrl=` + window.encodeURIComponent(this.gnosisBequestApp);
    },
    async donateETH() {
      const wei = toWei(this.amount);
      const web3 = await this.getWeb3();
      if (web3 !== null) {
        try {
          const contractAddress = (await this.myGetAddresses(this.prefix)).DonateETH.address;
          const abi = (await getABIs(this.prefix)).DonateETH;
          const contract = new web3.eth.Contract(abi, contractAddress);
          const account = (await getAccounts(web3))[0];
          if(!account) {
            // setConnectedToAccount(false); // TODO
            return;
          }
          await mySend(await this.getWeb3(), contract, contract.methods.donate,
            [this.oracleId,
             account,
             []],
            {from: account, value: wei}, null
          );
        }
        catch(e) {
          alert(e.message);
        }
      }
    },
    async donateToken() {
      const wei = toWei(this.amount);
      const web3 = await this.getWeb3();
      if (web3 !== null) {
        try {
          const addresses = await this.myGetAddresses(this.prefix);
          if (!addresses) return;
          const contractAddress = await this.lockContract();
          const scienceAbi = (await getABIs(this.prefix)).SalaryWithDAO;
          const science = new web3.eth.Contract(scienceAbi, addresses.SalaryWithDAO.address);
          const account = (await getAccounts(web3))[0];
          if(!account) {
            // setConnectedToAccount(false); // TODO
            return;
          }
          const [collateralContractAddress, collateralTokenId] = await this.obtainERC1155Token();
          const collateralContract = new web3.eth.Contract(erc1155Abi, collateralContractAddress);
          const approved = await collateralContract.methods.isApprovedForAll(account, contractAddress).call();
          if (!approved) {
            const tx = await mySend(
              await this.getWeb3(), collateralContract, collateralContract.methods.setApprovalForAll,
              [contractAddress, true], {from: account}, null
            );
            await tx;
          }
          await mySend(await this.getWeb3(), science, science.methods.donate,
            [collateralContractAddress,
             collateralTokenId,
             this.oracleId,
             wei,
             account,
             account,
             []],
            {from: account}, null
          );
        }
        catch(e) {
          alert(e.message);
        }
      }
    },
    async donate() {
      if (this.tokenKind === 'eth') {
        this.donateETH();
      } else {
        this.donateToken();
      }
    },
    setDonateButtonDisabled() {
      this.donateButtonDisabled =
        !validators.isRealNumber(this.amount) || this.paymentKind === '' || this.tokenKind === '' ||
        (this.tokenKind !== 'eth' && !validators.isEthAddressValid(this.tokenEthAddress)) ||
        (this.tokenKind === 'erc1155' && !validators.isUint256Valid(this.tokenId));
    },
    setPaymentKind(value) {
      this.paymentKind = value;
    },
    setTokenKind(value) {
      this.tokenKind = value;
    },
    setTokenEthAddress(value) {
      this.tokenEthAddress = value;
    },
    setTokenId(value) {
      this.tokenId = value;
    },
    setAmount(value) {
      this.value = value;
    },
    updateWalletTokenDisplay() {
      this.walletDisplayInline = this.paymentKind === 'bequestSafe' ? 'inline' : 'none'
      this.walletDisplayBlock = this.paymentKind === 'bequestSafe' ? 'block' : 'none'
      this.tokenDisplayInline = this.paymentKind !== 'bequestSafe' && this.tokenKind !== 'eth' ? 'inline' : 'none'
      this.tokenDisplayBlock = this.paymentKind !== 'bequestSafe' ? 'block' : 'none'
    },
    isPastDate(date) {
      const currentDate = new Date();
      return date < currentDate;
    },
    bequest() {
      window.open(this.gnosisBequestAppInSafe);
    },
    async getWeb3() {
      return this.web3 = this.web3Getter ? await this.web3Getter() : window.web3 // Duplicate code
    },
  },
}
</script>

<style src="../assets/Donate.css"></style>
