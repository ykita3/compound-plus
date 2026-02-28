<template>
  <div class="container">
    <div v-if="!isLoggedIn" class="login-screen">
      <div class="login-card">
        <h1>Compound Plus <span class="plus">+</span></h1>
        <p>シミュレーションを始めるにはログインしてください</p>
        <div id="google-login-btn"></div>
        <button @click="loginAsTestUser" class="btn-test-login">テストユーザーとして利用</button>
      </div>
    </div>
    <div v-else class="dashboard">
      <div class="left-column">
        <header>
          <div class="user-info">
            <img :src="user.picture" class="user-icon" v-if="user.picture" />
            <span>{{ user.name }} さん</span>
            <button @click="logout" class="btn-logout">ログアウト</button>
          </div>
          <h1>Compound Plus <span class="plus">+</span></h1>
          <p class="subtitle">資産運用のシミュレーション</p>
        </header>

        <section class="input-panel">
          <div class="input-group">
            <label>積立開始年齢</label>
            <input type="number" v-model.number="startAge" />
          </div>
          <div class="input-group">
            <label>積立終了年齢</label>
            <input type="number" v-model.number="endAge" />
          </div>
          <div class="input-group">
            <label>初期投資額 (万円)</label>
            <input type="number" v-model.number="initialAmount" />
          </div>
          <div class="input-group">
            <label>毎月の積立額 (万円)</label>
            <input type="number" v-model.number="monthlyInvestment" />
          </div>
          <div class="input-group">
            <label>想定利回り (%)</label>
            <input type="number" v-model.number="annualRate" step="0.1" />
          </div>

          <div class="step-up-section">
            <p class="step-label">積立額の変更設定</p>

            <div v-for="(step, index) in stepUpList" :key="index" class="step-item-box">
              <div class="input-group">
                <label>年齢</label>
                <input type="number" v-model.number="step.age" />
              </div>
              <div class="input-group">
                <label>積立額 (万円)</label>
                <input type="number" v-model.number="step.monthly" />
              </div>
              <button @click="removeStepUp(index)" class="btn-remove-step">削除</button>
            </div>

            <button @click="addStepUp" class="btn-add-step">+ 変更設定を追加</button>
          </div>

          <div class="favorite-section">
            <button @click="saveFavorite" class="btn-save">今の設定をお気に入り登録</button>
            <div v-if="favorites.length > 0" class="favorite-list">
              <div v-for="fav in favorites" :key="fav.id" class="favorite-item">
                <span @click="applyFavorite(fav)" class="fav-name">{{ fav.name }}</span>
                <button @click="deleteFavorite(fav.id)" class="btn-delete">×</button>
              </div>
            </div>
          </div>
        </section>
      </div>
      <section class="result-panel">
        <div class="summary-card">
          <h3>{{ endAge }}歳時点の予測</h3>
          <p class="total-amount">{{ finalBalance.toLocaleString() }}<span>万円</span></p>
          <div class="summary-details">
            <p>投資元本: {{ finalPrincipal.toLocaleString() }}万円</p>
            <p>運用収益: {{ (finalBalance - finalPrincipal).toLocaleString() }}万円</p>
          </div>
        </div>

        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>年齢</th>
                <th>資産総額</th>
                <th>(うち元本)</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="record in simulationData" :key="record.age">
                <td>{{ record.age }}歳</td>
                <td style="font-weight: bold">{{ record.total.toLocaleString() }}万円</td>
                <td style="color: #666">{{ record.principal.toLocaleString() }}万円</td>
              </tr>
            </tbody>
          </table>
        </div>
      </section>
    </div>
  </div>
</template>
<script setup>
import { ref, computed, onMounted } from 'vue';
import { jwtDecode } from 'jwt-decode';

// --- 基本設定 ---
const startAge = ref(25);
const endAge = ref(65);
const initialAmount = ref(100);
const monthlyInvestment = ref(5);
const annualRate = ref(5);

const stepUpList = ref([]);
const favorites = ref([]);
const isLoggedIn = ref(false);
const user = ref(null);

// --- ユーザー固有の保存キーを作る関数 ---
const getStorageKey = () => {
  if (!user.value) return 'compound_favorites_guest';
  // Googleユーザーならsub(固有ID)かemail、ゲストなら固定文字
  const userId = user.value.sub || user.value.email || 'guest';
  return `compound_favorites_${userId}`;
};

// --- お気に入りを読み込む ---
const loadFavorites = () => {
  const saved = localStorage.getItem(getStorageKey());
  if (saved) {
    try {
      favorites.value = JSON.parse(saved);
    } catch (e) {
      favorites.value = [];
    }
  } else {
    favorites.value = [];
  }
};

// --- ログイン処理 ---
const handleCredentialResponse = (response) => {
  const userData = jwtDecode(response.credential);
  user.value = userData;
  isLoggedIn.value = true;
  localStorage.setItem('user_token', response.credential);
  loadFavorites(); // ログイン直後にそのユーザーのデータを読み込む
};

const loginAsTestUser = () => {
  user.value = { name: 'ゲストユーザー', picture: 'https://cdn-icons-png.flaticon.com/512/217/217853.png' };
  isLoggedIn.value = true;
  localStorage.setItem('user_token', 'guest-mode');
  loadFavorites(); // ゲスト用のデータを読み込む
};

const logout = () => {
  localStorage.removeItem('user_token');
  location.reload();
};

const renderGoogleButton = () => {
  setTimeout(() => {
    const btnRoot = document.getElementById('google-login-btn');
    if (btnRoot && window.google) {
      window.google.accounts.id.renderButton(btnRoot, { theme: 'outline', size: 'large', width: '250' });
    }
  }, 100);
};

onMounted(() => {
  // ログイン状態の復元
  const token = localStorage.getItem('user_token');
  if (token) {
    if (token === 'guest-mode') {
      user.value = { name: 'ゲストユーザー', picture: 'https://cdn-icons-png.flaticon.com/512/217/217853.png' };
      isLoggedIn.value = true;
    } else {
      try {
        user.value = jwtDecode(token);
        isLoggedIn.value = true;
      } catch (e) {
        localStorage.removeItem('user_token');
      }
    }
    loadFavorites(); // 復元できたらデータをロード
  }

  // Googleボタン初期化
  if (window.google) {
    window.google.accounts.id.initialize({
      client_id: '61911223747-da5qv64n8tvvnfuikkbakh6g6ugu2fpm.apps.googleusercontent.com',
      callback: handleCredentialResponse,
    });
    if (!isLoggedIn.value) renderGoogleButton();
  }
});

// --- 操作系 ---
const addStepUp = () => {
  stepUpList.value.push({ age: startAge.value + 10, monthly: monthlyInvestment.value + 5 });
};

const removeStepUp = (index) => {
  stepUpList.value.splice(index, 1);
};

const saveFavorite = () => {
  const name = prompt('プラン名を入力', `プラン ${favorites.value.length + 1}`);
  if (!name) return;
  const newFavorite = {
    id: Date.now(),
    name,
    config: {
      startAge: startAge.value,
      endAge: endAge.value,
      initialAmount: initialAmount.value,
      monthlyInvestment: monthlyInvestment.value,
      annualRate: annualRate.value,
      stepUpList: JSON.parse(JSON.stringify(stepUpList.value)),
    },
  };
  favorites.value.push(newFavorite);
  localStorage.setItem(getStorageKey(), JSON.stringify(favorites.value));
};

const applyFavorite = (fav) => {
  startAge.value = fav.config.startAge;
  endAge.value = fav.config.endAge;
  initialAmount.value = fav.config.initialAmount;
  monthlyInvestment.value = fav.config.monthlyInvestment;
  annualRate.value = fav.config.annualRate;
  stepUpList.value = fav.config.stepUpList ? JSON.parse(JSON.stringify(fav.config.stepUpList)) : [];
};

const deleteFavorite = (id) => {
  favorites.value = favorites.value.filter((f) => f.id !== id);
  localStorage.setItem(getStorageKey(), JSON.stringify(favorites.value));
};

// --- 計算ロジック ---
const simulationData = computed(() => {
  const records = [];
  let currentBalance = initialAmount.value || 0;
  let totalPrincipal = initialAmount.value || 0;
  const years = (endAge.value || 0) - (startAge.value || 0);

  if (years <= 0) return [{ age: startAge.value, total: currentBalance, principal: totalPrincipal }];

  records.push({ age: startAge.value, total: Math.floor(currentBalance), principal: Math.floor(totalPrincipal) });

  const sortedSteps = [...stepUpList.value].sort((a, b) => a.age - b.age);

  for (let i = 1; i <= years; i++) {
    const currentAge = startAge.value + i;
    let monthly = monthlyInvestment.value || 0;
    for (const step of sortedSteps) {
      if (step.age > 0 && step.monthly > 0 && currentAge >= step.age) {
        monthly = step.monthly;
      }
    }
    const yearlyInvestment = monthly * 12;
    const interest = currentBalance * ((annualRate.value || 0) / 100);
    currentBalance += yearlyInvestment + interest;
    totalPrincipal += yearlyInvestment;
    records.push({ age: currentAge, total: Math.floor(currentBalance), principal: Math.floor(totalPrincipal) });
  }
  return records;
});

const finalBalance = computed(() =>
  simulationData.value.length > 0 ? simulationData.value[simulationData.value.length - 1].total : 0,
);
const finalPrincipal = computed(() =>
  simulationData.value.length > 0 ? simulationData.value[simulationData.value.length - 1].principal : 0,
);
</script>

<style scoped>
header {
  padding-bottom: 20px;
}

.input-panel {
  grid-area: input;
}

.result-panel {
  flex: 1;
  min-width: 0; /* はみ出し防止 */
}

h1 {
  margin: 0;
}
.subtitle {
  margin: 0 0 10px;
}
.container {
  max-width: 1000px;
  margin: 0 auto;
  padding: 40px 20px;
  font-family: sans-serif;
  color: #333;
}
.plus {
  color: #42b983;
}

.dashboard {
  display: flex;
  gap: 40px;
  align-items: flex-start;
}
.left-column {
  flex: 0 0 auto; /* 幅を自動（中身次第）にする魔法の言葉 */
  display: flex;
  flex-direction: column;
  gap: 0; /* タイトルと入力欄の間の隙間をゼロに */
}
.input-panel {
  background: #f8f9fa;
  padding: 20px;
  border-radius: 12px;
}
.input-group {
  margin-bottom: 15px;
}
.input-group label {
  display: block;
  font-weight: bold;
  margin-bottom: 5px;
  font-size: 13px;
  color: #555;
}
.input-group input {
  width: 100%;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 6px;
  background: white;
  color: #000;
  box-sizing: border-box;
}
.step-up-box {
  margin-top: 20px;
  padding-top: 15px;
  border-top: 1px dashed #ccc;
}
.step-label {
  color: #42b983;
  font-weight: bold;
  font-size: 14px;
  margin-bottom: 10px;
}
.summary-card {
  background: #005344;
  color: white;
  padding: 25px;
  border-radius: 12px;
  margin-bottom: 25px;
  text-align: center;
}
.total-amount {
  font-size: 3rem;
  font-weight: bold;
  margin: 5px 0;
}
.summary-details {
  display: flex;
  justify-content: space-around;
  font-size: 0.9rem;
  opacity: 0.9;
}
.table-container {
  max-height: 650px;
  overflow-y: auto;
  border: 1px solid #eee;
  border-radius: 8px;
}
table {
  width: 100%;
  border-collapse: collapse;
  background: white;
}
th,
td {
  padding: 12px;
  border-bottom: 1px solid #eee;
  text-align: left;
}
th {
  background: #f4f4f4;
  position: sticky;
  top: 0;
}
.favorite-section {
  margin-top: 25px;
  border-top: 1px solid #ddd;
  padding-top: 15px;
}
.btn-save {
  width: 100%;
  padding: 12px;
  background: #42b983;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
}
.favorite-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: white;
  padding: 10px;
  margin-top: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}
.fav-name {
  cursor: pointer;
  flex-grow: 1;
  font-size: 14px;
}
.btn-delete {
  background: none;
  border: none;
  color: #999;
  cursor: pointer;
  font-size: 18px;
}
.favorite-list {
  padding-top: 8px;
}

/* --- ログイン画面のスタイル --- */
.login-screen {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 80vh;
}

.login-card {
  background: white;
  padding: 40px;
  border-radius: 16px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
  text-align: center;
  max-width: 400px;
  width: 100%;
}

.login-card h1 {
  margin-bottom: 10px;
}

.login-card p {
  color: #666;
  margin-bottom: 30px;
}

#google-login-btn {
  display: flex;
  justify-content: center;
}

/* --- ユーザー情報のスタイル --- */
.user-info {
  display: flex;
  align-items: center;
  justify-content: flex-start;
  gap: 10px;
  margin-bottom: 10px;
  font-size: 14px;
}

.user-icon {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  border: 1px solid #ddd;
}

.btn-logout {
  background: #eee;
  border: none;
  padding: 5px 10px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
  margin-left: auto;
}

.btn-logout:hover {
  background: #ddd;
}

.divider {
  margin: 20px 0;
  color: #999;
  font-size: 12px;
  position: relative;
}
.btn-test-login {
  width: 250px;
  padding: 12px;
  background: white;
  color: #666;
  border: 1px solid #ddd;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
  transition: gray 0.2s;
  margin-top: 10px;
}
.btn-test-login:hover {
  background: #f5f5f5;
}
.step-item-box {
  background: #fff;
  border: 1px solid #eee;
  padding: 15px;
  border-radius: 8px;
  margin-bottom: 10px;
  position: relative;
}
.btn-remove-step {
  background: #ffeded;
  color: #d32f2f;
  border: none;
  padding: 5px 10px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
  width: 100%;
}
.btn-add-step {
  width: 100%;
  padding: 10px;
  background: none;
  border: 2px dashed #42b983;
  color: #42b983;
  border-radius: 8px;
  cursor: pointer;
  font-weight: bold;
  margin-top: 10px;
}
.btn-add-step:hover {
  background: #f0fff9;
}
/* --- スマホ対応（画面幅が 768px 以下の時） --- */
@media (max-width: 768px) {
  .container {
    /* 100%から左右のパディング分を引く設定にする */
    width: 100%;
    max-width: 100vw; /* 画面幅を超えないように強制 */
    padding: 15px;
    box-sizing: border-box; /* 枠線や余白を幅に含める */
    overflow-x: hidden; /* 万が一の中身のはみ出しを隠す */
  }

  .dashboard {
    display: block;
  }
  h1 {
    margin: 22px;
    margin: 0 0 10px 0;
  }
  header {
    text-align: center; /* タイトルを中央寄せに */
    margin-bottom: 10px;
  }

  .input-panel {
    margin-bottom: 30px; /* 入力パネルの下に余白を作る */
    width: 100%;
    padding: 15px;
    box-sizing: border-box;
    border-radius: 10px;
  }

  .summary-card {
    padding: 10px 0;
    align-items: center;
    width: 100%;
  }

  .total-amount {
    font-size: 2.2rem; /* スマホでは金額の文字を少し小さく */
  }

  .table-container {
    max-height: none; /* スマホではテーブルを全部見せる（または固定高さを調整） */
    overflow-x: auto; /* 横幅が狭い時にテーブルを横スクロール可能に */
  }

  /* 入力しやすさのために、スマホではフォントを16px以上に（自動ズーム防止） */
  .input-group input {
    font-size: 16px;
  }

  .result-panel {
    width: 100%;
  }
  .input-panel {
    margin-bottom: 20px;
    padding: 15px; /* スマホではパディングを少し詰めるとスッキリするよ */
  }
  .user-info {
    justify-content: center;
    margin-bottom: 20px;
  }
  .login-card {
    box-shadow: none;
  }
  .summary-details {
    display: block;
    padding-bottom: 20px;
  }
  .summary-details p {
    margin: 2px 0 0 0;
  }
}
</style>
