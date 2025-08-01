<script setup>
import { useMasternode } from '../composables/use_masternode.js';
import { storeToRefs } from 'pinia';
import { useWallets } from '../composables/use_wallet';
import NewMasternodeList from './NewMasternodeList.vue';
import Masternode from '../masternode.js';
import RestoreWallet from '../dashboard/RestoreWallet.vue';
import { cChainParams } from '../chain_params';
import Modal from '../Modal.vue';
import { ref, watch, onMounted } from 'vue';
import { getNetwork } from '../network/network_manager.js';
import { translation, ALERTS } from '../i18n.js';
import { generateMasternodePrivkey, parseIpAddress } from '../misc';
import { useAlerts } from '../composables/use_alerts.js';
import { COutpoint } from '../transaction.js';
import { valuesToComputed } from '../utils.js';

const { createAlert } = useAlerts();

/**
 * @type{{masternodes: import('vue').Ref<import('../masternode.js').default[]>}}
 */
const { masternodes } = storeToRefs(useMasternode());
const { activeWallet: wallet, activeVault, vaults } = storeToRefs(useWallets());
const showRestoreWallet = ref(false);
const showMasternodePrivateKey = ref(false);
const masternodePrivKey = ref('');
// Array of possible masternode UTXOs
const possibleUTXOs = ref(wallet.value.getMasternodeUTXOs());

const { balance, isViewOnly, isSynced, isHardwareWallet } =
    valuesToComputed(wallet);

watch(masternodes, (masternodes, oldValue) => {
    for (const vault of vaults.value) {
        for (const wallet of vault.wallets) {
            for (const oldMn of oldValue) {
                if (oldMn?.collateralTxId) {
                    wallet.unlockCoin(
                        new COutpoint({
                            txid: oldMn.collateralTxId,
                            n: oldMn.outidx,
                        })
                    );
                }
            }
            for (const masternode of masternodes) {
                if (masternode?.collateralTxId) {
                    wallet.lockCoin(
                        new COutpoint({
                            txid: masternode.collateralTxId,
                            n: masternode.outidx,
                        })
                    );
                }
            }
        }
    }
});
function updatePossibleUTXOs() {
    possibleUTXOs.value = wallet.value.getMasternodeUTXOs();
}

onMounted(() => {
    document
        .getElementById('masternodeTab')
        .addEventListener('click', updatePossibleUTXOs);
});

watch(isSynced, () => {
    updatePossibleUTXOs();
});
/**
 * Start a Masternode via a signed network broadcast
 * @param {boolean} fRestart - Whether this is a Restart or a first Start
 */
async function startMasternode(mn, fRestart = false) {
    if (
        !isHardwareWallet.value &&
        isViewOnly.value &&
        !(await restoreWallet())
    ) {
        return;
    }
    if (await mn.start()) {
        const strMsg = fRestart ? ALERTS.MN_RESTARTED : ALERTS.MN_STARTED;
        createAlert('success', strMsg, 4000);
    } else {
        const strMsg = fRestart
            ? ALERTS.MN_RESTART_FAILED
            : ALERTS.MN_START_FAILED;
        createAlert('warning', strMsg, 4000);
    }
}

async function destroyMasternode(mn) {
    console.log('HIIII');
    // TODO: Only delete 1
    masternodes.value = masternodes.value.filter(
        (masternode) => masternode.mnPrivateKey !== mn.mnPrivateKey
    );
}

/**
 * @param {string} privateKey - masternode private key
 * @param {string} ip - Ip to connect to. Can be ipv6 or ipv4
 * @param {import('../transaction.js').UTXO} utxo - Masternode utxo. Must be of exactly `cChainParams.current.collateralInSats` of value
 */
function importMasternode(privateKey, ip, utxo) {
    const address = parseIpAddress(ip);
    if (!address) {
        createAlert('warning', ALERTS.MN_BAD_IP, 5000);
        return;
    }
    if (!privateKey) {
        createAlert('warning', ALERTS.MN_BAD_PRIVKEY, 5000);
        return;
    }
    masternodes.value.push(
        new Masternode({
            walletPrivateKeyPath: wallet.value.getPath(utxo.script),
            mnPrivateKey: privateKey,
            collateralTxId: utxo.outpoint.txid,
            outidx: utxo.outpoint.n,
            addr: address,
        })
    );
}

async function restoreWallet() {
    if (!activeVault.value.isEncrypted) return false;
    if (wallet.value.isHardwareWallet) return true;
    showRestoreWallet.value = true;
    return await new Promise((res) => {
        watch(
            [showRestoreWallet, isViewOnly],
            () => {
                showRestoreWallet.value = false;
                res(!isViewOnly.value);
            },
            { once: true }
        );
    });
}

async function createMasternode({ isVPS }) {
    // Ensure wallet is unlocked
    if (!isHardwareWallet.value && isViewOnly.value && !(await restoreWallet()))
        return;
    console.log(cChainParams.current.collateralInSats);
    const [address] = wallet.value.getNewAddress(1);
    const res = await wallet.value.createAndSendTransaction(
        getNetwork(),
        address,
        cChainParams.current.collateralInSats
    );
    if (!res) {
        createAlert('warning', translation.ALERTS.TRANSACTION_FAILED);
        return;
    }

    if (isVPS) openShowPrivKeyModal();
    updatePossibleUTXOs();
}

function openShowPrivKeyModal() {
    masternodePrivKey.value = generateMasternodePrivkey();
    showMasternodePrivateKey.value = true;
}
</script>

<template>
    <RestoreWallet
        :show="showRestoreWallet"
        :wallet="wallet"
        @close="showRestoreWallet = false"
    />
    <!--
    <CreateMasternode
        v-if="!masternodes.length"
        :synced="isSynced"
        :balance="balance"
        :possibleUTXOs="possibleUTXOs"
        @createMasternode="createMasternode"
        @importMasternode="importMasternode"
    />
    <MasternodeController
        v-if="masternodes.length"
        :masternode="masternodes[0]"
        @start="({ restart }) => startMasternode(restart)"
        @destroy="destroyMasternode"
	 /> -->

    <NewMasternodeList
        :masternodes="masternodes"
        :possibleUTXOs="possibleUTXOs"
        :balance="balance"
        :synced="isSynced"
        @restartMasternode="(mn) => startMasternode(mn)"
        @deleteMasternode="(mn) => destroyMasternode(mn)"
        @createMasternode="createMasternode"
        @importMasternode="importMasternode"
    />
    <Modal :show="showMasternodePrivateKey">
        <template #header>
            <b>{{ translation?.ALERTS?.CONFIRM_POPUP_MN_P_KEY }}</b>
            <button
                @click="showMasternodePrivateKey = false"
                type="button"
                class="close"
                data-dismiss="modal"
                aria-label="Close"
            >
                <i class="fa-solid fa-xmark closeCross"></i>
            </button>
        </template>
        <template #body>
            <code>{{ masternodePrivKey }}</code>
            <span
                v-html="translation?.ALERTS?.CONFIRM_POPUP_MN_P_KEY_HTML"
            ></span>
        </template>
    </Modal>
</template>
