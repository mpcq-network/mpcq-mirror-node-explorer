<template>
  <div class="wallet-dialog-wrapper">
    
    <div v-if="walletManager.accountId.value" class="wallet-options">
      <div class="wallet-options-title">
        <div style="height: 40px">
          <EntityLink :route="accountRoute">
            <LabelView :icon-url="walletIconURL">
              <div style="display: flex; align-items: center; justify-content: center;">
                {{ accountId }}
                <span style="color: var(--text-secondary)">-{{ accountChecksum }}</span>
              </div>
            </LabelView>
          </EntityLink>
        </div>
      </div>

      <div class="wallet-options-content">
        <GroupBoxView>
          <template #groupBoxTitle>EVM Address</template>
          <template #default>
            <EVMAddress :address="accountEthereumAddress" :show-id="false"/>
          </template>
        </GroupBoxView>

        <GroupBoxView>
          <template #groupBoxTitle>Balance</template>
          <template #default>
            <HbarAmount :amount="tbarBalance"/>
            <div style="color: var(--text-secondary)">
              <HbarExtra :hide-zero="false" :tbar-amount="tbarBalance ?? 0"/>
            </div>
          </template>
        </GroupBoxView>

        <GroupBoxView>
          <template #groupBoxTitle>Account Operations</template>
          <template #default>
            <div class="account-operations">
              <div class="operation" @click="onUpdateAccount">
                <UserRoundPen :size="18"/>
                Account Update
              </div>
              <div class="operation" @click="onApproveAllowance">
                <CheckCheck :size="18"/>
                Approve Allowance
              </div>
            </div>
          </template>
        </GroupBoxView>
      </div>

      <div class="wallet-options-footer">
        <ButtonView @action="handleDisconnect">DISCONNECT WALLET</ButtonView>
        <AccountSelector :account-ids="accountIds"
                         :model-value="accountId"
                         @update:model-value="handleChangeAccount"/>
      </div>
    </div>

    <div v-else class="connect-wallet-container">
      <div class="wallet-grid">
        <button 
          v-for="provider in walletManager.providers.value" 
          :key="provider.info.uuid"
          class="wallet-option"
          @click="handleConnect(provider.info.uuid)"
        >
          <img :src="provider.info.icon" :alt="provider.info.name" class="wallet-icon" />
          <span class="wallet-name">{{ provider.info.name }}</span>
        </button>

        <button 
          v-if="walletConnectID"
          class="wallet-option"
          @click="handleConnect(null)"
        >
          <img src="https://altcoinsbox.com/walletconnect-logo/" alt="WalletConnect" class="wallet-icon" />
          <span class="wallet-name">WalletConnect</span>
        </button>
      </div>
      
      <div class="wallet-options-footer">
        <ButtonView @action="() => showWalletOptions = false">CANCEL</ButtonView>
      </div>
    </div>

    <UpdateAccountDialog
        v-model:show-dialog="showUpdateAccountDialog"
        @updated="onUpdateCompleted"
    />
    <ApproveAllowanceDialog 
        v-model:show-dialog="showApproveAllowanceDialog"
        @allowance-approved="onAllowanceApproved"
    />

  </div>
</template>

<script setup lang="ts">
import {computed, onBeforeUnmount, onMounted, ref} from "vue";
import GroupBoxView from "@/elements/GroupBoxView.vue";
import ButtonView from "@/elements/ButtonView.vue";
import LabelView from "@/elements/LabelView.vue";
import {AccountLocParser} from "@/utils/parser/AccountLocParser.ts";
import {NetworkConfig} from "@/config/NetworkConfig.ts";
import {BalanceAnalyzer} from "@/utils/analyzer/BalanceAnalyzer.ts";
import HbarExtra from "@/components/values/HbarExtra.vue";
import EntityLink from "@/components/values/link/EntityLink.vue";
import EVMAddress from "@/components/values/EVMAddress.vue";
import UpdateAccountDialog from "@/dialogs/UpdateAccountDialog.vue";
import ApproveAllowanceDialog from "@/dialogs/allowance/ApproveAllowanceDialog.vue";
import {CheckCheck, UserRoundPen} from 'lucide-vue-next';
import HbarAmount from "@/components/values/HbarAmount.vue";
import AccountSelector from "@/components/page/header/wallet/AccountSelector.vue";
import router, {routeManager, walletManager} from "@/utils/RouteManager.ts";

// Props / Models
const showWalletOptions = defineModel("showWalletOptions", {
  type: Boolean,
  required: true
})

// Logic & Composition
const networkConfig = NetworkConfig.inject()
const accountLocParser = new AccountLocParser(walletManager.accountId, networkConfig)
onMounted(() => accountLocParser.mount())
onBeforeUnmount(() => accountLocParser.unmount())

const balanceAnalyzer = new BalanceAnalyzer(accountLocParser.accountId, 10000)
onMounted(() => balanceAnalyzer.mount())
onBeforeUnmount(() => balanceAnalyzer.unmount())

// Computed
const walletConnectID = walletManager.routeManager.walletConnectID;
const accountRoute = computed(() => {
  const id = walletManager.accountId.value
  return id !== null ? routeManager.makeRouteToAccount(id) : null
})

const walletIconURL = computed(() => walletManager.walletIconURL.value ?? undefined)
const accountId = computed(() => walletManager.accountId.value ?? "No account ID")
const accountChecksum = computed(() => accountLocParser.accountChecksum ?? "")
const accountIds = walletManager.accountIds
const accountEthereumAddress = accountLocParser.ethereumAddress
const tbarBalance = balanceAnalyzer.hbarBalance
const currentNetwork = routeManager.currentNetwork

// Dialog states
const showUpdateAccountDialog = ref(false)
const onUpdateAccount = () => showUpdateAccountDialog.value = true
const onUpdateCompleted = () => console.log('Account update completed')

const showApproveAllowanceDialog = ref(false)
const onApproveAllowance = () => showApproveAllowanceDialog.value = true
const onAllowanceApproved = () => console.log('Approve Allowance completed')

// Handlers
const handleConnect = async (uuid: string | null) => {
  const success = await walletManager.connect(uuid);
  if (success) {
    // Keep dialog open to show account info, or close it:
    // showWalletOptions.value = false 
  }
}

const handleDisconnect = async () => {
  showWalletOptions.value = false
  await walletManager.disconnect()
}

const handleChangeAccount = (accountId: string) => {
  walletManager.selectAccountId(accountId)
  if (walletManager.accountId.value) {
    router.push(routeManager.makeRouteToAccount(walletManager.accountId.value))
  }
}
</script>

<style scoped>
/* Main Options View */
div.wallet-options {
  align-items: stretch;
  display: flex;
  flex-direction: column;
  row-gap: 24px;
}

div.wallet-options-title {
  align-items: center;
  border-bottom: 1px solid var(--network-theme-color);
  color: var(--text-primary);
  display: flex;
  padding-bottom: 16px;
}

div.wallet-options-content {
  display: flex;
  flex-direction: column;
  row-gap: 8px;
}

div.account-operations {
  display: flex;
  flex-direction: column;
  gap: 8px;
  cursor: pointer;
}

div.operation {
  align-items: center;
  display: flex;
  gap: 8px;
}

div.operation:hover {
  color: grey;
}

/* Connect Grid View */
.wallet-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 16px;
  padding: 10px 0 20px 0;
}

.wallet-option {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  border: 1px solid var(--border-secondary, #eee);
  padding: 20px;
  cursor: pointer;
  background: transparent;
  border-radius: 12px;
  transition: transform 0.1s ease;
  color: var(--text-primary);
}

.wallet-option:hover {
  background: var(--button-hover-bg, #f9f9f9);
  transform: translateY(-2px);
}

.wallet-icon {
  width: 40px;
  height: 40px;
  margin-bottom: 12px;
}

.wallet-name {
  font-weight: 500;
  font-size: 0.9rem;
}

div.wallet-options-footer {
  align-items: center;
  justify-content: space-between;
  display: flex;
  margin-top: 10px;
}
</style>