<template>
  <div class="container">
    <div class="dashboard">
      <header>
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

        <div class="input-group step-up-box">
          <p class="step-label">途中から積立額を変更</p>
          <div class="input-group">
            <label>変更する年齢</label>
            <input type="number" v-model.number="stepAge" />
          </div>
          <div class="input-group">
            <label>新しい積立額 (万円)</label>
            <input type="number" v-model.number="monthlyInvestment2" />
          </div>
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

const startAge = ref(25);
const endAge = ref(65);
const initialAmount = ref(100);
const monthlyInvestment = ref(5);
const annualRate = ref(5);
const stepAge = ref(35);
const monthlyInvestment2 = ref(10);
const favorites = ref([]);

onMounted(() => {
  const saved = localStorage.getItem('compound_favorites');
  if (saved) {
    try {
      favorites.value = JSON.parse(saved);
    } catch (e) {
      favorites.value = [];
    }
  }
});

const saveFavorite = () => {
  const name = prompt('プラン名を入力', `プラン ${favorites.value.length + 1}`);
  if (!name) return;
  const newFavorite = {
    id: Date.now(),
    name: name,
    config: {
      startAge: startAge.value,
      endAge: endAge.value,
      initialAmount: initialAmount.value,
      monthlyInvestment: monthlyInvestment.value,
      annualRate: annualRate.value,
      stepAge: stepAge.value,
      monthlyInvestment2: monthlyInvestment2.value,
    },
  };
  favorites.value.push(newFavorite);
  localStorage.setItem('compound_favorites', JSON.stringify(favorites.value));
};

const applyFavorite = (fav) => {
  startAge.value = fav.config.startAge;
  endAge.value = fav.config.endAge;
  initialAmount.value = fav.config.initialAmount;
  monthlyInvestment.value = fav.config.monthlyInvestment;
  annualRate.value = fav.config.annualRate;
  stepAge.value = fav.config.stepAge;
  monthlyInvestment2.value = fav.config.monthlyInvestment2;
};

const deleteFavorite = (id) => {
  favorites.value = favorites.value.filter((f) => f.id !== id);
  localStorage.setItem('compound_favorites', JSON.stringify(favorites.value));
};

// --- 計算ロジック本体 ---
const simulationData = computed(() => {
  const records = [];
  let currentBalance = initialAmount.value;
  let totalPrincipal = initialAmount.value;
  const years = endAge.value - startAge.value;

  if (years < 0) return [];

  records.push({
    age: startAge.value,
    year: 0,
    total: Math.floor(currentBalance),
    principal: Math.floor(totalPrincipal),
  });

  for (let i = 1; i <= years; i++) {
    const currentAge = startAge.value + i;
    const monthly = currentAge >= stepAge.value ? monthlyInvestment2.value : monthlyInvestment.value;
    const yearlyInvestment = monthly * 12;
    const interest = currentBalance * (annualRate.value / 100);

    currentBalance += yearlyInvestment + interest;
    totalPrincipal += yearlyInvestment;

    records.push({
      age: currentAge,
      total: Math.floor(currentBalance),
      principal: Math.floor(totalPrincipal),
    });
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
  grid-area: title;
}

.input-panel {
  grid-area: input;
}

.result-panel {
  grid-area: result;
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
  display: grid;
  grid-template-columns: 320px 1fr;
  grid-template-areas:
    'title result'
    'input result';
  gap: 20px 40px;
  align-items: start;
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
    display: flex; /* gridよりflexの方がスマホでは制御しやすい */
    flex-direction: column;
    gap: 20px;
    align-items: center;
  }

  header {
    text-align: center; /* タイトルを中央寄せに */
    margin-bottom: 10px;
  }

  .input-panel {
    margin-bottom: 30px; /* 入力パネルの下に余白を作る */
    width: 100%;
  }

  .summary-card {
    padding: 20px 0;
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
}
</style>
