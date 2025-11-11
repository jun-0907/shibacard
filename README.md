# shibacard
カードゲーム

<!DOCTYPE html>
<html lang="ja">
<head>
<script src="script.js" defer></script>
  <meta charset="UTF-8">
  <title>カードバトルゲーム</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <div id="card-selection">
    <h2>カードを5枚選んでください</h2>
    <div class="card-pool">
      <!-- プレイヤーカード10枚（例） -->
      <div class="card" data-id="1" data-hp="100" data-atk="30" data-def="10" data-heal="0" data-skill="normal">カード1<br>♡:100 ⚔:30 🛡:10</div>
      <div class="card" data-id="2" data-hp="80" data-atk="40" data-def="5" data-heal="10" data-skill="heal">カード2<br>♡:80 ⚔:40 🛡:5</div>
      <div class="card" data-id="3" data-hp="120" data-atk="20" data-def="20" data-heal="0" data-skill="buff">カード3<br>♡:120 ⚔:20 🛡:20</div>
      <div class="card" data-id="4" data-hp="90" data-atk="35" data-def="15" data-heal="5" data-skill="heal">カード4<br>♡:90 ⚔:35 🛡:15</div>
      <div class="card" data-id="5" data-hp="110" data-atk="25" data-def="10" data-heal="0" data-skill="aoe">カード5<br>♡:110 ⚔:25 🛡:10</div>
      <div class="card" data-id="6" data-hp="70" data-atk="45" data-def="5" data-heal="0" data-skill="normal">カード6<br>♡:70 ⚔:45 🛡:5</div>
      <div class="card" data-id="7" data-hp="95" data-atk="30" data-def="10" data-heal="10" data-skill="heal">カード7<br>♡:95 ⚔:30 🛡:10</div>
      <div class="card" data-id="8" data-hp="85" data-atk="40" data-def="10" data-heal="0" data-skill="buff">カード8<br>♡:85 ⚔:40 🛡:10</div>
      <div class="card" data-id="9" data-hp="100" data-atk="35" data-def="10" data-heal="0" data-skill="debuff">カード9<br>♡:100 ⚔:35 🛡:10</div>
      <div class="card" data-id="10" data-hp="75" data-atk="50" data-def="5" data-heal="0" data-skill="aoe">カード10<br>♡:75 ⚔50 🛡:5</div>
    </div>
    <button id="start-battle" disabled>バトル開始</button>
  </div>

  <div id="battle-field" style="display:none;">
    <h2>バトルフィールド</h2>
    <p>ターン: <span id="turn-display">プレイヤー</span></p>
    <p>エネルギー: <span id="energy-count">3</span></p>

    <h3>プレイヤーのカード</h3>
    <div id="player-cards" class="card-row"></div>

    <h3>敵のカード</h3>
    <div id="enemy-cards" class="card-row"></div>

    <div class="actions">
      <button id="attack-button">攻撃する</button>
      <button id="skill-button">スキル発動</button>
      <button id="end-turn">ターン終了</button>
    </div>
  </div>

</body>
</html>

:root{
  --bg: #f8f8f8;
  --card-bg: #ffffff;
  --card-border: #aaa;
  --primary: #2196f3;
  --muted: #777;
  --danger: #e53935;
  --shadow: rgba(0,0,0,0.06);
}

html,body{
  height: 100%;
  margin: 0;
  padding: 0;
  font-family: system-ui, "Hiragino Kaku Gothic ProN", "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
  background-color: var(--bg);
  color: #111;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

body{
  padding: 20px;
  box-sizing: border-box;
}

/* レイアウト */
.card-pool, .card-row{
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  margin: 10px 0;
  align-items: flex-start;
}

/* カード */
.card{
  width: 120px;
  min-height: 110px;
  box-sizing: border-box;
  background-color: var(--card-bg);
  border: 2px solid var(--card-border);
  border-radius: 8px;
  padding: 8px;
  cursor: pointer;
  white-space: pre-line;
  font-size: 14px;
  line-height: 1.35;
  box-shadow: 0 1px 3px var(--shadow);
  transition: transform .08s ease, box-shadow .12s ease, border-color .12s ease, background-color .12s ease;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  user-select: none;
  outline: none;
}

/* ホバー / フォーカス（キーボードで選べるように） */
.card:hover{
  transform: translateY(-3px);
  box-shadow: 0 6px 14px rgba(0,0,0,0.08);
}
.card:focus{
  border-color: var(--primary);
  box-shadow: 0 8px 18px rgba(33,150,243,0.12);
}

/* 選択（デッキ構築画面） */
.card.selected{
  border-color: var(--primary);
  background-color: #e8f6ff;
}

/* 攻撃者 / ターゲット選択の視覚差分 */
.card.attacker-selected{
  border-color: #ff9800; /* 橙：攻撃者 */
  box-shadow: 0 6px 14px rgba(255,152,0,0.10);
  background: linear-gradient(180deg, #fff7ef, #fffdf9);
}
.card.target-selected{
  border-color: #e53935; /* 赤：ターゲット */
  box-shadow: 0 6px 14px rgba(229,57,53,0.10);
  background: linear-gradient(180deg, #fff5f5, #fffdfd);
}

/* 戦闘不能 / 無効化 */
.card.disabled{
  opacity: 0.45;
  pointer-events: none;
  filter: grayscale(20%);
}

/* ボタン */
button{
  padding: 10px 18px;
  margin: 10px 8px 10px 0;
  font-size: 15px;
  border: none;
  border-radius: 8px;
  background-color: var(--primary);
  color: #fff;
  cursor: pointer;
  box-shadow: 0 2px 6px rgba(0,0,0,0.06);
  transition: background-color .12s ease, transform .08s ease, opacity .12s ease;
}
button:hover{ transform: translateY(-2px); }
button:active{ transform: translateY(0); }
button:focus{ outline: 3px solid rgba(33,150,243,0.18); }

/* 無効ボタンの見た目 */
button:disabled{
  background-color: #ddd;
  color: #888;
  cursor: default;
  opacity: 0.9;
  transform: none;
  box-shadow: none;
}

/* 開始ボタンの disabled は見た目で分かりやすく */
#start-battle:disabled{
  border: 2px dashed #ccc;
  background: linear-gradient(180deg,#fafafa,#f4f4f4);
  color: #999;
}

/* テキストの折返しと小さいビューでの見やすさ */
.card .meta{
  font-size: 12px;
  color: var(--muted);
  margin-top: 6px;
}

/* レスポンシブ：狭い画面ではカード幅を縮める */
@media (max-width: 480px){
  .card{
    width: calc(50% - 12px);
    min-height: 100px;
    font-size: 13px;
  }
  button{
    padding: 8px 12px;
    font-size: 14px;
  }
}

/* 大画面でカードを少し広げる */
@media (min-width: 900px){
  .card{
    width: 140px;
    min-height: 120px;
    font-size: 15px;
  }
}

document.addEventListener('DOMContentLoaded', () => {
  let selectedCards = [];
  let energy = 3;
  let isPlayerTurn = true;
  let selectedAttacker = null;
  let selectedTarget = null;

  const startButton = document.getElementById('start-battle');
  const attackButton = document.getElementById('attack-button');
  const skillButton = document.getElementById('skill-button');
  const endTurnButton = document.getElementById('end-turn');

  // 初期：カードプールの表示を詳細化して選びやすくする
  document.querySelectorAll('.card-pool .card').forEach(card => {
    Object.keys(card.dataset).forEach(k => {
      card.dataset[k] = String(card.dataset[k]);
    });
    updateCardDisplay(card);
    card.addEventListener('click', () => {
      const id = String(card.dataset.id);
      if (selectedCards.includes(id)) {
        selectedCards = selectedCards.filter(c => c !== id);
        card.classList.remove('selected');
      } else if (selectedCards.length < 5) {
        selectedCards.push(id);
        card.classList.add('selected');
      }
      updateSelectionUI();
    });
  });

  function updateSelectionUI() {
    startButton.disabled = selectedCards.length !== 5;
  }
  updateSelectionUI();

  startButton.addEventListener('click', () => {
    document.getElementById('card-selection').style.display = 'none';
    document.getElementById('battle-field').style.display = 'block';

    const playerCardData = getSelectedCardData();
    renderCards('player-cards', playerCardData);
    renderCards('enemy-cards', generateEnemyCards());
    // ターン開始時はプレイヤーのスキル使用フラグをリセット
    resetSkillUsage('player');
    updateBattleUI();
  });

  function generateEnemyCards() {
    const enemy = [];
    for (let i = 1; i <= 5; i++) {
      const hp = 80 + Math.floor(Math.random() * 40);
      enemy.push({
        id: `敵${i}`,
        hp: hp,
        maxHp: hp,
        atk: 20 + Math.floor(Math.random() * 30),
        def: 5 + Math.floor(Math.random() * 15),
        heal: 0,
        skill: 'normal'
      });
    }
    return enemy;
  }

  function renderCards(containerId, cardData) {
    const container = document.getElementById(containerId);
    container.innerHTML = '';
    cardData.forEach(data => {
      const card = document.createElement('div');
      card.className = 'card';
      Object.entries(data).forEach(([key, value]) => {
        card.dataset[key] = String(value);
      });
      // スキル使用済みフラグの初期化
      delete card.dataset._skillUsed;
      updateCardDisplay(card);
      card.addEventListener('click', () => handleCardClick(card, containerId));
      container.appendChild(card);
    });
  }

  function updateCardDisplay(card) {
    const id = card.dataset.id ?? '';
    const hp = parseInt(card.dataset.hp ?? '0', 10);
    const maxHp = parseInt(card.dataset.maxHp ?? card.dataset.hp ?? '0', 10);
    const atk = card.dataset.atk ?? '';
    const def = card.dataset.def ?? '';
    const heal = card.dataset.heal ?? '';
    const skill = card.dataset.skill ?? '';

    const healText = parseInt(heal, 10) > 0 ? `💖 回復: ${heal}` : '';
    const usedText = card.dataset._skillUsed === 'true' ? '🔒 スキル使用済み' : '';
    card.innerText =
      `🆔 ${id}
🧠 体力: ${hp}/${maxHp}
🗡 攻撃: ${atk}
🛡 防御: ${def}
🔮 スキル: ${skill} ${usedText}
${healText}`.trim();

    if (hp <= 0) {
      if (!card.classList.contains('disabled')) card.classList.add('disabled');
      if (!card.innerText.includes('🛑 戦闘不能')) card.innerText += '\n🛑 戦闘不能';
    } else {
      card.classList.remove('disabled');
    }
  }

  function handleCardClick(card, side) {
    if (card.classList.contains('disabled')) return;

    if (side === 'player-cards') {
      selectedAttacker = card;
    } else {
      selectedTarget = card;
    }

    document.querySelectorAll('.card').forEach(c => {
      c.classList.remove('attacker-selected', 'target-selected');
    });
    if (selectedAttacker && !selectedAttacker.classList.contains('disabled')) selectedAttacker.classList.add('attacker-selected');
    if (selectedTarget && !selectedTarget.classList.contains('disabled')) selectedTarget.classList.add('target-selected');

    updateBattleUI();
  }

  attackButton.addEventListener('click', () => {
    if (!isPlayerTurn || energy < 1 || !selectedAttacker) return;
    if (selectedAttacker.classList.contains('disabled')) {
      selectedAttacker = null;
      updateBattleUI();
      return;
    }

    const atk = parseInt(selectedAttacker.dataset.atk, 10) || 0;
    const skill = selectedAttacker.dataset.skill || 'normal';

    if (skill === 'aoe') {
      const targets = Array.from(document.querySelectorAll('#enemy-cards .card:not(.disabled)'));
      targets.forEach(target => applyDamage(target, atk));
    } else {
      if (!selectedTarget) return;
      if (selectedTarget.classList.contains('disabled')) {
        selectedTarget = null;
        updateBattleUI();
        return;
      }
      applyDamage(selectedTarget, atk);
    }

    energy -= 1;
    sanitizeSelectedRefs();
    updateBattleUI();
    checkGameEnd();
  });

  // スキル使用（プレイヤー）: 使用時に _skillUsed = 'true' を付与
  skillButton.addEventListener('click', () => {
    if (!isPlayerTurn || energy < 1 || !selectedAttacker) return;
    if (selectedAttacker.classList.contains('disabled')) {
      selectedAttacker = null;
      updateBattleUI();
      return;
    }
    // 既にそのカードがスキルを使っていれば実行しない
    if (selectedAttacker.dataset._skillUsed === 'true') return;

    const skill = selectedAttacker.dataset.skill || 'normal';

    if (skill === 'heal') {
      const healVal = parseInt(selectedAttacker.dataset.heal, 10) || 0;
      if (healVal > 0) {
        const hp = parseInt(selectedAttacker.dataset.hp, 10) || 0;
        const maxHp = parseInt(selectedAttacker.dataset.maxHp, 10) || hp;
        const newHp = Math.min(maxHp, hp + healVal);
        selectedAttacker.dataset.hp = String(newHp);
        updateCardDisplay(selectedAttacker);
      }
    } else if (skill === 'buff') {
      const curAtk = parseInt(selectedAttacker.dataset.atk, 10) || 0;
      selectedAttacker.dataset.atk = String(curAtk + 10);
      selectedAttacker.dataset._buffed = 'true';
    } else if (skill === 'debuff') {
      if (!selectedTarget || selectedTarget.classList.contains('disabled')) return;
      const curDef = parseInt(selectedTarget.dataset.def, 10) || 0;
      selectedTarget.dataset.def = String(Math.max(0, curDef - 5));
      updateCardDisplay(selectedTarget);
      selectedTarget.dataset._debuffed = 'true';
    } else if (skill === 'aoe') {
      const atk = parseInt(selectedAttacker.dataset.atk, 10) || 0;
      const targets = Array.from(document.querySelectorAll('#enemy-cards .card:not(.disabled)'));
      targets.forEach(target => applyDamage(target, atk));
    }

    // スキル使用フラグを立てる（このターンは二度使えない）
    selectedAttacker.dataset._skillUsed = 'true';
    // 表示更新
    updateCardDisplay(selectedAttacker);

    energy -= 1;
    sanitizeSelectedRefs();
    updateBattleUI();
    checkGameEnd();
  });

  function applyDamage(target, atk) {
    const def = parseInt(target.dataset.def ?? '0', 10);
    const damage = Math.max(0, atk - def);
    let hp = parseInt(target.dataset.hp ?? '0', 10);
    hp -= damage;
    target.dataset.hp = String(hp);
    updateCardDisplay(target);

    if (hp <= 0) {
      if (selectedAttacker === target) selectedAttacker = null;
      if (selectedTarget === target) selectedTarget = null;
    }
  }

  function updateBattleUI() {
    document.getElementById('energy-count').innerText = String(energy);
    document.getElementById('turn-display').innerText = isPlayerTurn ? 'プレイヤー' : '敵';

    attackButton.disabled = !(isPlayerTurn && energy >= 1 && selectedAttacker && !selectedAttacker.classList.contains('disabled') && (
      selectedAttacker.dataset.skill === 'aoe' || (selectedTarget && !selectedTarget.classList.contains('disabled'))
    ));

    // ここで既にそのカードがスキルを使っていたら無効化
    skillButton.disabled = !(isPlayerTurn && energy >= 1 && selectedAttacker && !selectedAttacker.classList.contains('disabled') && selectedAttacker.dataset.skill && selectedAttacker.dataset._skillUsed !== 'true');

    endTurnButton.disabled = false;

    if (selectedAttacker && selectedAttacker.classList.contains('disabled')) selectedAttacker = null;
    if (selectedTarget && selectedTarget.classList.contains('disabled')) selectedTarget = null;

    document.querySelectorAll('.card').forEach(c => {
      c.classList.remove('attacker-selected', 'target-selected');
    });
    if (selectedAttacker) selectedAttacker.classList.add('attacker-selected');
    if (selectedTarget) selectedTarget.classList.add('target-selected');
  }

  endTurnButton.addEventListener('click', () => {
    clearTurnTemporaryEffects();

    // ターン切り替え前に相手側のスキル使用フラグをクリアしておく（相手ターン開始時に相手のカードがスキルを使えるようにする）
    isPlayerTurn = !isPlayerTurn;
    energy = 3;
    selectedAttacker = null;
    selectedTarget = null;

    // 新しく行動する側のカードのスキル使用フラグをリセット
    resetSkillUsage(isPlayerTurn ? 'player' : 'enemy');

    updateBattleUI();

    if (!isPlayerTurn) {
      setTimeout(() => {
        enemyTurn();
        clearTurnTemporaryEffects();
        // 敵行動後にプレイヤーのターン開始準備
        isPlayerTurn = true;
        energy = 3;
        sanitizeSelectedRefs();
        // プレイヤー側のスキル使用フラグをリセット（ターン開始）
        resetSkillUsage('player');
        updateBattleUI();
        checkGameEnd();
      }, 700);
    }
  });

  function enemyTurn() {
    const enemies = Array.from(document.querySelectorAll('#enemy-cards .card:not(.disabled)'));
    const players = Array.from(document.querySelectorAll('#player-cards .card:not(.disabled)'));
    if (enemies.length === 0 || players.length === 0) return;

    // 敵はそれぞれ1回だけ行動（攻撃 or スキル）
    enemies.forEach(enemy => {
      // その敵が既にスキルを使っていたら通常攻撃のみ（skill 使用は一回のみ）
      const atk = parseInt(enemy.dataset.atk ?? '0', 10);
      const skill = enemy.dataset.skill ?? 'normal';

      if (skill !== 'normal' && enemy.dataset._skillUsed !== 'true') {
        // 簡易AI: 低HPならheal、aoeなら使う、その他は確率で使う
        const hp = parseInt(enemy.dataset.hp ?? '0', 10);
        if (skill === 'heal' && hp < (parseInt(enemy.dataset.maxHp ?? hp, 10) * 0.6)) {
          // heal
          const healVal = parseInt(enemy.dataset.heal, 10) || 0;
          enemy.dataset.hp = String(Math.min(parseInt(enemy.dataset.maxHp ?? enemy.dataset.hp, 10), hp + healVal));
          enemy.dataset._skillUsed = 'true';
          updateCardDisplay(enemy);
          return;
        }
        if (skill === 'aoe') {
          players.forEach(player => applyDamage(player, atk));
          enemy.dataset._skillUsed = 'true';
          return;
        }
        // それ以外は 30% の確率でスキルを使う（デバフやバフ等）
        if (Math.random() < 0.3) {
          if (skill === 'buff') {
            const curAtk = parseInt(enemy.dataset.atk, 10) || 0;
            enemy.dataset.atk = String(curAtk + 10);
            enemy.dataset._buffed = 'true';
            enemy.dataset._skillUsed = 'true';
            updateCardDisplay(enemy);
            return;
          }
          if (skill === 'debuff') {
            const target = players[Math.floor(Math.random() * players.length)];
            const curDef = parseInt(target.dataset.def, 10) || 0;
            target.dataset.def = String(Math.max(0, curDef - 5));
            target.dataset._debuffed = 'true';
            updateCardDisplay(target);
            enemy.dataset._skillUsed = 'true';
            return;
          }
        }
      }

      // スキルを使わなかった場合またはスキル使用済みなら通常攻撃
      const target = players[Math.floor(Math.random() * players.length)];
      applyDamage(target, atk);
    });
  }

  function getSelectedCardData() {
    const allCards = Array.from(document.querySelectorAll('.card-pool .card'));
    return selectedCards.map(id => {
      const card = allCards.find(c => String(c.dataset.id) === String(id));
      const hp = parseInt(card.dataset.hp ?? '0', 10);
      return {
        id: card.dataset.id,
        hp: hp,
        maxHp: hp,
        atk: parseInt(card.dataset.atk ?? '0', 10),
        def: parseInt(card.dataset.def ?? '0', 10),
        heal: parseInt(card.dataset.heal ?? '0', 10),
        skill: card.dataset.skill
      };
    });
  }

  function clearTurnTemporaryEffects() {
    document.querySelectorAll('.card').forEach(card => {
      if (card.dataset._buffed === 'true') {
        const atk = parseInt(card.dataset.atk ?? '0', 10);
        card.dataset.atk = String(Math.max(0, atk - 10));
        delete card.dataset._buffed;
        updateCardDisplay(card);
      }
      if (card.dataset._debuffed === 'true') {
        delete card.dataset._debuffed;
      }
    });
  }

  function sanitizeSelectedRefs() {
    if (selectedAttacker && selectedAttacker.classList.contains('disabled')) selectedAttacker = null;
    if (selectedTarget && selectedTarget.classList.contains('disabled')) selectedTarget = null;
  }

  // 勝敗判定：自分 or 敵のカードが全滅したらゲーム終了して開始画面へ戻す
  function checkGameEnd() {
    const playerAlive = document.querySelectorAll('#player-cards .card:not(.disabled)').length;
    const enemyAlive = document.querySelectorAll('#enemy-cards .card:not(.disabled)').length;

    if (playerAlive === 0 || enemyAlive === 0) {
      const winner = enemyAlive === 0 ? 'プレイヤーの勝利！' : '敵の勝利...';
      alert(winner);
      resetToStartScreen();
    }
  }

  function resetToStartScreen() {
    document.getElementById('battle-field').style.display = 'none';
    document.getElementById('card-selection').style.display = 'block';
    document.getElementById('player-cards').innerHTML = '';
    document.getElementById('enemy-cards').innerHTML = '';
    energy = 3;
    isPlayerTurn = true;
    selectedAttacker = null;
    selectedTarget = null;
    document.querySelectorAll('.card-pool .card').forEach(c => {
      c.classList.remove('attacker-selected', 'target-selected');
      updateCardDisplay(c);
    });
    updateSelectionUI();
    updateBattleUI();
  }

  // 指定サイドのカードの _skillUsed フラグをリセットする
  function resetSkillUsage(side) {
    const selector = side === 'player' ? '#player-cards .card' : '#enemy-cards .card';
    document.querySelectorAll(selector).forEach(card => {
      delete card.dataset._skillUsed;
      // 表示更新
      updateCardDisplay(card);
    });
  }

});
