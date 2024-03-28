<template>
  <div class="container text-center">

    <div class="home-placeholder-div"></div>

    <div class="text-align-header mt-4 mb-2">
      <h1>Mint a 5+ character {{getTldName}} name for free!</h1>
      <p>
        If you hold a <a href="https://www.rareboard.com/blast-sniperz" target="_blank">BlastSniperz NFT</a>.
      </p>
    </div>

    <div v-if="!isEligible">
      <div class="d-flex justify-content-center domain-input-container mb-3 mt-5">
        <div class="mb-3">
          <label for="nftIdInputField" class="form-label">Enter your BlastSniperz NFT ID</label>
          <input 
            type="number"
            pattern="[0-9]*" 
            step="1" 
            class="form-control form-control-lg" 
            id="nftIdInputField" 
            v-model="nftId"
            placeholder="Enter your NFT ID"
          />
        </div>
      </div>

      <button 
        class="btn btn-secondary btn-lg mt-1 buy-button" 
        :disabled="!nftId || waitingEligibility"
        @click="checkEligibility"
      >
        <span v-if="waitingEligibility" class="spinner-border spinner-border-sm mx-1" role="status" aria-hidden="true"></span>
        <span>Check eligibility</span>
      </button>
    </div>

    <div v-if="isEligible">
      <div class="d-flex justify-content-center domain-input-container mb-3 mt-5">
        <div class="input-group domain-input input-group-lg input-sizing">
          <input
            v-model="chosenDomainName" 
            placeholder="enter name"
            type="text" 
            class="form-control text-end border-2 border-end-0 border-light"
            aria-label="Text input with dropdown button"
          >

          <span class="input-group-text tld-addon border-2 border-light">
            <span v-if="loading" class="spinner-border spinner-border-sm mx-1" role="status" aria-hidden="true"></span>
            <span>{{getTldName}}</span>
          </span>
        </div>
      </div>

      <p class="error">
        <small v-if="buyNotValid(chosenDomainName).invalid">
          <em>{{ buyNotValid(chosenDomainName).message }}</em>
        </small>
      </p>

      <div class="text-align-header">
        <p class="mt-5 price-text">
            5+ char domain price: FREE
        </p>
      </div>

      <!-- Minter contract paused -->
      <button v-if="isActivated && getMinterPaused && !getMinterLoadingData" class="btn btn-secondary btn-lg mt-3 buy-button" :disabled="true">
        <span v-if="getMinterPaused">Minting paused</span>
      </button>

      <!-- Minter contract loading data -->
      <button v-if="isActivated && isNetworkSupported && getMinterLoadingData" class="btn btn-secondary btn-lg mt-3 buy-button" :disabled="true">
        <span class="spinner-border spinner-border-sm mx-1" role="status" aria-hidden="true"></span>
        <span>Loading data</span>
      </button>

      <!-- Mint free domain -->
      <button 
        v-if="isActivated && isNetworkSupported && getCanUserBuy && isEligible && !getMinterPaused && !getMinterLoadingData" 
        class="btn btn-secondary btn-lg mt-3 buy-button" 
        @click="mintFreeDomain" 
        :disabled="waiting || buyNotValid(chosenDomainName).invalid || !isCorrectLength"
      >
        <span v-if="waiting" class="spinner-border spinner-border-sm mx-1" role="status" aria-hidden="true"></span>
        <span>Mint free domain</span>
      </button>
    </div>

    <!-- Connect Wallet -->
    <button v-if="!isActivated" class="btn btn-secondary btn-lg mt-3 btn-Disconnected" data-bs-toggle="modal" data-bs-target="#connectModal">Connect wallet</button>

    <div v-if="isActivated && !isNetworkSupported" class="mt-4">
      <button class="btn btn-secondary btn-lg btn-Disconnected" @click="changeNetwork(this.getTldChainName)">Switch to {{getTldChainName}}</button>
    </div>
    
  </div>

</template>

<script>
import { ethers } from 'ethers';
import { useBoard, useEthers } from 'vue-dapp';
import { mapActions, mapGetters, mapMutations } from 'vuex';
import { useToast, TYPE } from "vue-toastification";
import WaitingToast from "../components/toasts/WaitingToast.vue";
import useDomainHelpers from "../hooks/useDomainHelpers";
import useChainHelpers from "../hooks/useChainHelpers";

export default {
  name: "NftFreeDomain",

  data() {
    return {
      chosenDomainName: null,
      chosenAllowance: null,
      isEligible: false,
      loading: false, // loading data
      minterContract: null,
      nftId: null,
      waiting: false, // waiting for TX to complete
      waitingEligibility: false, // waiting for TX to complete
    }
  },

  computed: {
    ...mapGetters("user", ["getNftAddress", "getPaymentTokenAddress", "getPaymentTokenName", "getPaymentTokenAllowance", "getUserBalance", "getCanUserBuy", "getDiscountEligible"]),
    ...mapGetters("network", ["getBlockExplorerBaseUrl"]),
    ...mapGetters("tld", ["getTldChainId", "getTldChainName", "getMinterAddress", "getTldContract", "getMinterLoadingData", "getMinterPaused", "getTldName"]),

    domainLowerCase() {
      return this.chosenDomainName.toLowerCase();
    },

    isCorrectLength() {
      // domain must be 4 chars or longer
      if (this.chosenDomainName) {
        return this.chosenDomainName.length >= 5;
      }

      return false;
    },

    isNetworkSupported() {
      if (this.isActivated) {
        if (this.chainId === this.getTldChainId) {
          return true;
        }
      }

      return false;
    }
  },

  methods: {
    ...mapActions("user", ["fetchCanUserBuy", "getPaymentTokenDecimals"]),
    ...mapMutations("user", ["addDomainManually", "setPaymentTokenAllowance"]),

    async checkEligibility() {
      // check if user is eligible for a free domain
      this.waitingEligibility = true;

      if (!this.nftId || this.nftId > 1427 || this.nftId < 1) {
        this.toast("Please enter a valid NFT ID.", {type: TYPE.ERROR});
        this.waitingEligibility = false;
        return;
      }

      const minterInterface = new ethers.utils.Interface([
        "function freeMintEligibility(uint256 nftId_, address user_) external view returns(bool alreadyMinted_, bool isOwner_)"
      ]);

      const minterContract = new ethers.Contract(this.getMinterAddress, minterInterface, this.signer);
      
      const eligibilityData = await minterContract.freeMintEligibility(this.nftId, this.address);

      if (eligibilityData.isOwner_ && !eligibilityData.alreadyMinted_) {
        this.isEligible = true;
      } else if (eligibilityData.alreadyMinted_) {
        this.isEligible = false;
        this.toast("NFT with ID " + this.nftId + " has already been used to mint a free domain.", {type: TYPE.ERROR});
      } else if (!eligibilityData.isOwner_) {
        this.isEligible = false;
        this.toast("NFT with ID " + this.nftId + " does not belong to you.", {type: TYPE.ERROR});
      }

      this.waitingEligibility = false;
    },

    async mintFreeDomain() {
      this.waiting = true;
      const fullDomainName = this.domainLowerCase + this.getTldName;

      // check if domain already minted
      const existingHolder = await this.getTldContract.getDomainHolder(this.domainLowerCase);

      if (existingHolder !== ethers.constants.AddressZero) {
        this.toast("Sorry, but this domain name is already taken...", {type: TYPE.ERROR});
        this.waiting = false;
        return;
      }

      // wrapper contract (with signer)
      const wrapperIntfc = new ethers.utils.Interface([
        "function holderFreeMint(string memory _domainName, address _domainHolder, uint256 _nftId) external nonReentrant returns(uint256)"
      ]);
      const minterContractSigner = new ethers.Contract(this.getMinterAddress, wrapperIntfc, this.signer);

      try {
        const tx = await minterContractSigner.holderFreeMint(
          this.domainLowerCase,
          this.address,
          this.nftId
        );

        const toastWait = this.toast(
          {
            component: WaitingToast,
            props: {
              text: "Please wait for your transaction to confirm. Click on this notification to see transaction in the block explorer."
            }
          },
          {
            type: TYPE.INFO,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          }
        );

        const receipt = await tx.wait();

        if (receipt.status === 1) {
          this.toast.dismiss(toastWait);
          this.toast("You have successfully claimed a free domain!", {
            type: TYPE.SUCCESS,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          });
          this.addDomainManually(fullDomainName);
          this.isEligible = false;
          this.chosenDomainName = null;
          this.nftId = null;

          this.waiting = false;
        } else {
          this.toast.dismiss(toastWait);
          this.toast("Transaction has failed.", {
            type: TYPE.ERROR,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          });
          console.log(receipt);
          this.waiting = false;
        }

      } catch (e) {
        console.log(e)
        this.waiting = false;
        this.toast(e.message, {type: TYPE.ERROR});
      }

      this.waiting = false;
    },

    changeNetwork(networkName) {
      this.switchOrAddChain(window.ethereum, networkName);
    },

  },

  setup() {
    const { open } = useBoard();
    const { address, chainId, isActivated, signer } = useEthers();
    const toast = useToast();
    const { buyNotValid } = useDomainHelpers();
    const { switchOrAddChain } = useChainHelpers();

    return { address, buyNotValid, chainId, isActivated, open, signer, switchOrAddChain, toast }
  }
}
</script>


<style scoped>
.and {
  font-size: 1.7em;
  vertical-align: bottom;
  padding-left: 0.2em;
  padding-right: 0.1em;
}

.buy-button {
  margin-bottom: 20px;
}

.container {
  padding-top: 80px;
  padding-bottom: 50px;
}

.domain-input {
  width: 50%;
}

.domain-input > input {
  color: #0D0F1A;
}

.domain-input > input::placeholder {
  color: #0D0F1A;
  opacity: 0.7;
}

.domain-input-container {
  margin-top: 30px;
}

.error {
  color: #DBDFEA;
}

.main-img {
  width: 6em;
}

th:first-of-type {
  border-top-left-radius: 10px;
  border-collapse: separate;
}
th:last-of-type {
  border-top-right-radius: 10px;
}
tr:last-of-type td:first-of-type {
  border-bottom-left-radius: 10px;
}
tr:last-of-type td:last-of-type {
  border-bottom-right-radius: 10px;
}

.table-light {
  color: #0D0F1A;
}

.table-ppl {
  border: 2px solid white;
  border-radius: 10px !important;
  border-collapse: separate;
  color: white;
}

.tld-addon {
  background-color: white;
  color:#0D0F1A;
}

@media only screen and (max-width: 767px) {
  .container {
    padding-top: 0px;
    padding-bottom: 50px;
  }

  .domain-input {
    width: 100%;
  }
}
</style>