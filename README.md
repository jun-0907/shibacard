# shibacard

<html lang="ja">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>カードバトル（UI改善・日本語化）</title>
  <style>
    :root{
      --bg:#f4f6f8; --card-bg:#fff; --card-border:#cfcfcf; --primary:#2196f3; --muted:#666;
      --accent:#ff9800; --danger:#e53935; --ok:#4caf50; --buff:#2e7d32; --debuff:#c62828;
    }
    html,body{height:100%;margin:0;padding:0;font-family:system-ui,"Hiragino Kaku Gothic ProN","Segoe UI",Roboto,Arial; background:var(--bg); color:#111}
    #game-root{max-width:1200px;margin:20px auto;display:grid;grid-template-columns:1fr 360px;gap:18px}
    header{grid-column:1/-1}
    h1{margin:0 0 8px;font-size:20px}
    .panel{background:#fff;border-radius:10px;padding:14px;box-shadow:0 6px 18px rgba(0,0,0,0.06)}
    .card-pool, .card-row{display:flex;flex-wrap:wrap;gap:12px;margin:10px 0;align-items:flex-start}
    .card{width:140px;min-height:120px;background:var(--card-bg);border:2px solid var(--card-border);border-radius:8px;padding:8px;cursor:pointer;white-space:pre-line;font-size:13px;line-height:1.3;box-shadow:0 1px 3px rgba(0,0,0,0.04);display:flex;flex-direction:column;justify-content:space-between;user-select:none}
    .card.selected{border-color:var(--primary);background:#e8f6ff}
    .card.disabled{opacity:0.45;pointer-events:none;filter:grayscale(30%)}
    .card.attacker-selected{border-color:var(--accent);box-shadow:0 6px 14px rgba(255,152,0,0.08);background:linear-gradient(180deg,#fff7ef,#fffdf9)}
    .card.target-selected{border-color:var(--danger);box-shadow:0 6px 14px rgba(229,57,53,0.08);background:linear-gradient(180deg,#fff5f5,#fffdfd)}
    .meta{font-size:12px;color:var(--muted);margin-top:6px}
    .controls{display:flex;gap:8px;align-items:center;margin-top:10px}
    button{padding:10px 14px;border:none;border-radius:8px;background:var(--primary);color:#fff;cursor:pointer}
    button:disabled{background:#ddd;color:#888;cursor:default}
    .row{display:flex;gap:12px;align-items:center}
    #detail-panel{position:sticky;top:20px;height:fit-content}
    .detail-title{font-weight:700;margin-bottom:6px}
    .stat{font-size:14px;margin:6px 0}
    .buff-list{font-size:13px;margin:4px 0}
    .badge{display:inline-block;padding:4px 8px;border-radius:12px;background:#f0f0f0;font-size:12px;margin-right:6px}
    .special-badge{background:#fff4e5;border:1px solid #ffd7a8;color:#b35900}
    .buff-badge{background:#e8f5e9;border:1px solid #c8e6c9;color:var(--buff);padding:3px 6px;border-radius:8px;margin-right:6px}
    .debuff-badge{background:#ffebee;border:1px solid #ffcdd2;color:var(--debuff);padding:3px 6px;border-radius:8px;margin-right:6px}
    .small{font-size:12px;color:var(--muted)}
    .buff-remaining{font-weight:700;color:var(--primary)}
    @media (max-width:1000px){ #game-root{grid-template-columns:1fr} #detail-panel{position:static} .card{width:120px} }
    #selection-detail { margin-top:10px; font-size:13px; color:var(--muted); }
    .jp-label { color:var(--muted); font-size:12px; display:inline-block; width:86px; }
  </style>
</head>
<body>
  <div id="game-root">
    <header>
      <h1>カードバトルゲーム（UI改善・日本語化）</h1>
      <div class="small">・攻撃は常にエネルギー1消費・スキルはカードごとのコストを消費します</div>
    </header>

    <main class="panel" id="left-panel">
      <section id="card-selection">
        <h2>カードを5枚選んでください</h2>
        <div class="card-pool"></div>
        <div id="selection-detail" style="display:none"></div>
        <div class="controls">
          <button id="start-battle" disabled>バトル開始</button>
          <div class="small">選択済み: <span id="selected-count">0</span>/5</div>
        </div>
      </section>

      <section id="battle-field" style="display:none;margin-top:12px;">
        <h2>バトルフィールド</h2>
        <div class="row" style="justify-content:space-between">
          <div>ターン: <strong id="turn-display">プレイヤー</strong></div>
          <div>エネルギー: <strong id="energy-count">10</strong>/<span id="energy-max">10</span></div>
          <div class="small">回復量/ターン: <span id="energy-regen">5</span>（倍率適用あり）</div>
        </div>

        <h3>プレイヤーのカード</h3>
        <div id="player-cards" class="card-row"></div>

        <h3>敵のカード</h3>
        <div id="enemy-cards" class="card-row"></div>

        <div class="controls">
          <button id="attack-button">攻撃（-1）</button>
          <button id="skill-button">スキル発動</button>
          <button id="end-turn">ターン終了</button>
        </div>
      </section>
    </main>

    <aside class="panel" id="detail-panel">
      <div class="detail-title">カード詳細</div>
      <div id="detail-empty" class="small">カードを選択すると詳細が表示されます</div>
      <div id="detail" style="display:none">
        <div class="stat"><span class="jp-label">カード名</span><strong id="d-name"></strong></div>
        <div class="stat"><span class="jp-label">体力</span><span id="d-hp"></span></div>
        <div class="stat"><span class="jp-label">攻撃 / 防御</span><span id="d-atk"></span> / <span id="d-def"></span></div>
        <div class="stat"><span class="jp-label">回復量</span><span id="d-heal"></span></div>
        <div class="stat"><span class="jp-label">スキル</span><span id="d-skill"></span>　<small id="d-cost"></small></div>
        <div class="stat"><span class="jp-label">攻撃対象</span><span id="d-attackTarget"></span></div>
        <div class="stat"><span class="jp-label">特性</span><span id="d-specials"></span></div>
        <div class="stat"><span class="jp-label">永続バフ</span><div id="d-perm" class="buff-list"></div></div>
        <div class="stat"><span class="jp-label">一時バフ</span><div id="d-temp" class="buff-list"></div></div>
        <div class="stat"><span class="jp-label">永続デバフ</span><div id="d-perm-debuff" class="buff-list"></div></div>
      </div>
    </aside>
  </div>

<script>
document.addEventListener('DOMContentLoaded', () => {
  // ========== カード定義 ==========
  const cardPoolConfig = [
    { id:'1',  name:'挑発兵',   hp:100, atk:20, def:12, heal:0,  skill:'buff',    cost:2,  hpCost:0, hpCostPercent:0, buffParams:{hp:1,atk:1,def:1,heal:1}, buffDuration:2, buffTarget:'self',    buffPermanent:false, buffChanceRepeat:0.0, special:['taunt'], attackTarget:'enemySingle' },
    { id:'2',  name:'戦闘医',   hp:80,  atk:15, def:8,  heal:20, skill:'heal',    cost:3,  hpCost:0, hpCostPercent:0, buffParams:{hp:1,atk:1,def:1,heal:1.2}, buffDuration:1, buffTarget:'all',     buffPermanent:false, buffChanceRepeat:0.0, special:[], attackTarget:'enemySingle' },
    { id:'3',  name:'バッファー',hp:120, atk:10, def:15, heal:0,  skill:'buff',    cost:4,  hpCost:0, hpCostPercent:0, buffParams:{hp:1.15,atk:1.25,def:1.1,heal:1}, buffDuration:2, buffTarget:'all',     buffPermanent:false, buffChanceRepeat:0.5, special:[], attackTarget:'enemySingle' },
    { id:'4',  name:'突撃兵',   hp:90,  atk:40, def:6,  heal:0,  skill:'normal',  cost:3,  hpCost:0, hpCostPercent:0, buffParams:{hp:1,atk:1,def:1,heal:1}, buffDuration:0, buffTarget:'self',    buffPermanent:false, buffChanceRepeat:0.0, special:['ignoreDef'], attackTarget:'enemySingle' },
    { id:'5',  name:'破壊者',   hp:110, atk:22, def:10, heal:0,  skill:'aoe',     cost:5,  hpCost:0, hpCostPercent:0, buffParams:{hp:1,atk:1.1,def:1,heal:1}, buffDuration:0, buffTarget:'self',    buffPermanent:false, buffChanceRepeat:0.0, special:[], attackTarget:'enemyAll' },
    { id:'6',  name:'狂戦士',   hp:70,  atk:55, def:4,  heal:0,  skill:'normal',  cost:4,  hpCost:10, hpCostPercent:0.0, buffParams:{hp:1,atk:1.2,def:1,heal:1}, buffDuration:0, buffTarget:'self',    buffPermanent:false, buffChanceRepeat:0.0, special:['grantExtraAction'], attackTarget:'enemySingle' },
    { id:'7',  name:'防御主任', hp:95,  atk:18, def:20, heal:0,  skill:'buff',    cost:3,  hpCost:0, hpCostPercent:0, buffParams:{hp:1.1,atk:1,def:1.2,heal:1}, buffDuration:3, buffTarget:'self',    buffPermanent:false, buffChanceRepeat:0.2, special:['invincible'], attackTarget:'enemySingle' },
    { id:'8',  name:'エネルギー長',hp:85,  atk:20, def:9,  heal:0,  skill:'buff',    cost:4,  hpCost:0, hpCostPercent:0, buffParams:{hp:1,atk:1,def:1,heal:1}, buffDuration:0, buffTarget:'self',    buffPermanent:true,  buffChanceRepeat:0.0, special:['increaseMaxEnergy:5','doubleEnergyRegen'], attackTarget:'enemySingle' },
    { id:'9',  name:'弱体手',   hp:100, atk:16, def:11, heal:0,  skill:'debuff',  cost:3,  hpCost:0, hpCostPercent:0, buffParams:{hp:1,atk:1,def:0.8,heal:1}, buffDuration:2, buffTarget:'target',  buffPermanent:false, buffChanceRepeat:0.0, special:['permanentDebuffOverwrite'], attackTarget:'enemySingle' },
    { id:'10', name:'破滅者',   hp:75,  atk:48, def:5,  heal:0,  skill:'none',    cost:5,  hpCost:0, hpCostPercent:0, buffParams:{hp:1,atk:1,def:1,heal:1}, buffDuration:0, buffTarget:'self',    buffPermanent:false, buffChanceRepeat:0.0, special:[], attackTarget:'allExceptSelf' }
  ];

  // ========== 状態 ==========
  let selectedCards = [];
  let energy = 10;
  let playerMaxEnergy = 10;
  const BASE_ENERGY_REGEN = 5;
  let energyRegenMultiplier = 1;
  let isPlayerTurn = true;
  let selectedAttacker = null;
  let selectedTarget = null;

  // ========== DOM ==========
  const poolEl = document.querySelector('.card-pool');
  const startButton = document.getElementById('start-battle');
  const attackButton = document.getElementById('attack-button');
  const skillButton = document.getElementById('skill-button');
  const endTurnButton = document.getElementById('end-turn');
  const selectedCountEl = document.getElementById('selected-count');
  const energyCountEl = document.getElementById('energy-count');
  const energyMaxEl = document.getElementById('energy-max');
  const energyRegenEl = document.getElementById('energy-regen');
  const selectionDetail = document.getElementById('selection-detail');

  // detail panel elements
  const detailEl = document.getElementById('detail');
  const detailEmpty = document.getElementById('detail-empty');
  const dName = document.getElementById('d-name');
  const dHp = document.getElementById('d-hp');
  const dAtk = document.getElementById('d-atk');
  const dDef = document.getElementById('d-def');
  const dHeal = document.getElementById('d-heal');
  const dSkill = document.getElementById('d-skill');
  const dCost = document.getElementById('d-cost');
  const dAttackTarget = document.getElementById('d-attackTarget');
  const dSpecials = document.getElementById('d-specials');
  const dPerm = document.getElementById('d-perm');
  const dTemp = document.getElementById('d-temp');
  const dPermDebuff = document.getElementById('d-perm-debuff');

  // ========== プール初期化（選択画面で詳細を表示） ==========
  function buildCardPoolDOM(){
    poolEl.innerHTML = '';
    cardPoolConfig.forEach(cfg => {
      const el = document.createElement('div');
      el.className = 'card';
      el.tabIndex = 0;
      el.dataset.id = String(cfg.id);
      el.dataset.name = cfg.name;
      el.dataset.hp = String(cfg.hp);
      el.dataset.maxHp = String(cfg.hp);
      el.dataset.atk = String(cfg.atk);
      el.dataset.def = String(cfg.def);
      el.dataset.heal = String(cfg.heal);
      el.dataset.skill = cfg.skill;
      el.dataset.cost = String(cfg.cost || 0);
      el.dataset.hpCost = String(cfg.hpCost || 0);
      el.dataset.hpCostPercent = String(cfg.hpCostPercent || 0);
      el.dataset._buffsPermanent = JSON.stringify({hp:1,atk:1,def:1,heal:1});
      el.dataset._buffsTempList = JSON.stringify([]);
      el.dataset._configBuff = JSON.stringify({ params: cfg.buffParams || {hp:1,atk:1,def:1,heal:1}, duration: cfg.buffDuration || 0, target: cfg.buffTarget || 'self', permanent: !!cfg.buffPermanent, chanceRepeat: cfg.buffChanceRepeat || 0 });
      el.dataset._attackTarget = cfg.attackTarget || 'enemySingle';
      el.dataset._special = JSON.stringify(cfg.special || []);
      el.innerText = `${cfg.name}\n♡:${cfg.hp} ⚔:${cfg.atk} 🛡:${cfg.def}\nスキル:${humanSkill(cfg.skill)}\n行動コスト:${cfg.cost}\n攻撃: ${humanAttackTarget(cfg.attackTarget)}`;

      // クリックで選択状態の切替と「カード詳細」パネルに表示
      el.addEventListener('click', () => {
        const id = String(el.dataset.id);
        if (selectedCards.includes(id)) {
          selectedCards = selectedCards.filter(c => c !== id);
          el.classList.remove('selected');
        } else if (selectedCards.length < 5) {
          selectedCards.push(id);
          el.classList.add('selected');
        }
        updateSelectionUI();
        renderDetail(el); // ここで右側の「カード詳細」パネルに表示
      });

      // ホバーでも詳細を表示（選択画面での確認を快適にする）
      el.addEventListener('mouseover', () => {
        renderDetail(el);
      });

      poolEl.appendChild(el);
    });
    updateSelectionUI();
  }
  buildCardPoolDOM();

  function updateSelectionUI(){
    startButton.disabled = selectedCards.length !== 5;
    selectedCountEl.innerText = String(selectedCards.length);
    if (selectedCards.length === 0) selectionDetail.style.display = 'none';
  }

  function renderSelectionDetail(cardEl) {
    if (!cardEl) { selectionDetail.style.display = 'none'; return; }
    selectionDetail.style.display = 'block';
    const name = cardEl.dataset.name;
    const hp = cardEl.dataset.hp;
    const atk = cardEl.dataset.atk;
    const def = cardEl.dataset.def;
    const skill = humanSkill(cardEl.dataset.skill);
    const cost = cardEl.dataset.cost || 0;
    const attackTarget = humanAttackTarget(cardEl.dataset._attackTarget);
    const specials = JSON.parse(cardEl.dataset._special || '[]').map(s => specialToJapanese(s)).join(', ') || 'なし';
    selectionDetail.innerHTML =
      `<div><strong>${name}</strong></div>
       <div class="small">体力: ${hp}　攻撃: ${atk}　防御: ${def}</div>
       <div class="small">スキル: ${skill}　スキルコスト: ${cost}</div>
       <div class="small">攻撃対象: ${attackTarget}</div>
       <div class="small">特性: ${specials}</div>`;
  }

  function humanSkill(s) {
    if (!s) return '-';
    if (s === 'normal') return '通常';
    if (s === 'heal') return '回復（味方全体）';
    if (s === 'buff') return 'バフ';
    if (s === 'debuff') return 'デバフ';
    if (s === 'aoe') return '全体攻撃';
    if (s === 'none') return 'なし';
    return s;
  }
  function humanAttackTarget(t) {
    if (!t) return '-';
    if (t === 'enemySingle') return '敵単体';
    if (t === 'enemyAll') return '敵全体';
    if (t === 'allExceptSelf') return '自分以外すべて';
    return t;
  }
  function specialToJapanese(s) {
    if (!s) return '';
    if (s === 'invincible') return '無敵（ダメージ無効）';
    if (s === 'ignoreDef') return '防御力無視（攻撃時）';
    if (s === 'grantExtraAction') return '味方に再行動付与';
    if (s.startsWith('increaseMaxEnergy:')) return `最大エネ増加 +${s.split(':')[1]}`;
    if (s === 'doubleEnergyRegen') return 'エネルギー回復2倍';
    if (s === 'taunt') return '挑発（ターゲット誘導）';
    if (s === 'permanentDebuffOverwrite') return '永続デバフ（上書き）';
    return s;
  }

  // ========== 戦闘開始 ==========
  startButton.addEventListener('click', () => {
    document.getElementById('card-selection').style.display = 'none';
    document.getElementById('battle-field').style.display = 'block';
    renderCards('player-cards', getSelectedCardDataFromConfig());
    renderCards('enemy-cards', generateEnemyCardsFromPoolConfig());
    resetAllTurnFlags();
    updateBattleUI();
    energy = playerMaxEnergy;
    updateEnergyUI();
    selectionDetail.style.display = 'none';
  });

  function getSelectedCardDataFromConfig(){
    return selectedCards.map(id => {
      const cfg = cardPoolConfig.find(c => String(c.id) === String(id));
      return {
        id: cfg.id,
        name: cfg.name,
        hp: cfg.hp,
        maxHp: cfg.hp,
        atk: cfg.atk,
        def: cfg.def,
        heal: cfg.heal,
        skill: cfg.skill,
        cost: cfg.cost || 0,
        hpCost: cfg.hpCost || 0,
        hpCostPercent: cfg.hpCostPercent || 0,
        buffsPermanent: {hp:1,atk:1,def:1,heal:1},
        buffsTempList: [],
        configBuff: { params: cfg.buffParams || {hp:1,atk:1,def:1,heal:1}, duration: cfg.buffDuration || 0, target: cfg.buffTarget || 'self', permanent: !!cfg.buffPermanent, chanceRepeat: cfg.buffChanceRepeat || 0 },
        attackTarget: cfg.attackTarget || 'enemySingle',
        special: cfg.special || []
      };
    });
  }

  function generateEnemyCardsFromPoolConfig(){
    const pool = cardPoolConfig.slice();
    for (let i = pool.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [pool[i], pool[j]] = [pool[j], pool[i]];
    }
    const pick = pool.slice(0, 5);
    return pick.map(cfg => ({
      id: `E${cfg.id}`,
      name: cfg.name,
      hp: cfg.hp,
      maxHp: cfg.hp,
      atk: cfg.atk,
      def: cfg.def,
      heal: cfg.heal,
      skill: cfg.skill,
      cost: cfg.cost || 0,
      hpCost: cfg.hpCost || 0,
      hpCostPercent: cfg.hpCostPercent || 0,
      buffsPermanent: {hp:1,atk:1,def:1,heal:1},
      buffsTempList: [],
      configBuff: { params: cfg.buffParams || {hp:1,atk:1,def:1,heal:1}, duration: cfg.buffDuration || 0, target: cfg.buffTarget || 'self', permanent: !!cfg.buffPermanent, chanceRepeat: cfg.buffChanceRepeat || 0 },
      attackTarget: cfg.attackTarget || 'enemySingle',
      special: cfg.special || []
    }));
  }

  function renderCards(containerId, cardData) {
    const container = document.getElementById(containerId);
    container.innerHTML = '';
    cardData.forEach(data => {
      const card = document.createElement('div');
      card.className = 'card';
      card.dataset.id = String(data.id);
      card.dataset.name = data.name ?? '';
      card.dataset.hp = String(data.hp);
      card.dataset.maxHp = String(data.maxHp);
      card.dataset.atk = String(data.atk);
      card.dataset.def = String(data.def);
      card.dataset.heal = String(data.heal);
      card.dataset.skill = data.skill ?? 'normal';
      card.dataset.cost = String(data.cost || 0);
      card.dataset.hpCost = String(data.hpCost || 0);
      card.dataset.hpCostPercent = String(data.hpCostPercent || 0);
      card.dataset._buffsPermanent = JSON.stringify(data.buffsPermanent ?? {hp:1,atk:1,def:1,heal:1});
      card.dataset._buffsTempList = JSON.stringify(data.buffsTempList ?? []);
      card.dataset._configBuff = JSON.stringify(data.configBuff ?? {params:{hp:1,atk:1,def:1,heal:1},duration:0,target:'self',permanent:false, chanceRepeat:0});
      card.dataset._attackTarget = data.attackTarget || 'enemySingle';
      card.dataset._special = JSON.stringify(data.special || []);
      delete card.dataset._skillUsed;
      delete card.dataset._attacked;
      updateCardDisplay(card);
      card.addEventListener('click', () => {
        handleCardClick(card, containerId);
        renderDetail(card);
      });
      container.appendChild(card);
    });
  }

  function computeTotalMultiplier(card) {
    const perm = JSON.parse(card.dataset._buffsPermanent ?? '{"hp":1,"atk":1,"def":1,"heal":1}');
    const tempList = JSON.parse(card.dataset._buffsTempList ?? '[]');
    const total = { hp: perm.hp || 1, atk: perm.atk || 1, def: perm.def || 1, heal: perm.heal || 1 };
    tempList.forEach(t => {
      total.hp   *= (t.hp   || 1);
      total.atk  *= (t.atk  || 1);
      total.def  *= (t.def  || 1);
      total.heal *= (t.heal || 1);
    });
    return total;
  }

  function getEffectiveStat(card, key) {
    const base = parseInt(card.dataset[key] ?? '0', 10);
    const total = computeTotalMultiplier(card);
    const mult = key === 'maxHp' ? (total.hp || 1) : (total[key] || 1);
    return Math.round(base * mult);
  }

  function updateCardDisplay(card) {
    const id = card.dataset.id ?? '';
    const rawHp = parseInt(card.dataset.hp ?? '0', 10);
    const effMaxHp = getEffectiveStat(card, 'maxHp');
    const effAtk = getEffectiveStat(card, 'atk');
    const effDef = getEffectiveStat(card, 'def');
    const effHeal = getEffectiveStat(card, 'heal');
    const skill = humanSkill(card.dataset.skill);
    const cost = card.dataset.cost || '0';
    const specials = JSON.parse(card.dataset._special || '[]').map(s=>specialToJapanese(s)).join(',');
    const usedText = card.dataset._skillUsed === 'true' ? '🔒(スキル使用済)' : '';
    const attackedText = card.dataset._attacked === 'true' ? '⚔(行動済)' : '';
    const tempList = JSON.parse(card.dataset._buffsTempList || '[]');
    const tempText = tempList.map(t => {
      const parts = [];
      if ((t.hp||1) !== 1) parts.push(`体力×${t.hp}`);
      if ((t.atk||1) !== 1) parts.push(`攻撃×${t.atk}`);
      if ((t.def||1) !== 1) parts.push(`防御×${t.def}`);
      if ((t.heal||1) !== 1) parts.push(`回復×${t.heal}`);
      return parts.length ? `${parts.join(',')}（残:${t.remainingTurns}）` : null;
    }).filter(Boolean).join('; ');

    card.innerText =
`🆔 ${card.dataset.name || id}
♡ ${rawHp}/${effMaxHp}
⚔ ${effAtk}  🛡 ${effDef}  💖 ${effHeal}
スキル: ${skill} ${usedText}
行動コスト: ${cost}（攻撃は固定 -1）
特性: ${specials || 'なし'}
バフ: ${tempText || 'なし'}
${attackedText}`.trim();

    if (rawHp <= 0) {
      if (!card.classList.contains('disabled')) card.classList.add('disabled');
      if (!card.innerText.includes('🛑 戦闘不能')) card.innerText += '\n🛑 戦闘不能';
    } else {
      card.classList.remove('disabled');
    }
  }

  function handleCardClick(card, side) {
    if (card.classList.contains('disabled')) return;
    if (side === 'player-cards') selectedAttacker = card; else selectedTarget = card;
    document.querySelectorAll('.card').forEach(c => c.classList.remove('attacker-selected', 'target-selected'));
    if (selectedAttacker) selectedAttacker.classList.add('attacker-selected');
    if (selectedTarget) selectedTarget.classList.add('target-selected');
    updateBattleUI();
  }

  function renderDetail(card) {
    if (!card) { detailEl.style.display='none'; detailEmpty.style.display='block'; return; }
    detailEmpty.style.display='none';
    detailEl.style.display='block';
    dName.innerText = card.dataset.name || card.dataset.id;
    dHp.innerText = `${card.dataset.hp}/${getEffectiveStat(card,'maxHp')}`;
    dAtk.innerText = getEffectiveStat(card,'atk');
    dDef.innerText = getEffectiveStat(card,'def');
    dHeal.innerText = getEffectiveStat(card,'heal');
    dSkill.innerText = humanSkill(card.dataset.skill);
    dCost.innerText = `スキルコスト: ${card.dataset.cost || 0}`;
    dAttackTarget.innerText = humanAttackTarget(card.dataset._attackTarget);
    const specials = JSON.parse(card.dataset._special || '[]');
    dSpecials.innerHTML = specials.length ? specials.map(s=>`<span class="badge special-badge">${specialToJapanese(s)}</span>`).join('') : 'なし';
    const perm = JSON.parse(card.dataset._buffsPermanent || '{"hp":1,"atk":1,"def":1,"heal":1}');
    dPerm.innerHTML = `体力 × ${perm.hp}, 攻撃 × ${perm.atk}, 防御 × ${perm.def}, 回復 × ${perm.heal}`;
    const temp = JSON.parse(card.dataset._buffsTempList || '[]');
    if (temp.length===0) dTemp.innerText='なし'; else {
      dTemp.innerHTML = temp.map(t=>{
        const parts = [];
        if ((t.hp||1)!==1) parts.push(`<span class="${t.hp>1?'buff-badge':'debuff-badge'}">体力×${t.hp}</span>`);
        if ((t.atk||1)!==1) parts.push(`<span class="${t.atk>1?'buff-badge':'debuff-badge'}">攻撃×${t.atk}</span>`);
        if ((t.def||1)!==1) parts.push(`<span class="${t.def>1?'buff-badge':'debuff-badge'}">防御×${t.def}</span>`);
        if ((t.heal||1)!==1) parts.push(`<span class="${t.heal>1?'buff-badge':'debuff-badge'}">回復×${t.heal}</span>`);
        return `<div>${parts.join(' ')} <span class="buff-remaining">（残:${t.remainingTurns}）</span></div>`;
      }).join('');
    }
    dPermDebuff.innerHTML = card.dataset._permanentDebuff ? (() => {
      const pd = JSON.parse(card.dataset._permanentDebuff);
      const parts = [];
      if ((pd.hp||1)!==1) parts.push(`<span class="${pd.hp>1?'buff-badge':'debuff-badge'}">体力×${pd.hp}</span>`);
      if ((pd.atk||1)!==1) parts.push(`<span class="${pd.atk>1?'buff-badge':'debuff-badge'}">攻撃×${pd.atk}</span>`);
      if ((pd.def||1)!==1) parts.push(`<span class="${pd.def>1?'buff-badge':'debuff-badge'}">防御×${pd.def}</span>`);
      if ((pd.heal||1)!==1) parts.push(`<span class="${pd.heal>1?'buff-badge':'debuff-badge'}">回復×${pd.heal}</span>`);
      return parts.join(' ');
    })() : 'なし';
  }

  // ========== ロジック（ダメージ/バフ/攻撃/スキル/AI/ターン管理） ==========
  function applyDamage(target, atkRaw, options = {}) {
    if (!target) return;
    if (parseInt(target.dataset._invincibleRemaining || '0', 10) > 0) return;
    const ignoreDef = options.ignoreDef === true;
    const effDef = ignoreDef ? 0 : getEffectiveStat(target, 'def');
    const damage = Math.max(0, atkRaw - effDef);
    let hp = parseInt(target.dataset.hp ?? '0', 10);
    hp -= damage;
    target.dataset.hp = String(hp);
    updateCardDisplay(target);
    if (hp <= 0) {
      if (selectedAttacker === target) selectedAttacker = null;
      if (selectedTarget === target) selectedTarget = null;
    }
    if (detailEl.style.display!=='none' && detailEl) {
      const showingId = dName.innerText;
      if (showingId === target.dataset.name) renderDetail(target);
    }
  }

  function applyBuffToCard(targetCard, buffs, duration, permanent, options = {}) {
    if (!targetCard) return;
    const isDebuff = (buffs.def && buffs.def < 1) || (buffs.atk && buffs.atk < 1) || (buffs.hp && buffs.hp < 1) || (buffs.heal && buffs.heal < 1);
    if (permanent && isDebuff && options.permanentDebuffOverwrite) {
      targetCard.dataset._permanentDebuff = JSON.stringify(buffs);
    }
    if (permanent && !isDebuff) {
      const perm = JSON.parse(targetCard.dataset._buffsPermanent ?? '{"hp":1,"atk":1,"def":1,"heal":1}');
      perm.hp   = (perm.hp   || 1) * (buffs.hp   || 1);
      perm.atk  = (perm.atk  || 1) * (buffs.atk  || 1);
      perm.def  = (perm.def  || 1) * (buffs.def  || 1);
      perm.heal = (perm.heal || 1) * (buffs.heal || 1);
      targetCard.dataset._buffsPermanent = JSON.stringify(perm);
      if (options.increaseMaxEnergy) {
        playerMaxEnergy += options.increaseMaxEnergy;
        energy = Math.min(energy, playerMaxEnergy);
        energyMaxEl.innerText = String(playerMaxEnergy);
      }
      if (options.doubleEnergyRegen) energyRegenMultiplier *= 2;
    } else if (!permanent) {
      const list = JSON.parse(targetCard.dataset._buffsTempList ?? '[]');
      list.push({
        hp:   buffs.hp   ?? 1,
        atk:  buffs.atk  ?? 1,
        def:  buffs.def  ?? 1,
        heal: buffs.heal ?? 1,
        remainingTurns: Math.max(1, duration || 1)
      });
      targetCard.dataset._buffsTempList = JSON.stringify(list);
    }
    const specials = JSON.parse(targetCard.dataset._special || '[]');
    if (specials.includes('invincible') && duration > 0) {
      targetCard.dataset._invincibleRemaining = String(Math.max(parseInt(targetCard.dataset._invincibleRemaining || '0',10), duration));
    }
    if (specials.includes('grantExtraAction')) delete targetCard.dataset._attacked;
    if (specials.includes('taunt') && duration>0) targetCard.dataset._tauntRemaining = String(Math.max(parseInt(targetCard.dataset._tauntRemaining || '0',10), duration));
    updateCardDisplay(targetCard);
    if (detailEl.style.display!=='none' && detailEl) {
      const showingId = dName.innerText;
      if (showingId === targetCard.dataset.name) renderDetail(targetCard);
    }
  }

  // 攻撃（エネ-1）
  attackButton.addEventListener('click', () => {
    if (!isPlayerTurn || selectedAttacker == null) return;
    if (selectedAttacker.classList.contains('disabled')) { selectedAttacker = null; updateBattleUI(); return; }
    if (selectedAttacker.dataset._attacked === 'true') return;
    if (energy < 1) return;
    energy -= 1;
    updateEnergyUI();

    const effAtk = getEffectiveStat(selectedAttacker, 'atk');
    const attackMode = selectedAttacker.dataset._attackTarget || 'enemySingle';
    const isPlayerSide = !!selectedAttacker.closest('#player-cards');
    const attackerSpecials = JSON.parse(selectedAttacker.dataset._special || '[]');
    const ignoreDefAttacker = attackerSpecials.includes('ignoreDef');

    if (attackMode === 'enemyAll') {
      const selector = isPlayerSide ? '#enemy-cards .card:not(.disabled)' : '#player-cards .card:not(.disabled)';
      document.querySelectorAll(selector).forEach(t => applyDamage(t, effAtk, { ignoreDef: ignoreDefAttacker }));
    } else if (attackMode === 'allExceptSelf') {
      const all = Array.from(document.querySelectorAll('#player-cards .card:not(.disabled), #enemy-cards .card:not(.disabled)'));
      all.forEach(t => { if (t !== selectedAttacker) applyDamage(t, effAtk, { ignoreDef: ignoreDefAttacker }); });
    } else {
      const sideSelector = isPlayerSide ? '#enemy-cards .card:not(.disabled)' : '#player-cards .card:not(.disabled)';
      const potentialTaunts = Array.from(document.querySelectorAll(sideSelector)).filter(c => parseInt(c.dataset._tauntRemaining || '0',10) > 0);
      let target = null;
      if (selectedTarget && selectedTarget.closest && ((isPlayerSide && selectedTarget.closest('#enemy-cards')) || (!isPlayerSide && selectedTarget.closest('#player-cards')))) {
        target = selectedTarget;
      } else if (potentialTaunts.length > 0) {
        target = potentialTaunts[Math.floor(Math.random() * potentialTaunts.length)];
      } else {
        const list = Array.from(document.querySelectorAll(sideSelector));
        if (list.length > 0) target = list[Math.floor(Math.random() * list.length)];
      }
      if (target) applyDamage(target, effAtk, { ignoreDef: ignoreDefAttacker });
    }

    selectedAttacker.dataset._attacked = 'true';
    updateCardDisplay(selectedAttacker);
    sanitizeSelectedRefs();
    updateBattleUI();
    checkGameEnd();
  });

  // スキル（カードごとの cost を消費）
  skillButton.addEventListener('click', () => {
    if (!isPlayerTurn || selectedAttacker == null) return;
    if (selectedAttacker.classList.contains('disabled')) { selectedAttacker = null; updateBattleUI(); return; }
    if (selectedAttacker.dataset._skillUsed === 'true') return;
    const cost = parseInt(selectedAttacker.dataset.cost || '0', 10);
    if (energy < cost) return;

    // HP cost
    const hpCost = parseInt(selectedAttacker.dataset.hpCost || '0', 10);
    const hpCostPercent = parseFloat(selectedAttacker.dataset.hpCostPercent || '0');
    let currentHp = parseInt(selectedAttacker.dataset.hp || '0', 10);
    const hpCostVal = hpCost > 0 ? hpCost : Math.floor(currentHp * (hpCostPercent || 0));
    if (hpCostVal > 0) {
      currentHp = Math.max(0, currentHp - hpCostVal);
      selectedAttacker.dataset.hp = String(currentHp);
      if (currentHp <= 0) { updateCardDisplay(selectedAttacker); sanitizeSelectedRefs(); updateBattleUI(); checkGameEnd(); return; }
    }

    energy -= cost;
    updateEnergyUI();

    let config = JSON.parse(selectedAttacker.dataset._configBuff || 'null');
    if (!config || !config.params) {
      const rawId = String(selectedAttacker.dataset.id);
      const baseId = rawId.startsWith('E') ? rawId.slice(1) : rawId;
      const poolCfg = cardPoolConfig.find(c => String(c.id) === String(baseId));
      if (poolCfg) config = { params: poolCfg.buffParams || {hp:1,atk:1,def:1,heal:1}, duration: poolCfg.buffDuration || 0, target: poolCfg.buffTarget || 'self', permanent: !!poolCfg.buffPermanent, chanceRepeat: poolCfg.buffChanceRepeat || 0 };
      else config = { params:{hp:1,atk:1,def:1,heal:1}, duration:0, target:'self', permanent:false, chanceRepeat:0 };
    }

    const skill = selectedAttacker.dataset.skill || 'normal';
    const isPlayerSide = !!selectedAttacker.closest('#player-cards');

    if (skill === 'heal') {
      const selector = isPlayerSide ? '#player-cards .card:not(.disabled)' : '#enemy-cards .card:not(.disabled)';
      document.querySelectorAll(selector).forEach(a => {
        const effHeal = getEffectiveStat(a, 'heal');
        const maxHp = getEffectiveStat(a, 'maxHp');
        const cur = parseInt(a.dataset.hp || '0',10);
        a.dataset.hp = String(Math.min(maxHp, cur + effHeal));
        updateCardDisplay(a);
      });
    } else if (skill === 'buff') {
      applyBuffAccordingToConfig(selectedAttacker, config, isPlayerSide);
    } else if (skill === 'debuff') {
      applyDebuffAccordingToConfig(selectedAttacker, config, isPlayerSide);
    } else if (skill === 'aoe') {
      const selector = isPlayerSide ? '#enemy-cards .card:not(.disabled)' : '#player-cards .card:not(.disabled)';
      document.querySelectorAll(selector).forEach(t => applyDamage(t, getEffectiveStat(selectedAttacker, 'atk')));
    }

    selectedAttacker.dataset._skillUsed = 'true';
    updateCardDisplay(selectedAttacker);
    sanitizeSelectedRefs();
    updateBattleUI();
    checkGameEnd();
  });

  function applyBuffAccordingToConfig(actorCard, config, isPlayerSide) {
    if (config.target === 'all') {
      const selector = isPlayerSide ? '#player-cards .card:not(.disabled)' : '#enemy-cards .card:not(.disabled)';
      document.querySelectorAll(selector).forEach(t => applyBuffToCard(t, config.params, config.duration, config.permanent, { permanentDebuffOverwrite: config.permanentDebuffOverwrite, increaseMaxEnergy: extractIncreaseMaxEnergy(actorCard), doubleEnergyRegen: extractDoubleEnergyRegen(actorCard) }));
      maybeRepeatBuff(actorCard, config, () => applyBuffAccordingToConfig(actorCard, config, isPlayerSide));
    } else if (config.target === 'enemyAll') {
      const selector = isPlayerSide ? '#enemy-cards .card:not(.disabled)' : '#player-cards .card:not(.disabled)';
      document.querySelectorAll(selector).forEach(t => applyBuffToCard(t, config.params, config.duration, config.permanent, { permanentDebuffOverwrite: config.permanentDebuffOverwrite }));
      maybeRepeatBuff(actorCard, config, () => applyBuffAccordingToConfig(actorCard, config, isPlayerSide));
    } else if (config.target === 'target') {
      let targetCard = null;
      if (selectedTarget && selectedTarget.closest && ((isPlayerSide && selectedTarget.closest('#enemy-cards')) || (!isPlayerSide && selectedTarget.closest('#player-cards')))) targetCard = selectedTarget;
      else {
        const selector = isPlayerSide ? '#player-cards .card:not(.disabled)' : '#enemy-cards .card:not(.disabled)';
        const list = Array.from(document.querySelectorAll(selector));
        if (list.length>0) targetCard = list[Math.floor(Math.random()*list.length)];
      }
      if (targetCard) {
        applyBuffToCard(targetCard, config.params, config.duration, config.permanent, { permanentDebuffOverwrite: config.permanentDebuffOverwrite });
        maybeRepeatBuff(actorCard, config, () => applyBuffToCard(targetCard, config.params, config.duration, config.permanent, { permanentDebuffOverwrite: config.permanentDebuffOverwrite }));
      }
    } else {
      applyBuffToCard(actorCard, config.params, config.duration, config.permanent, { permanentDebuffOverwrite: config.permanentDebuffOverwrite, increaseMaxEnergy: extractIncreaseMaxEnergy(actorCard), doubleEnergyRegen: extractDoubleEnergyRegen(actorCard) });
      maybeRepeatBuff(actorCard, config, () => applyBuffToCard(actorCard, config.params, config.duration, config.permanent));
    }
  }

  function applyDebuffAccordingToConfig(actorCard, config, isPlayerSide) {
    if (config.target === 'enemyAll' || config.target === 'all') {
      const selector = isPlayerSide ? '#enemy-cards .card:not(.disabled)' : '#player-cards .card:not(.disabled)';
      document.querySelectorAll(selector).forEach(t => applyBuffToCard(t, config.params, config.duration, config.permanent, { permanentDebuffOverwrite: config.permanentDebuffOverwrite }));
      maybeRepeatBuff(actorCard, config, () => applyDebuffAccordingToConfig(actorCard, config, isPlayerSide));
    } else if (config.target === 'target') {
      let targetCard = null;
      if (selectedTarget && selectedTarget.closest && ((isPlayerSide && selectedTarget.closest('#enemy-cards')) || (!isPlayerSide && selectedTarget.closest('#player-cards')))) targetCard = selectedTarget;
      else {
        const selector = isPlayerSide ? '#enemy-cards .card:not(.disabled)' : '#player-cards .card:not(.disabled)';
        const list = Array.from(document.querySelectorAll(selector));
        if (list.length>0) targetCard = list[Math.floor(Math.random()*list.length)];
      }
      if (targetCard) {
        applyBuffToCard(targetCard, config.params, config.duration, config.permanent, { permanentDebuffOverwrite: config.permanentDebuffOverwrite });
        maybeRepeatBuff(actorCard, config, () => applyBuffToCard(targetCard, config.params, config.duration, config.permanent));
      }
    } else {
      const selector = isPlayerSide ? '#enemy-cards .card:not(.disabled)' : '#player-cards .card:not(.disabled)';
      const list = Array.from(document.querySelectorAll(selector));
      if (list.length>0) {
        const target = list[Math.floor(Math.random()*list.length)];
        applyBuffToCard(target, config.params, config.duration, config.permanent, { permanentDebuffOverwrite: config.permanentDebuffOverwrite });
      }
    }
  }

  function maybeRepeatBuff(actorCard, config, repeatFn) {
    if (!config) return;
    const chance = config.chanceRepeat || 0;
    if (Math.random() < chance) repeatFn();
  }

  function extractIncreaseMaxEnergy(card) {
    const specials = JSON.parse(card.dataset._special || '[]');
    for (const s of specials) {
      if (typeof s === 'string' && s.startsWith('increaseMaxEnergy:')) {
        const parts = s.split(':'); return parseInt(parts[1] || '0', 10) || 0;
      }
    }
    return 0;
  }
  function extractDoubleEnergyRegen(card) {
    const specials = JSON.parse(card.dataset._special || '[]');
    return specials.includes('doubleEnergyRegen');
  }

  // ========== 敵AI ==========
  function enemyTurn() {
    const enemies = Array.from(document.querySelectorAll('#enemy-cards .card:not(.disabled)'));
    const players = Array.from(document.querySelectorAll('#player-cards .card:not(.disabled)'));
    if (enemies.length === 0 || players.length === 0) return;
    enemies.forEach(enemy => {
      if (enemy.dataset._attacked === 'true') return;
      const skill = enemy.dataset.skill ?? 'normal';
      const rawId = String(enemy.dataset.id);
      const baseId = rawId.startsWith('E') ? rawId.slice(1) : rawId;
      const poolCfg = cardPoolConfig.find(c => String(c.id) === String(baseId));
      let cfg = poolCfg ? { params: poolCfg.buffParams || {hp:1,atk:1,def:1,heal:1}, duration: poolCfg.buffDuration || 0, target: poolCfg.buffTarget || 'self', permanent: !!poolCfg.buffPermanent, chanceRepeat: poolCfg.buffChanceRepeat || 0 } : null;

      if (skill !== 'normal' && enemy.dataset._skillUsed !== 'true') {
        const hp = parseInt(enemy.dataset.hp || '0', 10);
        const effMax = getEffectiveStat(enemy,'maxHp');
        if (skill === 'heal' && hp < effMax * 0.6) {
          document.querySelectorAll('#enemy-cards .card:not(.disabled)').forEach(a => {
            const effHeal = getEffectiveStat(a, 'heal');
            const maxHp = getEffectiveStat(a, 'maxHp');
            a.dataset.hp = String(Math.min(maxHp, parseInt(a.dataset.hp||'0',10) + effHeal));
            updateCardDisplay(a);
          });
          enemy.dataset._skillUsed = 'true';
          return;
        }
        if (skill === 'aoe') {
          document.querySelectorAll('#player-cards .card:not(.disabled)').forEach(t => applyDamage(t, getEffectiveStat(enemy, 'atk')));
          enemy.dataset._skillUsed = 'true';
          enemy.dataset._attacked = 'true';
          updateCardDisplay(enemy);
          return;
        }
        if (Math.random() < 0.35) {
          if (skill === 'buff' && cfg) {
            if (cfg.target === 'all') document.querySelectorAll('#enemy-cards .card:not(.disabled)').forEach(a => applyBuffToCard(a, cfg.params, cfg.duration, cfg.permanent));
            else if (cfg.target === 'enemyAll') document.querySelectorAll('#player-cards .card:not(.disabled)').forEach(a => applyBuffToCard(a, cfg.params, cfg.duration, cfg.permanent));
            else if (cfg.target === 'target') {
              const target = players[Math.floor(Math.random() * players.length)];
              if (target) applyBuffToCard(target, cfg.params, cfg.duration, cfg.permanent);
            } else applyBuffToCard(enemy, cfg.params, cfg.duration, cfg.permanent);
            enemy.dataset._skillUsed = 'true';
            updateCardDisplay(enemy);
            return;
          }
          if (skill === 'debuff' && cfg) {
            if (cfg.target === 'enemyAll' || cfg.target === 'all') document.querySelectorAll('#player-cards .card:not(.disabled)').forEach(t => applyBuffToCard(t, cfg.params, cfg.duration, cfg.permanent));
            else if (cfg.target === 'target') {
              const target = players[Math.floor(Math.random() * players.length)];
              if (target) applyBuffToCard(target, cfg.params, cfg.duration, cfg.permanent);
            } else {
              const target = players[Math.floor(Math.random() * players.length)];
              if (target) applyBuffToCard(target, cfg.params, cfg.duration, cfg.permanent);
            }
            enemy.dataset._skillUsed = 'true';
            enemy.dataset._attacked = 'true';
            return;
          }
        }
      }

      // 攻撃
      const effAtk = getEffectiveStat(enemy,'atk');
      const attackMode = enemy.dataset._attackTarget || 'enemySingle';
      if (attackMode === 'enemyAll') {
        document.querySelectorAll('#player-cards .card:not(.disabled)').forEach(t => applyDamage(t, effAtk));
      } else if (attackMode === 'allExceptSelf') {
        const all = Array.from(document.querySelectorAll('#player-cards .card:not(.disabled), #enemy-cards .card:not(.disabled)'));
        all.forEach(t => { if (t !== enemy) applyDamage(t, effAtk); });
      } else {
        const taunts = Array.from(document.querySelectorAll('#player-cards .card:not(.disabled)')).filter(c => parseInt(c.dataset._tauntRemaining || '0',10) > 0);
        let target = null;
        if (taunts.length > 0) target = taunts[Math.floor(Math.random() * taunts.length)];
        else {
          const list = Array.from(document.querySelectorAll('#player-cards .card:not(.disabled)'));
          if (list.length > 0) target = list[Math.floor(Math.random() * list.length)];
        }
        if (target) applyDamage(target, effAtk);
      }
      enemy.dataset._attacked = 'true';
      updateCardDisplay(enemy);
    });
  }

  // ========== ターン管理 ==========
  function endPlayerTurn() {
    decrementTempBuffsAndClear();
    isPlayerTurn = !isPlayerTurn;
    if (!isPlayerTurn) {
      setTimeout(() => {
        enemyTurn();
        decrementTempBuffsAndClear();
        isPlayerTurn = true;
        regenEnergyPerTurn();
        resetAllTurnFlags();
        updateBattleUI();
        checkGameEnd();
      }, 600);
    } else {
      regenEnergyPerTurn();
      resetAllTurnFlags();
    }
    updateBattleUI();
  }
  endTurnButton.addEventListener('click', () => endPlayerTurn());

  function regenEnergyPerTurn() {
    const regen = Math.round(BASE_ENERGY_REGEN * (energyRegenMultiplier || 1));
    energy = Math.min(playerMaxEnergy, energy + regen);
    updateEnergyUI();
  }

  function decrementTempBuffsAndClear() {
    document.querySelectorAll('#player-cards .card, #enemy-cards .card').forEach(card => {
      let list = JSON.parse(card.dataset._buffsTempList || '[]');
      list = list.map(t => ({...t, remainingTurns: (t.remainingTurns || 1) - 1})).filter(t => t.remainingTurns > 0);
      card.dataset._buffsTempList = JSON.stringify(list);
      if (card.dataset._tauntRemaining) {
        let val = Math.max(0, parseInt(card.dataset._tauntRemaining || '0', 10) - 1);
        if (val <= 0) delete card.dataset._tauntRemaining; else card.dataset._tauntRemaining = String(val);
      }
      if (card.dataset._invincibleRemaining) {
        let v = Math.max(0, parseInt(card.dataset._invincibleRemaining || '0', 10) - 1);
        if (v <= 0) delete card.dataset._invincibleRemaining; else card.dataset._invincibleRemaining = String(v);
      }
      delete card.dataset._skillUsed;
      updateCardDisplay(card);
    });
  }

  function resetAllTurnFlags() {
    document.querySelectorAll('#player-cards .card, #enemy-cards .card').forEach(card => {
      delete card.dataset._attacked;
      delete card.dataset._skillUsed;
      updateCardDisplay(card);
    });
  }

  function sanitizeSelectedRefs() {
    if (selectedAttacker && selectedAttacker.classList.contains('disabled')) selectedAttacker = null;
    if (selectedTarget && selectedTarget.classList.contains('disabled')) selectedTarget = null;
  }

  function updateEnergyUI() {
    energyCountEl.innerText = String(energy);
    energyMaxEl.innerText = String(playerMaxEnergy);
    energyRegenEl.innerText = String(Math.round(BASE_ENERGY_REGEN * (energyRegenMultiplier || 1)));
  }

  function updateBattleUI() {
    document.getElementById('turn-display').innerText = isPlayerTurn ? 'プレイヤー' : '敵';
    updateEnergyUI();
    attackButton.disabled = !(isPlayerTurn && selectedAttacker && !selectedAttacker.classList.contains('disabled') && selectedAttacker.dataset._attacked !== 'true' && energy >= 1);
    skillButton.disabled = !(isPlayerTurn && selectedAttacker && !selectedAttacker.classList.contains('disabled') && selectedAttacker.dataset._skillUsed !== 'true' && energy >= parseInt(selectedAttacker.dataset.cost || '0',10));
    endTurnButton.disabled = false;
    if (selectedAttacker && selectedAttacker.classList.contains('disabled')) selectedAttacker = null;
    if (selectedTarget && selectedTarget.classList.contains('disabled')) selectedTarget = null;
    document.querySelectorAll('.card').forEach(c => c.classList.remove('attacker-selected', 'target-selected'));
    if (selectedAttacker) selectedAttacker.classList.add('attacker-selected');
    if (selectedTarget) selectedTarget.classList.add('target-selected');
  }

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
    energy = 10; playerMaxEnergy = 10; energyRegenMultiplier = 1; isPlayerTurn = true; selectedAttacker = null; selectedTarget = null;
    document.querySelectorAll('.card-pool .card').forEach(c => c.classList.remove('selected','attacker-selected','target-selected'));
    selectedCards = [];
    updateSelectionUI();
    updateBattleUI();
    detailEl.style.display='none'; detailEmpty.style.display='block';
    selectionDetail.style.display='none';
  }

  // 初期 UI
  updateEnergyUI();
  updateSelectionUI();
  updateBattleUI();
});
</script>
</body>
</html>
