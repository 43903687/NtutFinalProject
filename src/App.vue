<template>
  <main class="app-shell">
    <header class="hero">
      <nav class="topbar" aria-label="Main navigation">
        <div class="brand-mark">NTUT</div>
        <button class="wallet-button" type="button" @click="connectWallet" :disabled="isLoading">
          {{ isConnected ? formatAddress(account) : 'Connect Wallet' }}
        </button>
      </nav>

      <section class="hero-grid">
        <div class="hero-copy">
          <p class="eyebrow">Sepolia Testnet DApp</p>
          <h1>北科畢業模擬器</h1>
          <p class="subtitle-en">NTUT Graduation Simulator</p>
          <p class="hero-subtitle">花 10 NTUT，跑完你的四年北科人生。</p>
          <button class="hero-play-button" type="button" @click="playGame" :disabled="!canPlay">
            <span>開始模擬</span>
            <strong>PLAY COST 10 NTUT</strong>
          </button>
          <p v-if="isConnected && isCorrectNetwork && !hasEnoughNtut" class="hero-hint">NTUT 不足，請先往下購買代幣。</p>
          <p v-else-if="!isConnected" class="hero-hint">先連接 MetaMask，再開始你的鏈上大學生活。</p>
          <p v-else-if="!isCorrectNetwork" class="hero-hint">目前不是 Sepolia 測試網，請先切換網路。</p>
          <div class="ticker" aria-label="Game themes">
            <span>學分危機</span>
            <span>教授點名</span>
            <span>期末爆炸</span>
            <span>專題上岸</span>
          </div>
        </div>

        <div class="slot-stage" aria-label="Graduation simulator slot machine">
          <div class="machine-top">FINAL WEEK</div>
          <div class="slot-window">
            <div v-for="item in slotItems" :key="item" class="slot-reel">{{ item }}</div>
          </div>
          <div class="machine-light-row">
            <span v-for="n in 10" :key="n"></span>
          </div>
        </div>
      </section>
    </header>

    <section v-if="message" class="message" :class="messageType" role="status">
      {{ message }}
    </section>

    <section v-if="txStatus" class="tx-status" aria-live="polite">
      {{ txStatus }}
    </section>

    <section class="purchase-band">
      <div class="panel buy-panel">
        <div class="panel-heading">
          <p class="eyebrow">Token Shop</p>
          <h2>購買 NTUT Token</h2>
        </div>
        <p class="exchange-rate">0.001 Sepolia ETH = 10 NTUT = 遊玩 1 次</p>

        <div class="quick-buy" aria-label="Quick buy options">
          <button v-for="option in buyOptions" :key="option.amount" type="button" @click="selectBuyAmount(option.amount)">
            <span>{{ option.label }}</span>
            <strong>{{ option.amount }} ETH</strong>
          </button>
        </div>

        <label class="field-label" for="ethAmount">輸入 Sepolia ETH 數量</label>
        <div class="input-row">
          <input id="ethAmount" v-model="customEthAmount" inputmode="decimal" placeholder="0.001" />
          <button type="button" @click="buyTokens(customEthAmount)" :disabled="!canTransact || isLoading">
            購買 NTUT
          </button>
        </div>
      </div>

      <div class="panel mini-rule-panel">
        <p class="eyebrow">How It Works</p>
        <h2>合約決定結局，前端跑四年劇情</h2>
        <p>按下開始後，Sepolia 合約會抽出 S/A/B/C。前端再從事件資料庫抽主線，塞進大一到大四的 24 個節點。</p>
      </div>
    </section>

    <section class="wallet-bottom" aria-label="Wallet status">
      <div class="panel status-panel">
        <div class="panel-heading">
          <p class="eyebrow">Wallet Status</p>
          <h2>錢包狀態</h2>
        </div>

        <div class="status-grid">
          <div class="status-item">
            <span>目前網路</span>
            <strong :class="isCorrectNetwork ? 'ok-text' : 'danger-text'">{{ chainLabel }}</strong>
          </div>
          <div class="status-item">
            <span>錢包地址</span>
            <strong>{{ account || '尚未連接' }}</strong>
          </div>
          <div class="status-item">
            <span>Sepolia ETH 餘額</span>
            <strong>{{ displayEthBalance }} ETH</strong>
          </div>
          <div class="status-item">
            <span>NTUT 餘額</span>
            <strong>{{ displayNtutBalance }} NTUT</strong>
          </div>
        </div>

        <div v-if="isConnected && !isCorrectNetwork" class="network-warning">
          <span>請切換到 Sepolia 測試網</span>
          <button type="button" @click="switchToSepolia" :disabled="isLoading">切換到 Sepolia</button>
        </div>
      </div>

      <div class="panel notice-panel">
        <p>本 DApp 部署於 Sepolia 測試網，所有 ETH 與 NTUT Token 皆為測試用途，沒有實際金錢價值。</p>
        <p>每次購買與遊玩都需要支付少量 Sepolia gas fee。</p>
      </div>
    </section>

    <section class="panel metamask-panel">
      <div class="panel-heading">
        <p class="eyebrow">MetaMask Token</p>
        <h2>想在 MetaMask 裡看到 NTUT？</h2>
      </div>
      <p>請匯入以下代幣地址：</p>
      <code>{{ contractAddress }}</code>
      <button type="button" @click="addTokenToMetaMask">加入 NTUT 到 MetaMask</button>
    </section>

    <div v-if="storyModalOpen" class="modal-backdrop" role="dialog" aria-modal="true" aria-label="北科四年劇情模擬">
      <section class="story-modal" :class="resultClass">
        <button class="modal-close" type="button" @click="closeStoryModal" aria-label="Close story">X</button>
        <div class="story-header">
          <div>
            <p class="eyebrow">Four-Year Simulation</p>
            <h2>北科四年時間軸</h2>
          </div>
          <div class="story-rank" v-if="resultRank">{{ resultRank }}</div>
        </div>

        <div v-if="isSpinning" class="loading-strip story-loading" aria-live="polite">
          <div class="spinner-dot"></div>
          <p>{{ loadingText }}</p>
        </div>

        <div v-else-if="activeStory" class="story-body">
          <div class="story-progress">
            <span>{{ activeStory.label }}</span>
            <strong>{{ storyStep + 1 }} / {{ timeline.length }}</strong>
          </div>
          <div class="story-card" :class="`event-${activeStory.rank?.toLowerCase() || 'b'}`">
            <div class="event-meta">
              <span>{{ activeStory.type === 'mainline' ? '主線事件' : activeStory.category }}</span>
              <strong>{{ activeStory.rank || 'B' }}</strong>
            </div>
            <p>{{ activeStory.text }}</p>
          </div>
          <div class="story-controls">
            <button type="button" @click="prevStoryStep" :disabled="storyStep === 0">上一段</button>
            <button type="button" @click="nextStoryStep">
              {{ storyStep === timeline.length - 1 ? '看最終結局' : '下一段' }}
            </button>
          </div>
        </div>

        <div v-if="resultVisible" class="modal-result">
          <div class="rank-badge">{{ resultRank }}</div>
          <div>
            <p class="eyebrow">Final Ending</p>
            <h2>{{ resultContent.title }}</h2>
            <p>{{ resultContent.description }}</p>
            <strong>獎勵：{{ formattedPayout }} NTUT</strong>
          </div>
        </div>
      </section>
    </div>
  </main>
</template>

<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue';
import { ethers } from 'ethers';
import eventData from './event.json';

const contractAddress = '0x34d56E04F7fE36d4c04AE0b37ac6b80731De7e48';
const contractABI = [
  'function name() view returns (string)',
  'function symbol() view returns (string)',
  'function decimals() view returns (uint8)',
  'function balanceOf(address account) view returns (uint256)',
  'function buyTokens() payable',
  'function playGame()',
  'function PLAY_COST() view returns (uint256)',
  'function TOKENS_PER_ETH() view returns (uint256)',
  'function MIN_PURCHASE() view returns (uint256)',
  'event TokensPurchased(address indexed player, uint256 ethAmount, uint256 tokenAmount)',
  'event GamePlayed(address indexed player, string resultRank, uint256 payout)'
];

const sepoliaChainId = '0xaa36a7';
const TOTAL_STEPS = 24;

const account = ref('');
const chainId = ref('');
const ethBalance = ref('0');
const ntutBalance = ref('0');
const isConnected = ref(false);
const isCorrectNetwork = ref(false);
const isLoading = ref(false);
const txStatus = ref('');
const resultRank = ref('');
const payout = ref('0');
const resultVisible = ref(false);

const customEthAmount = ref('0.001');
const message = ref('');
const messageType = ref('info');
const isSpinning = ref(false);
const loadingIndex = ref(0);
const storyModalOpen = ref(false);
const timeline = ref([]);
const storyStep = ref(0);
let loadingTimer;
let provider;
let signer;
let contract;

const slotItems = ['學分', '教授', '專題', '畢業', '實習'];
const buyOptions = [
  { label: '買 1 次', amount: '0.001' },
  { label: '買 5 次', amount: '0.005' },
  { label: '買 10 次', amount: '0.01' }
];
const loadingTexts = [
  '正在排隊等加退選...',
  '教授正在改期末成績...',
  '專題組員正在消失...',
  '正在計算畢業門檻...',
  '正在尋找失蹤的學分...'
];

const resultMap = {
  S: { title: 'S 級結局：北科傳說', description: '你不只準時畢業，還拿到書卷獎、推甄上岸，教授甚至記得你的名字。' },
  A: { title: 'A 級結局：穩穩畢業', description: '雖然期末有點硬，但你成功活下來了。履歷有專題、有實習，人生暫時看起來沒壞掉。' },
  B: { title: 'B 級結局：低空飛過', description: '你以一種差點出事但又沒真的出事的姿勢通過了。教授放你一馬，學分驚險到手。' },
  C: { title: 'C 級結局：延畢警報', description: '專題爆炸、期末爆炸、加退選也爆炸。你開始懷疑自己是不是被學校選中當壓力測試樣本。' }
};

const displayEthBalance = computed(() => Number(ethBalance.value || '0').toFixed(4));
const displayNtutBalance = computed(() => Number(ntutBalance.value || '0').toFixed(2));
const formattedPayout = computed(() => Number(payout.value || '0').toFixed(2));
const hasEnoughNtut = computed(() => Number(ntutBalance.value || '0') >= 10);
const canTransact = computed(() => isConnected.value && isCorrectNetwork.value && !isLoading.value);
const canPlay = computed(() => canTransact.value && hasEnoughNtut.value);
const loadingText = computed(() => loadingTexts[loadingIndex.value]);
const chainLabel = computed(() => {
  if (!isConnected.value) return '尚未連接';
  return isCorrectNetwork.value ? 'Sepolia' : '錯誤網路';
});
const resultContent = computed(() => resultMap[resultRank.value] || resultMap.C);
const resultClass = computed(() => `rank-${String(resultRank.value || 'c').toLowerCase()}`);
const activeStory = computed(() => timeline.value[storyStep.value]);

function setMessage(text, type = 'info') {
  message.value = text;
  messageType.value = type;
}

function formatAddress(address) {
  if (!address) return '';
  return `${address.slice(0, 6)}...${address.slice(-4)}`;
}

function randomPick(list) {
  return list[Math.floor(Math.random() * list.length)];
}

function semesterLabel(index) {
  const years = ['大一', '大二', '大三', '大四'];
  const terms = ['上 開學', '上 期中', '上 期末', '下 開學', '下 期中', '下 期末'];
  return `${years[Math.floor(index / 6)]}${terms[index % 6]}`;
}

function buildTimeline(rank) {
  const mainlinePool = eventData.mainlines[rank] || eventData.mainlines.C;
  const selectedMainline = randomPick(mainlinePool);
  const nextTimeline = Array.from({ length: TOTAL_STEPS }, (_, index) => ({
    semesterIndex: index,
    label: semesterLabel(index),
    type: 'empty',
    text: ''
  }));

  for (const event of selectedMainline) {
    nextTimeline[event.semesterIndex] = {
      ...event,
      label: event.label || semesterLabel(event.semesterIndex),
      type: 'mainline',
      rank
    };
  }

  for (let index = 0; index < nextTimeline.length; index += 1) {
    if (nextTimeline[index].type === 'empty') {
      const independent = randomPick(eventData.independentEvents);
      nextTimeline[index] = {
        ...independent,
        semesterIndex: index,
        label: semesterLabel(index),
        type: 'independent'
      };
    }
  }

  timeline.value = nextTimeline;
  storyStep.value = 0;
}

function normalizeError(error) {
  if (error?.code === 4001 || error?.info?.error?.code === 4001) return '你取消了交易。';
  if (error?.code === 'ACTION_REJECTED') return '你取消了交易。';
  if (String(error?.message || '').toLowerCase().includes('insufficient')) return 'NTUT 不足，請先購買代幣。';
  return '交易失敗，請稍後再試。';
}

async function initContract() {
  provider = new ethers.BrowserProvider(window.ethereum);
  signer = await provider.getSigner();
  contract = new ethers.Contract(contractAddress, contractABI, signer);
}

async function connectWallet() {
  if (!window.ethereum) {
    setMessage('請先安裝 MetaMask 錢包。', 'error');
    return;
  }

  try {
    isLoading.value = true;
    txStatus.value = '等待 MetaMask 確認...';
    const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' });
    account.value = accounts[0] || '';
    isConnected.value = Boolean(account.value);
    await initContract();
    await checkNetwork();
    if (isCorrectNetwork.value) await loadBalances();
    txStatus.value = '';
    setMessage('錢包已連接。', 'success');
    listenForGamePlayed();
  } catch (error) {
    txStatus.value = '';
    setMessage(error?.code === 4001 ? '你取消了錢包連接。' : '錢包連接失敗，請稍後再試。', 'error');
  } finally {
    isLoading.value = false;
  }
}

async function checkNetwork() {
  if (!window.ethereum) return;
  const currentChainId = await window.ethereum.request({ method: 'eth_chainId' });
  chainId.value = currentChainId;
  isCorrectNetwork.value = currentChainId.toLowerCase() === sepoliaChainId;
  if (!isCorrectNetwork.value && isConnected.value) setMessage('目前不是 Sepolia 測試網，請切換網路。', 'error');
}

async function switchToSepolia() {
  if (!window.ethereum) {
    setMessage('請先安裝 MetaMask 錢包。', 'error');
    return;
  }

  try {
    isLoading.value = true;
    await window.ethereum.request({ method: 'wallet_switchEthereumChain', params: [{ chainId: sepoliaChainId }] });
    await checkNetwork();
    await initContract();
    await loadBalances();
    setMessage('已切換到 Sepolia 測試網。', 'success');
  } catch (error) {
    if (error?.code === 4902) {
      try {
        await window.ethereum.request({
          method: 'wallet_addEthereumChain',
          params: [{
            chainId: sepoliaChainId,
            chainName: 'Sepolia',
            nativeCurrency: { name: 'Sepolia ETH', symbol: 'ETH', decimals: 18 },
            rpcUrls: ['https://rpc.sepolia.org'],
            blockExplorerUrls: ['https://sepolia.etherscan.io']
          }]
        });
        await switchToSepolia();
      } catch {
        setMessage('無法新增 Sepolia 網路，請在 MetaMask 手動加入。', 'error');
      }
      return;
    }
    setMessage(error?.code === 4001 ? '你取消了網路切換。' : '目前不是 Sepolia 測試網，請切換網路。', 'error');
  } finally {
    isLoading.value = false;
  }
}

async function loadBalances() {
  if (!account.value || !isCorrectNetwork.value) return;
  if (!provider || !contract) await initContract();
  const eth = await provider.getBalance(account.value);
  const ntut = await contract.balanceOf(account.value);
  ethBalance.value = ethers.formatEther(eth);
  ntutBalance.value = ethers.formatUnits(ntut, 18);
}

function selectBuyAmount(amount) {
  customEthAmount.value = amount;
}

async function buyTokens(amount) {
  if (!canTransact.value) {
    setMessage(isConnected.value ? '目前不是 Sepolia 測試網，請切換網路。' : '請先連接 MetaMask 錢包。', 'error');
    return;
  }

  try {
    if (!amount || Number(amount) <= 0) {
      setMessage('請輸入有效的 Sepolia ETH 數量。', 'error');
      return;
    }
    isLoading.value = true;
    txStatus.value = '等待 MetaMask 確認...';
    const tx = await contract.buyTokens({ value: ethers.parseEther(amount) });
    txStatus.value = '交易送出中...';
    await tx.wait();
    txStatus.value = '購買成功！';
    setMessage('購買成功！NTUT 已進入你的鏈上人生背包。', 'success');
    await loadBalances();
  } catch (error) {
    txStatus.value = normalizeError(error).replace('交易失敗，請稍後再試。', '購買失敗，請重試。');
    setMessage(txStatus.value, 'error');
  } finally {
    isLoading.value = false;
  }
}

async function playGame() {
  if (!canTransact.value) {
    setMessage(isConnected.value ? '目前不是 Sepolia 測試網，請切換網路。' : '請先連接 MetaMask 錢包。', 'error');
    return;
  }
  if (!hasEnoughNtut.value) {
    setMessage('NTUT 不足，請先購買代幣。', 'error');
    return;
  }

  try {
    storyModalOpen.value = true;
    timeline.value = [];
    isLoading.value = true;
    resultVisible.value = false;
    startLoadingAnimation();
    txStatus.value = '等待 MetaMask 確認...';
    const tx = await contract.playGame();
    txStatus.value = '鏈上模擬中...';
    txStatus.value = '正在等待交易確認...';
    const receipt = await tx.wait();
    parseGamePlayedEvent(receipt);
    txStatus.value = '抽卡完成！';
    await loadBalances();
  } catch (error) {
    txStatus.value = normalizeError(error);
    setMessage(txStatus.value, 'error');
  } finally {
    stopLoadingAnimation();
    isLoading.value = false;
  }
}

function parseGamePlayedEvent(receipt) {
  for (const log of receipt.logs) {
    try {
      const parsed = contract.interface.parseLog(log);
      if (parsed?.name === 'GamePlayed') {
        const player = parsed.args.player;
        const parsedRank = parsed.args.resultRank;
        const parsedPayout = parsed.args.payout;
        if (player.toLowerCase() === account.value.toLowerCase()) {
          showResult(parsedRank, parsedPayout);
          return;
        }
      }
    } catch {
      continue;
    }
  }
}

function showResult(nextRank, nextPayout) {
  resultRank.value = nextRank;
  payout.value = ethers.formatUnits(nextPayout, 18);
  buildTimeline(nextRank);
  resultVisible.value = false;
  setMessage(`${nextRank} 級結局已抽出，開始播放四年人生。`, 'success');
}

function nextStoryStep() {
  if (storyStep.value < timeline.value.length - 1) {
    storyStep.value += 1;
    return;
  }
  resultVisible.value = true;
}

function prevStoryStep() {
  if (storyStep.value > 0) storyStep.value -= 1;
}

function closeStoryModal() {
  storyModalOpen.value = false;
}

function listenForGamePlayed() {
  if (!contract || !account.value) return;
  contract.removeAllListeners('GamePlayed');
  contract.on('GamePlayed', (player, eventRank, eventPayout) => {
    if (player.toLowerCase() !== account.value.toLowerCase()) return;
    if (!storyModalOpen.value) showResult(eventRank, eventPayout);
    loadBalances();
  });
}

function startLoadingAnimation() {
  isSpinning.value = true;
  loadingIndex.value = 0;
  clearInterval(loadingTimer);
  loadingTimer = setInterval(() => {
    loadingIndex.value = (loadingIndex.value + 1) % loadingTexts.length;
  }, 1300);
}

function stopLoadingAnimation() {
  isSpinning.value = false;
  clearInterval(loadingTimer);
}

async function addTokenToMetaMask() {
  if (!window.ethereum) {
    setMessage('請先安裝 MetaMask 錢包。', 'error');
    return;
  }

  try {
    await window.ethereum.request({
      method: 'wallet_watchAsset',
      params: { type: 'ERC20', options: { address: contractAddress, symbol: 'NTUT', decimals: 18 } }
    });
  } catch {
    setMessage('無法加入 NTUT 到 MetaMask。', 'error');
  }
}

function handleAccountsChanged(accounts) {
  account.value = accounts[0] || '';
  isConnected.value = Boolean(account.value);
  if (isConnected.value) {
    initContract().then(() => {
      checkNetwork().then(loadBalances);
      listenForGamePlayed();
    });
  }
}

function handleChainChanged(nextChainId) {
  chainId.value = nextChainId;
  isCorrectNetwork.value = nextChainId.toLowerCase() === sepoliaChainId;
  if (isCorrectNetwork.value && isConnected.value) initContract().then(loadBalances);
}

onMounted(async () => {
  if (!window.ethereum) {
    setMessage('請先安裝 MetaMask 錢包。', 'error');
    return;
  }

  window.ethereum.on?.('accountsChanged', handleAccountsChanged);
  window.ethereum.on?.('chainChanged', handleChainChanged);
  const accounts = await window.ethereum.request({ method: 'eth_accounts' });
  if (accounts.length) {
    account.value = accounts[0];
    isConnected.value = true;
    await initContract();
    await checkNetwork();
    if (isCorrectNetwork.value) await loadBalances();
    listenForGamePlayed();
  }
});

onBeforeUnmount(() => {
  stopLoadingAnimation();
  contract?.removeAllListeners('GamePlayed');
  window.ethereum?.removeListener?.('accountsChanged', handleAccountsChanged);
  window.ethereum?.removeListener?.('chainChanged', handleChainChanged);
});
</script>