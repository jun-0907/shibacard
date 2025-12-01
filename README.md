# Shibacard

<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>カードバトル（付与元トラッキング・再付与制限・付与は死亡後も残る）</title>
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
    .card{background:var(--card-bg);border:2px solid var(--card-border);border-radius:8px;padding:8px;cursor:pointer;white-space:pre-line;font-size:13px;line-height:1.3;box-shadow:0 1px 3px rgba(0,0,0,0.04);display:flex;flex-direction:column;justify-content:space-between;user-select:none}
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
    .cooldown-badge{display:inline-block;padding:2px 6px;border-radius:10px;background:#333;color:#fff;font-size:11px;margin-left:6px}
    @media (max-width:1000px){ #game-root{grid-template-columns:1fr} #detail-panel{position:static} .card{width:120px} }

    /* 選択画面用：カードを小さくして名前のみ表示 */
    .card-pool .card{
      width:96px;
      height:56px;
      padding:6px;
      font-size:13px;
      display:flex;
      align-items:center;
      justify-content:center;
      text-align:center;
    }
    /* バトル画面用カードは少し大きく */
    #player-cards .card, #enemy-cards .card{
      width:140px;
      min-height:120px;
      padding:8px;
      text-align:left;
      position:relative;
    }

    #selection-detail { margin-top:10px; font-size:13px; color:var(--muted); }
    .jp-label { color:var(--muted); font-size:12px; display:inline-block; width:86px; }
  </style>
</head>
<body>
  <div id="game-root">
    <header>
      <h1>カードバトル（付与元トラッキング・再付与制限・付与は死亡後も残る）</h1>
      <div class="small">・初期エネ10・攻撃は常にエネルギー1消費（回復専用カードは別）・1ターン回復上限15</div>
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
          <div class="small">回復量/ターン: <span id="energy-regen">5</span>（最大15）</div>
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
        <div class="stat"><span class="jp-label">攻撃での回復</span><span id="d-healOnAttack"></span></div>
        <div class="stat"><span class="jp-label">スキル</span><span id="d-skill"></span>　<small id="d-cost"></small></div>
        <div class="stat"><span class="jp-label">攻撃対象</span><span id="d-attackTarget"></span></div>
        <div class="stat"><span class="jp-label">特性</span><span id="d-specials"></span></div>
        <div class="stat"><span class="jp-label">回復倍率（特性）</span><span id="d-energyMult"></span></div>
        <div class="stat"><span class="jp-label">永続バフ</span><div id="d-perm" class="buff-list"></div></div>
        <div class="stat"><span class="jp-label">一時バフ</span><div id="d-temp" class="buff-list"></div></div>
        <div class="stat"><span class="jp-label">永続デバフ</span><div id="d-perm-debuff" class="buff-list"></div></div>
      </div>
    </aside>
  </div>

<script>
document.addEventListener('DOMContentLoaded', () => {
  // ---------- カード定義（例） ----------
  // 特性: doubleenergyRegen, invincible, ignoreDef, taunt, increaseMaxEnergy:5, permanentDebuffOverwrite, lifeSteal 等
  const cardPoolConfig = [
    { id:'1',  name:'挑発兵',   hp:100, atk:20, def:12, heal:0,  skill:'buff',    cost:2,  hpCost:0, hpCostPercent:0, buffParams:{hp:1,atk:1,def:1,heal:1}, buffDuration:2, buffTarget:'self',    buffPermanent:false, buffChanceRepeat:0.0, special:['taunt'], attackTarget:'enemySingle' },
    { id:'2',  name:'戦闘医',   hp:80,  atk:15, def:8,  heal:20, skill:'heal',    cost:3,  hpCost:0, hpCostPercent:0, buffParams:{hp:1,atk:1,def:1,heal:1.2}, buffDuration:1, buffTarget:'all',     buffPermanent:false, buffChanceRepeat:0.0, special:[], attackTarget:'enemySingle' },
    { id:'3',  name:'バッファー',hp:120, atk:10, def:15, heal:0,  skill:'buff',    cost:4,  hpCost:0, hpCostPercent:0, buffParams:{hp:1.15,atk:1.25,def:1.1,heal:1}, buffDuration:2, buffTarget:'all',     buffPermanent:false, buffChanceRepeat:0.5, special:[], attackTarget:'enemySingle' },
    { id:'4',  name:'突撃兵',   hp:90,  atk:40, def:6,  heal:0,  skill:'normal',  cost:3,  hpCost:0, hpCostPercent:0, buffParams:{hp:1,atk:1,def:1,heal:1}, buffDuration:0, buffTarget:'self',    buffPermanent:false, buffChanceRepeat:0.0, special:['ignoreDef'], attackTarget:'enemySingle' },
    { id:'5',  name:'破壊者',   hp:110, atk:22, def:10, heal:0,  skill:'aoe',     cost:5,  hpCost:0, hpCostPercent:0, buffParams:{hp:1,atk:1.1,def:1,heal:1}, buffDuration:0, buffTarget:'self',    buffPermanent:false, buffChanceRepeat:0.0, special:[], attackTarget:'enemyAll' },
    { id:'6',  name:'狂戦士',   hp:70,  atk:55, def:4,  heal:0,  skill:'normal',  cost:4,  hpCost:10, hpCostPercent:0.0, buffParams:{hp:1,atk:1.2,def:1,heal:1}, buffDuration:0, buffTarget:'self',    buffPermanent:false, buffChanceRepeat:0.0, special:['grantExtraAction'], attackTarget:'enemySingle' },
    { id:'7',  name:'防御主任', hp:95,  atk:18, def:20, heal:0,  skill:'buff',    cost:3,  hpCost:0, hpCostPercent:0, buffParams:{hp:1.1,atk:1,def:1.2,heal:1}, buffDuration:3, buffTarget:'self',    buffPermanent:false, buffChanceRepeat:0.2, special:['invincible'], attackTarget:'enemySingle' },
    { id:'8',  name:'エネルギー長',hp:85,  atk:20, def:9,  heal:0,  skill:'buff',    cost:4,  hpCost:0, hpCostPercent:0, buffParams:{hp:1,atk:1,def:1,heal:1}, buffDuration:0, buffTarget:'self',    buffPermanent:true,  buffChanceRepeat:0.0, special:['increaseMaxEnergy:5'], attackTarget:'enemySingle' },
    // doubleenergyRegen を持つカード（場にいると回復が3倍になる）
    { id:'11', name:'エネルギー供給官', hp:70, atk:8, def:6, heal:0, skill:'none', cost:2, hpCost:0, hpCostPercent:0, buffParams:{hp:1,atk:1,def:1,heal:1}, buffDuration:0, buffTarget:'self', buffPermanent:false, buffChanceRepeat:0.0, special:['doubleenergyRegen'], attackTarget:'enemySingle' },
    { id:'9',  name:'弱体手',   hp:100, atk:16, def:11, heal:0,  skill:'debuff',  cost:3,  hpCost:0, hpCostPercent:0, buffParams:{hp:1,atk:1,def:0.8,heal:1}, buffDuration:2, buffTarget:'target',  buffPermanent:false, buffChanceRepeat:0.0, special:['permanentDebuffOverwrite'], attackTarget:'enemySingle' },
    { id:'10', name:'破滅者',   hp:75,  atk:48, def:5,  heal:0,  skill:'none',    cost:5,  hpCost:0, hpCostPercent:0, buffParams:{hp:1,atk:1,def:1,heal:1}, buffDuration:0, buffTarget:'self',    buffPermanent:false, buffChanceRepeat:0.0, special:[], attackTarget:'allExceptSelf' }
  ];

  // ---------- 状態 ----------
  let selectedCards = [];
  let energy = 10;
  let playerMaxEnergy = 10;
  const BASE_ENERGY_REGEN = 5;
  let isPlayerTurn = true;
  let selectedAttacker = null;
  let selectedTarget = null;
  let lastDetailCard = null;

  // 回復上限（1ターンあたりの最大回復量）
  const PER_TURN_REGEN_CAP = 15;

  // ---------- DOM ----------
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
  const dHealOnAttack = document.getElementById('d-healOnAttack');
  const dSkill = document.getElementById('d-skill');
  const dCost = document.getElementById('d-cost');
  const dAttackTarget = document.getElementById('d-attackTarget');
  const dSpecials = document.getElementById('d-specials');
  const dPerm = document.getElementById('d-perm');
  const dTemp = document.getElementById('d-temp');
  const dPermDebuff = document.getElementById('d-perm-debuff');
  const dEnergyMult = document.getElementById('d-energyMult');

  // ---------- ヘルパー（日本語化） ----------
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
    if (s.startsWith('increaseMaxEnergy:')) return '最大エネ増加 +' + s.split(':')[1];
    if (s === 'doubleenergyRegen') return '回復量を3倍にする（特性）';
    if (s === 'taunt') return '挑発（ターゲット誘導）';
    if (s === 'permanentDebuffOverwrite') return '永続デバフ（上書き）';
    if (s === 'lifeSteal') return '与ダメージで回復（吸血）';
    return s;
  }

  // ---------- プール初期化（選択画面は名前のみ表示） ----------
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
      // 永続バフは配列で管理（source を含めるため）
      el.dataset._permanentBuffs = JSON.stringify([]); // [{params:{...}, source:'id'}]
      el.dataset._buffsTempList = JSON.stringify([]);  // [{hp,atk,def,heal,remainingTurns,source}]
      el.dataset._configBuff = JSON.stringify({ params: cfg.buffParams || {hp:1,atk:1,def:1,heal:1}, duration: cfg.buffDuration || 0, target: cfg.buffTarget || 'self', permanent: !!cfg.buffPermanent, chanceRepeat: cfg.buffChanceRepeat || 0 });
      el.dataset._attackTarget = cfg.attackTarget || 'enemySingle';
      el.dataset._special = JSON.stringify(cfg.special || []);
      // 選択画面では名前のみ表示（小さく）
      el.innerText = `${cfg.name}`;

      // クリックで選択/解除、右側に詳細表示
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
        renderDetail(el);
      });

      // ホバーでも詳細表示
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

  // ---------- 戦闘開始 ----------
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
        permanentBuffs: [], // [{params, source}]
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
      permanentBuffs: [],
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
      card.dataset._permanentBuffs = JSON.stringify(data.permanentBuffs ?? []);
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
      card.addEventListener('mouseover', () => renderDetail(card));
      container.appendChild(card);
    });
  }

  function computeTotalMultiplier(card) {
    const permList = JSON.parse(card.dataset._permanentBuffs || '[]'); // [{params, source}]
    const tempList = JSON.parse(card.dataset._buffsTempList || '[]');
    const total = { hp:1, atk:1, def:1, heal:1 };
    permList.forEach(p => {
      const params = p.params || {};
      total.hp   *= (params.hp   || 1);
      total.atk  *= (params.atk  || 1);
      total.def  *= (params.def  || 1);
      total.heal *= (params.heal || 1);
    });
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
    const mult = (key === 'maxHp') ? (total.hp || 1) : (total[key] || 1);
    return Math.round(base * mult);
  }

  function updateCardDisplay(card) {
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
      const src = t.source ? `付与元:${t.source}` : '';
      return parts.length ? `${parts.join(',')}（残:${t.remainingTurns}） ${src}` : null;
    }).filter(Boolean).join('; ');

    if (card.closest('#player-cards') || card.closest('#enemy-cards')) {
      const invCd = parseInt(card.dataset._invincibleCooldown || '0', 10);
      const invRem = parseInt(card.dataset._invincibleRemaining || '0', 10);
      let baseText =
`🆔 ${card.dataset.name || card.dataset.id}
♡ ${rawHp}/${effMaxHp}
⚔ ${effAtk}  🛡 ${effDef}  💖 ${effHeal}
スキル: ${skill} ${usedText}
行動コスト: ${cost}（攻撃は固定 -1）
特性: ${specials || 'なし'}
バフ: ${tempText || 'なし'}
${attackedText}`.trim();
      card.innerText = baseText;

      const existing = card.querySelector('.cooldown-badge');
      if (existing) existing.remove();
      if (invRem > 0) {
        const b = document.createElement('div');
        b.className = 'cooldown-badge';
        b.style.position = 'absolute';
        b.style.top = '6px';
        b.style.right = '6px';
        b.innerText = `無敵:${invRem}`;
        card.appendChild(b);
      } else if (invCd > 0) {
        const b = document.createElement('div');
        b.className = 'cooldown-badge';
        b.style.position = 'absolute';
        b.style.top = '6px';
        b.style.right = '6px';
        b.innerText = `CD:${invCd}`;
        card.appendChild(b);
      }
    } else {
      card.innerText = card.dataset.name || card.dataset.id;
    }

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
    lastDetailCard = card || null;
    if (!card) { detailEl.style.display='none'; detailEmpty.style.display='block'; return; }
    detailEmpty.style.display='none';
    detailEl.style.display='block';
    dName.innerText = card.dataset.name || card.dataset.id;
    dHp.innerText = `${card.dataset.hp}/${getEffectiveStat(card,'maxHp')}`;
    dAtk.innerText = getEffectiveStat(card,'atk');
    dDef.innerText = getEffectiveStat(card,'def');
    dHeal.innerText = getEffectiveStat(card,'heal');
    const lifeStealRatio = card.dataset.lifeStealRatio ? parseFloat(card.dataset.lifeStealRatio) : 0;
    if (lifeStealRatio > 0) dHealOnAttack.innerText = `与ダメージの ${Math.round(lifeStealRatio*100)}% を回復`;
    else dHealOnAttack.innerText = card.dataset.heal && parseInt(card.dataset.heal||'0',10) > 0 ? `${card.dataset.heal} 回復（攻撃時）` : 'なし';
    dSkill.innerText = humanSkill(card.dataset.skill);
    dCost.innerText = `スキルコスト: ${card.dataset.cost || 0}`;
    dAttackTarget.innerText = humanAttackTarget(card.dataset._attackTarget);
    const specials = JSON.parse(card.dataset._special || '[]');
    dSpecials.innerHTML = specials.length ? specials.map(s=>`<span class="badge special-badge">${specialToJapanese(s)}</span>`).join('') : 'なし';

    // 永続バフ（配列）
    const permList = JSON.parse(card.dataset._permanentBuffs || '[]');
    if (permList.length === 0) dPerm.innerText = 'なし';
    else {
      dPerm.innerHTML = permList.map(p => {
        const pr = p.params || {};
        const parts = [];
        if ((pr.hp||1) !== 1) parts.push(`<span class="${pr.hp>1?'buff-badge':'debuff-badge'}">体力×${pr.hp}</span>`);
        if ((pr.atk||1) !== 1) parts.push(`<span class="${pr.atk>1?'buff-badge':'debuff-badge'}">攻撃×${pr.atk}</span>`);
        if ((pr.def||1) !== 1) parts.push(`<span class="${pr.def>1?'buff-badge':'debuff-badge'}">防御×${pr.def}</span>`);
        if ((pr.heal||1) !== 1) parts.push(`<span class="${pr.heal>1?'buff-badge':'debuff-badge'}">回復×${pr.heal}</span>`);
        const src = p.source ? `<small>付与元:${p.source}</small>` : '';
        return `<div>${parts.join(' ')} ${src}</div>`;
      }).join('');
    }

    // 一時バフ
    const temp = JSON.parse(card.dataset._buffsTempList || '[]');
    if (temp.length===0) dTemp.innerText='なし'; else {
      dTemp.innerHTML = temp.map(t=>{
        const parts = [];
        if ((t.hp||1)!==1) parts.push(`<span class="${t.hp>1?'buff-badge':'debuff-badge'}">体力×${t.hp}</span>`);
        if ((t.atk||1)!==1) parts.push(`<span class="${t.atk>1?'buff-badge':'debuff-badge'}">攻撃×${t.atk}</span>`);
        if ((t.def||1)!==1) parts.push(`<span class="${t.def>1?'buff-badge':'debuff-badge'}">防御×${t.def}</span>`);
        if ((t.heal||1)!==1) parts.push(`<span class="${t.heal>1?'buff-badge':'debuff-badge'}">回復×${t.heal}</span>`);
        const src = t.source ? `<small>付与元:${t.source}</small>` : '';
        return `<div>${parts.join(' ')} <span class="buff-remaining">（残:${t.remainingTurns}）</span> ${src}</div>`;
      }).join('');
    }

    // 永続デバフ（既存の _permanentDebuff を source 付きで保存している場合に対応）
    if (card.dataset._permanentDebuff) {
      try {
        const pd = JSON.parse(card.dataset._permanentDebuff);
        const parts = [];
        if ((pd.hp||1)!==1) parts.push(`<span class="${pd.hp>1?'buff-badge':'debuff-badge'}">体力×${pd.hp}</span>`);
        if ((pd.atk||1)!==1) parts.push(`<span class="${pd.atk>1?'buff-badge':'debuff-badge'}">攻撃×${pd.atk}</span>`);
        if ((pd.def||1)!==1) parts.push(`<span class="${pd.def>1?'buff-badge':'debuff-badge'}">防御×${pd.def}</span>`);
        if ((pd.heal||1)!==1) parts.push(`<span class="${pd.heal>1?'buff-badge':'debuff-badge'}">回復×${pd.heal}</span>`);
        const src = pd.source ? `<small>付与元:${pd.source}</small>` : '';
        dPermDebuff.innerHTML = `${parts.join(' ')} ${src}`;
      } catch(e) { dPermDebuff.innerText = 'なし'; }
    } else dPermDebuff.innerText = 'なし';

    // 特性に doubleenergyRegen があれば表示
    const energySpec = (specials || []).find(s => s === 'doubleenergyRegen');
    if (energySpec) {
      dEnergyMult.innerText = `回復倍率（特性）: ×3.00`;
    } else {
      dEnergyMult.innerText = 'なし';
    }
  }

  // ---------- ロジック（ダメージ / バフ / 攻撃 / スキル / AI / ターン管理） ----------
  function applyDamage(target, atkRaw, options = {}) {
    if (!target) return 0;
    if (parseInt(target.dataset._invincibleRemaining || '0', 10) > 0) return 0;
    const ignoreDef = options.ignoreDef === true;
    const effDef = ignoreDef ? 0 : getEffectiveStat(target, 'def');
    const damage = Math.max(0, atkRaw - effDef);
    let hp = parseInt(target.dataset.hp ?? '0', 10);
    hp -= damage;
    target.dataset.hp = String(hp);
    updateCardDisplay(target);
    if (lastDetailCard && lastDetailCard.dataset && lastDetailCard.dataset.id === target.dataset.id) renderDetail(target);
    if (hp <= 0) {
      // 死亡しても付与したバフは残す（仕様）
      if (selectedAttacker === target) selectedAttacker = null;
      if (selectedTarget === target) selectedTarget = null;
    }
    return damage;
  }

  function applyHealTo(card, amount) {
    if (!card || amount <= 0) return;
    const maxHp = getEffectiveStat(card, 'maxHp');
    const cur = parseInt(card.dataset.hp || '0', 10);
    card.dataset.hp = String(Math.min(maxHp, cur + amount));
    updateCardDisplay(card);
    if (lastDetailCard && lastDetailCard.dataset && lastDetailCard.dataset.id === card.dataset.id) renderDetail(card);
  }

  /**
   * applyBuffToCard
   * targetCard: DOM element
   * buffs: {hp,atk,def,heal}
   * duration: number (0 -> permanent if permanent flag true)
   * permanent: boolean
   * options: { sourceId: 'a', permanentDebuffOverwrite: bool, increaseMaxEnergy: num, actorHasInvincible: bool }
   */
  function applyBuffToCard(targetCard, buffs, duration, permanent, options = {}) {
    if (!targetCard) return;
    const source = options.sourceId || null;
    const isDebuff = (buffs.def && buffs.def < 1) || (buffs.atk && buffs.atk < 1) || (buffs.hp && buffs.hp < 1) || (buffs.heal && buffs.heal < 1);

    if (permanent && isDebuff && options.permanentDebuffOverwrite) {
      // 永続デバフを source 付きで保存
      const pd = {...buffs, source: source};
      targetCard.dataset._permanentDebuff = JSON.stringify(pd);
    } else if (permanent && !isDebuff) {
      // 永続バフは配列で管理（source を保持）
      const permList = JSON.parse(targetCard.dataset._permanentBuffs || '[]');
      permList.push({ params: buffs, source: source });
      targetCard.dataset._permanentBuffs = JSON.stringify(permList);
      if (options.increaseMaxEnergy) {
        playerMaxEnergy += options.increaseMaxEnergy;
        energy = Math.min(energy, playerMaxEnergy);
        energyMaxEl.innerText = String(playerMaxEnergy);
      }
    } else {
      // 一時バフに source を付与して保存
      const list = JSON.parse(targetCard.dataset._buffsTempList || '[]');
      list.push({
        hp:   buffs.hp   ?? 1,
        atk:  buffs.atk  ?? 1,
        def:  buffs.def  ?? 1,
        heal: buffs.heal ?? 1,
        remainingTurns: Math.max(1, duration || 1),
        source: source
      });
      targetCard.dataset._buffsTempList = JSON.stringify(list);
    }

    // 無敵は付与元が invincible 特性を持つ場合のみ付与（options.actorHasInvincible）
    if (options.actorHasInvincible && duration > 0) {
      targetCard.dataset._invincibleRemaining = String(Math.max(parseInt(targetCard.dataset._invincibleRemaining || '0',10), duration));
      targetCard.dataset._invincibleCooldown = String(Math.max(parseInt(targetCard.dataset._invincibleCooldown || '0',10), 2));
    }

    const specials = JSON.parse(targetCard.dataset._special || '[]');
    if (specials.includes('grantExtraAction')) delete targetCard.dataset._attacked;
    if (specials.includes('taunt') && duration>0) targetCard.dataset._tauntRemaining = String(Math.max(parseInt(targetCard.dataset._tauntRemaining || '0',10), duration));
    updateCardDisplay(targetCard);

    if (lastDetailCard && lastDetailCard.dataset && lastDetailCard.dataset.id === targetCard.dataset.id) renderDetail(targetCard);
  }

  // 判定: actorId が付与したバフ/デバフが場に残っているか（自分の場・敵の場問わず）
  function hasOwnBuffsOnField(actorId) {
    const all = Array.from(document.querySelectorAll('#player-cards .card, #enemy-cards .card'));
    for (const c of all) {
      // 一時バフをチェック
      const temp = JSON.parse(c.dataset._buffsTempList || '[]');
      if (temp.some(t => t.source === actorId && (t.remainingTurns || 0) > 0)) return true;

      // 永続バフをチェック
      const perm = JSON.parse(c.dataset._permanentBuffs || '[]');
      if (perm.some(p => p.source === actorId)) return true;

      // 永続デバフフィールドがある場合もチェック（_permanentDebuff がオブジェクトで source を持つ）
      if (c.dataset._permanentDebuff) {
        try {
          const pd = JSON.parse(c.dataset._permanentDebuff);
          if (pd && pd.source === actorId) return true;
        } catch(e) {}
      }
    }
    return false;
  }

  // 攻撃（エネ-1）: 攻撃で回復するカード（吸血）や回復専用カードに対応
  attackButton.addEventListener('click', () => {
    if (!isPlayerTurn || selectedAttacker == null) return;
    if (selectedAttacker.classList.contains('disabled')) { selectedAttacker = null; updateBattleUI(); return; }
    if (selectedAttacker.dataset._attacked === 'true') return;
    const canAttack = selectedAttacker.dataset.canAttack !== 'false' && (selectedAttacker.dataset.atk && parseInt(selectedAttacker.dataset.atk||'0',10) > 0);
    if (!canAttack) {
      // 回復専用カード：簡易実装として自分を回復
      const healVal = parseInt(selectedAttacker.dataset.heal || '0', 10);
      if (healVal > 0) {
        applyHealTo(selectedAttacker, healVal);
        selectedAttacker.dataset._attacked = 'true';
        updateCardDisplay(selectedAttacker);
        if (lastDetailCard && lastDetailCard.dataset && lastDetailCard.dataset.id === selectedAttacker.dataset.id) renderDetail(selectedAttacker);
      }
      sanitizeSelectedRefs();
      updateBattleUI();
      checkGameEnd();
      return;
    }

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
      document.querySelectorAll(selector).forEach(t => {
        const dmg = applyDamage(t, effAtk, { ignoreDef: ignoreDefAttacker });
        const lifeStealRatio = parseFloat(selectedAttacker.dataset.lifeStealRatio || '0');
        if (lifeStealRatio > 0 && dmg > 0) {
          const healFrom = Math.floor(dmg * lifeStealRatio);
          applyHealTo(selectedAttacker, healFrom);
        }
      });
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
      if (target) {
        const dmg = applyDamage(target, effAtk, { ignoreDef: ignoreDefAttacker });
        const healVal = parseInt(selectedAttacker.dataset.heal || '0', 10);
        if (healVal > 0) applyHealTo(selectedAttacker, healVal);
        const lifeStealRatio = parseFloat(selectedAttacker.dataset.lifeStealRatio || '0');
        if (lifeStealRatio > 0 && dmg > 0) {
          const healFrom = Math.floor(dmg * lifeStealRatio);
          applyHealTo(selectedAttacker, healFrom);
        }
      }
    }

    selectedAttacker.dataset._attacked = 'true';
    updateCardDisplay(selectedAttacker);
    if (lastDetailCard && lastDetailCard.dataset && lastDetailCard.dataset.id === selectedAttacker.dataset.id) renderDetail(selectedAttacker);
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

    // スキルがバフ/デバフ系なら、付与元が場に残っているかチェックしてブロック
    const skillType = selectedAttacker.dataset.skill || 'normal';
    const actorId = selectedAttacker.dataset.id;
    const isBuffSkill = (skillType === 'buff' || skillType === 'debuff' || skillType === 'heal');
    if (isBuffSkill && hasOwnBuffsOnField(actorId)) {
      // ボタン無効化の方針に合わせてここでも阻止（UI上はボタンが disabled になるが念のため）
      alert('このキャラは、以前付与したバフ/デバフが場に残っているため、再度バフ/デバフを付与できません。');
      return;
    }

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
    const actorHasInv = JSON.parse(selectedAttacker.dataset._special || '[]').includes('invincible');

    if (skill === 'heal') {
      const selector = isPlayerSide ? '#player-cards .card:not(.disabled)' : '#enemy-cards .card:not(.disabled)';
      document.querySelectorAll(selector).forEach(a => {
        const effHeal = getEffectiveStat(a, 'heal');
        const maxHp = getEffectiveStat(a, 'maxHp');
        const cur = parseInt(a.dataset.hp || '0',10);
        a.dataset.hp = String(Math.min(maxHp, cur + effHeal));
        updateCardDisplay(a);
        if (lastDetailCard && lastDetailCard.dataset && lastDetailCard.dataset.id === a.dataset.id) renderDetail(a);
      });
    } else if (skill === 'buff') {
      applyBuffAccordingToConfig(selectedAttacker, config, isPlayerSide, selectedAttacker.dataset.id);
    } else if (skill === 'debuff') {
      applyDebuffAccordingToConfig(selectedAttacker, config, isPlayerSide, selectedAttacker.dataset.id);
    } else if (skill === 'aoe') {
      const selector = isPlayerSide ? '#enemy-cards .card:not(.disabled)' : '#player-cards .card:not(.disabled)';
      document.querySelectorAll(selector).forEach(t => { applyDamage(t, getEffectiveStat(selectedAttacker, 'atk')); if (lastDetailCard && lastDetailCard.dataset && lastDetailCard.dataset.id === t.dataset.id) renderDetail(t); });
    }

    selectedAttacker.dataset._skillUsed = 'true';
    updateCardDisplay(selectedAttacker);
    if (lastDetailCard && lastDetailCard.dataset && lastDetailCard.dataset.id === selectedAttacker.dataset.id) renderDetail(selectedAttacker);
    sanitizeSelectedRefs();
    updateBattleUI();
    checkGameEnd();
  });

  function applyBuffAccordingToConfig(actorCard, config, isPlayerSide, actorId) {
    const actorHasInv = JSON.parse(actorCard.dataset._special || '[]').includes('invincible');
    if (config.target === 'all') {
      const selector = isPlayerSide ? '#player-cards .card:not(.disabled)' : '#enemy-cards .card:not(.disabled)';
      document.querySelectorAll(selector).forEach(t => applyBuffToCard(t, config.params, config.duration, config.permanent, { permanentDebuffOverwrite: config.permanentDebuffOverwrite, increaseMaxEnergy: extractIncreaseMaxEnergy(actorCard), actorHasInvincible: actorHasInv, sourceId: actorId }));
      maybeRepeatBuff(actorCard, config, () => applyBuffAccordingToConfig(actorCard, config, isPlayerSide, actorId));
    } else if (config.target === 'enemyAll') {
      const selector = isPlayerSide ? '#enemy-cards .card:not(.disabled)' : '#player-cards .card:not(.disabled)';
      document.querySelectorAll(selector).forEach(t => applyBuffToCard(t, config.params, config.duration, config.permanent, { permanentDebuffOverwrite: config.permanentDebuffOverwrite, actorHasInvincible: actorHasInv, sourceId: actorId }));
      maybeRepeatBuff(actorCard, config, () => applyBuffAccordingToConfig(actorCard, config, isPlayerSide, actorId));
    } else if (config.target === 'target') {
      let targetCard = null;
      if (selectedTarget && selectedTarget.closest && ((isPlayerSide && selectedTarget.closest('#enemy-cards')) || (!isPlayerSide && selectedTarget.closest('#player-cards')))) targetCard = selectedTarget;
      else {
        const selector = isPlayerSide ? '#player-cards .card:not(.disabled)' : '#enemy-cards .card:not(.disabled)';
        const list = Array.from(document.querySelectorAll(selector));
        if (list.length>0) targetCard = list[Math.floor(Math.random()*list.length)];
      }
      if (targetCard) {
        applyBuffToCard(targetCard, config.params, config.duration, config.permanent, { permanentDebuffOverwrite: config.permanentDebuffOverwrite, actorHasInvincible: actorHasInv, sourceId: actorId });
        maybeRepeatBuff(actorCard, config, () => applyBuffToCard(targetCard, config.params, config.duration, config.permanent, { permanentDebuffOverwrite: config.permanentDebuffOverwrite, actorHasInvincible: actorHasInv, sourceId: actorId }));
      }
    } else {
      applyBuffToCard(actorCard, config.params, config.duration, config.permanent, { permanentDebuffOverwrite: config.permanentDebuffOverwrite, increaseMaxEnergy: extractIncreaseMaxEnergy(actorCard), actorHasInvincible: actorHasInv, sourceId: actorId });
      maybeRepeatBuff(actorCard, config, () => applyBuffToCard(actorCard, config.params, config.duration, config.permanent, { actorHasInvincible: actorHasInv, sourceId: actorId }));
    }
  }

  function applyDebuffAccordingToConfig(actorCard, config, isPlayerSide, actorId) {
    const actorHasInv = JSON.parse(actorCard.dataset._special || '[]').includes('invincible');
    if (config.target === 'enemyAll' || config.target === 'all') {
      const selector = isPlayerSide ? '#enemy-cards .card:not(.disabled)' : '#player-cards .card:not(.disabled)';
      document.querySelectorAll(selector).forEach(t => applyBuffToCard(t, config.params, config.duration, config.permanent, { permanentDebuffOverwrite: config.permanentDebuffOverwrite, actorHasInvincible: actorHasInv, sourceId: actorId }));
      maybeRepeatBuff(actorCard, config, () => applyDebuffAccordingToConfig(actorCard, config, isPlayerSide, actorId));
    } else if (config.target === 'target') {
      let targetCard = null;
      if (selectedTarget && selectedTarget.closest && ((isPlayerSide && selectedTarget.closest('#enemy-cards')) || (!isPlayerSide && selectedTarget.closest('#player-cards')))) targetCard = selectedTarget;
      else {
        const selector = isPlayerSide ? '#enemy-cards .card:not(.disabled)' : '#player-cards .card:not(.disabled)';
        const list = Array.from(document.querySelectorAll(selector));
        if (list.length>0) targetCard = list[Math.floor(Math.random()*list.length)];
      }
      if (targetCard) {
        applyBuffToCard(targetCard, config.params, config.duration, config.permanent, { permanentDebuffOverwrite: config.permanentDebuffOverwrite, actorHasInvincible: actorHasInv, sourceId: actorId });
        maybeRepeatBuff(actorCard, config, () => applyBuffToCard(targetCard, config.params, config.duration, config.permanent, { actorHasInvincible: actorHasInv, sourceId: actorId }));
      }
    } else {
      const selector = isPlayerSide ? '#enemy-cards .card:not(.disabled)' : '#player-cards .card:not(.disabled)';
      const list = Array.from(document.querySelectorAll(selector));
      if (list.length>0) {
        const target = list[Math.floor(Math.random() * list.length)];
        applyBuffToCard(target, config.params, config.duration, config.permanent, { permanentDebuffOverwrite: config.permanentDebuffOverwrite, actorHasInvincible: actorHasInv, sourceId: actorId });
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

  // ---------- 敵AI ----------
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
      const enemyHasInv = JSON.parse(enemy.dataset._special || '[]').includes('invincible');

      if (skill !== 'normal' && enemy.dataset._skillUsed !== 'true') {
        if (parseInt(enemy.dataset._invincibleCooldown || '0', 10) > 0 && enemyHasInv) {
          // 無敵クールダウン中は無敵付与を控える（AI簡易処理）
        } else {
          const hp = parseInt(enemy.dataset.hp || '0', 10);
          const effMax = getEffectiveStat(enemy,'maxHp');
          if (skill === 'heal' && hp < effMax * 0.6) {
            document.querySelectorAll('#enemy-cards .card:not(.disabled)').forEach(a => {
              const effHeal = getEffectiveStat(a, 'heal');
              const maxHp = getEffectiveStat(a, 'maxHp');
              a.dataset.hp = String(Math.min(maxHp, parseInt(a.dataset.hp||'0',10) + effHeal));
              updateCardDisplay(a);
              if (lastDetailCard && lastDetailCard.dataset && lastDetailCard.dataset.id === a.dataset.id) renderDetail(a);
            });
            enemy.dataset._skillUsed = 'true';
            return;
          }
          if (skill === 'aoe') {
            document.querySelectorAll('#player-cards .card:not(.disabled)').forEach(t => applyDamage(t, getEffectiveStat(enemy, 'atk')));
            enemy.dataset._skillUsed = 'true';
            enemy.dataset._attacked = 'true';
            updateCardDisplay(enemy);
            if (lastDetailCard && lastDetailCard.dataset && lastDetailCard.dataset.id === enemy.dataset.id) renderDetail(enemy);
            return;
          }
          if (Math.random() < 0.35) {
            if (skill === 'buff' && cfg) {
              if (cfg.target === 'all') document.querySelectorAll('#enemy-cards .card:not(.disabled)').forEach(a => applyBuffToCard(a, cfg.params, cfg.duration, cfg.permanent, { actorHasInvincible: enemyHasInv, sourceId: enemy.dataset.id }));
              else if (cfg.target === 'enemyAll') document.querySelectorAll('#player-cards .card:not(.disabled)').forEach(a => applyBuffToCard(a, cfg.params, cfg.duration, cfg.permanent, { actorHasInvincible: enemyHasInv, sourceId: enemy.dataset.id }));
              else if (cfg.target === 'target') {
                const target = players[Math.floor(Math.random() * players.length)];
                if (target) applyBuffToCard(target, cfg.params, cfg.duration, cfg.permanent, { actorHasInvincible: enemyHasInv, sourceId: enemy.dataset.id });
              } else applyBuffToCard(enemy, cfg.params, cfg.duration, cfg.permanent, { actorHasInvincible: enemyHasInv, sourceId: enemy.dataset.id });
              enemy.dataset._skillUsed = 'true';
              updateCardDisplay(enemy);
              if (lastDetailCard && lastDetailCard.dataset && lastDetailCard.dataset.id === enemy.dataset.id) renderDetail(enemy);
              return;
            }
            if (skill === 'debuff' && cfg) {
              if (cfg.target === 'enemyAll' || cfg.target === 'all') document.querySelectorAll('#player-cards .card:not(.disabled)').forEach(t => applyBuffToCard(t, cfg.params, cfg.duration, cfg.permanent, { actorHasInvincible: enemyHasInv, sourceId: enemy.dataset.id }));
              else if (cfg.target === 'target') {
                const target = players[Math.floor(Math.random() * players.length)];
                if (target) applyBuffToCard(target, cfg.params, cfg.duration, cfg.permanent, { actorHasInvincible: enemyHasInv, sourceId: enemy.dataset.id });
              } else {
                const target = players[Math.floor(Math.random() * players.length)];
                if (target) applyBuffToCard(target, cfg.params, cfg.duration, cfg.permanent, { actorHasInvincible: enemyHasInv, sourceId: enemy.dataset.id });
              }
              enemy.dataset._skillUsed = 'true';
              enemy.dataset._attacked = 'true';
              return;
            }
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
      if (lastDetailCard && lastDetailCard.dataset && lastDetailCard.dataset.id === enemy.dataset.id) renderDetail(enemy);
    });
  }

  // ---------- ターン管理 ----------
  function endPlayerTurn() {
    decrementTempBuffsAndClear();
    isPlayerTurn = false;
    updateBattleUI();

    setTimeout(() => {
      enemyTurn();
      isPlayerTurn = true;
      regenEnergyPerTurn();
      resetAllTurnFlags();
      updateBattleUI();
      if (lastDetailCard) {
        const id = lastDetailCard.dataset && lastDetailCard.dataset.id;
        if (id) {
          const found = document.querySelector(`[data-id="${id}"]`);
          if (found) renderDetail(found);
          else renderDetail(null);
        } else renderDetail(null);
      }
      checkGameEnd();
    }, 600);
  }
  endTurnButton.addEventListener('click', () => endPlayerTurn());

  // ---------- エネルギー回復（doubleenergyRegen 特性対応） ----------
  function hasDoubleEnergyRegenOnField() {
    const cards = Array.from(document.querySelectorAll('#player-cards .card:not(.disabled)'));
    for (const c of cards) {
      const specials = JSON.parse(c.dataset._special || '[]');
      if (specials.includes('doubleenergyRegen')) return true;
    }
    return false;
  }

  function computeEnergyRegenMultiplierFromField() {
    return hasDoubleEnergyRegenOnField() ? 3 : 1;
  }

  function regenEnergyPerTurn() {
    const fieldMultiplier = computeEnergyRegenMultiplierFromField();
    const rawRegen = Math.round(BASE_ENERGY_REGEN * fieldMultiplier);
    const regenAmount = Math.min(rawRegen, PER_TURN_REGEN_CAP);
    energy = Math.min(playerMaxEnergy, energy + regenAmount);
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
      if (card.dataset._invincibleCooldown) {
        let cd = Math.max(0, parseInt(card.dataset._invincibleCooldown || '0', 10) - 1);
        if (cd <= 0) delete card.dataset._invincibleCooldown; else card.dataset._invincibleCooldown = String(cd);
      }

      delete card.dataset._skillUsed;
      updateCardDisplay(card);
    });

    if (lastDetailCard) {
      const id = lastDetailCard.dataset && lastDetailCard.dataset.id;
      if (id) {
        const found = document.querySelector(`[data-id="${id}"]`);
        if (found) renderDetail(found);
        else renderDetail(null);
      } else renderDetail(null);
    }
  }

  function resetAllTurnFlags() {
    document.querySelectorAll('#player-cards .card, #enemy-cards .card').forEach(card => {
      delete card.dataset._attacked;
      delete card.dataset._skillUsed;
      updateCardDisplay(card);
    });
    if (lastDetailCard) {
      const id = lastDetailCard.dataset && lastDetailCard.dataset.id;
      if (id) {
        const found = document.querySelector(`[data-id="${id}"]`);
        if (found) renderDetail(found);
        else renderDetail(null);
      } else renderDetail(null);
    }
  }

  function sanitizeSelectedRefs() {
    if (selectedAttacker && selectedAttacker.classList.contains('disabled')) selectedAttacker = null;
    if (selectedTarget && selectedTarget.classList.contains('disabled')) selectedTarget = null;
  }

  function updateEnergyUI() {
    energyCountEl.innerText = String(energy);
    energyMaxEl.innerText = String(playerMaxEnergy);
    const fieldMultiplier = computeEnergyRegenMultiplierFromField();
    const rawRegen = Math.round(BASE_ENERGY_REGEN * fieldMultiplier);
    const shownRegen = Math.min(rawRegen, PER_TURN_REGEN_CAP);
    energyRegenEl.innerText = `${shownRegen}/${PER_TURN_REGEN_CAP}`;
  }

  function updateBattleUI() {
    document.getElementById('turn-display').innerText = isPlayerTurn ? 'プレイヤー' : '敵';
    updateEnergyUI();

    // スキルボタンの無効化: バフ/デバフ系スキルは、付与元が場に残っている場合無効化
    let skillAvailable = false;
    if (isPlayerTurn && selectedAttacker && !selectedAttacker.classList.contains('disabled')) {
      const cost = parseInt(selectedAttacker.dataset.cost || '0',10);
      const skillType = selectedAttacker.dataset.skill || 'normal';
      const actorId = selectedAttacker.dataset.id;
      const isBuffSkill = (skillType === 'buff' || skillType === 'debuff' || skillType === 'heal');
      const blockedByOwnBuff = isBuffSkill && hasOwnBuffsOnField(actorId);
      skillAvailable = (!blockedByOwnBuff) && (selectedAttacker.dataset._skillUsed !== 'true') && (energy >= cost);
    }

    const invCooldownActive = selectedAttacker && parseInt(selectedAttacker.dataset._invincibleCooldown || '0', 10) > 0 && JSON.parse(selectedAttacker.dataset._special || '[]').includes('invincible');
    attackButton.disabled = !(isPlayerTurn && selectedAttacker && !selectedAttacker.classList.contains('disabled') && selectedAttacker.dataset._attacked !== 'true' && (selectedAttacker.dataset.canAttack !== 'false' ? energy >= 1 : true));
    skillButton.disabled = !(skillAvailable && !invCooldownActive);
    endTurnButton.disabled = false;

    if (selectedAttacker && selectedAttacker.classList.contains('disabled')) selectedAttacker = null;
    if (selectedTarget && selectedTarget.classList.contains('disabled')) selectedTarget = null;
    document.querySelectorAll('.card').forEach(c => c.classList.remove('attacker-selected', 'target-selected'));
    if (selectedAttacker) selectedAttacker.classList.add('attacker-selected');
    if (selectedTarget) selectedTarget.classList.add('target-selected');

    if (lastDetailCard) {
      const id = lastDetailCard.dataset && lastDetailCard.dataset.id;
      if (id) {
        const found = document.querySelector(`[data-id="${id}"]`);
        if (found) renderDetail(found);
        else renderDetail(null);
      } else renderDetail(null);
    }
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
    energy = 10; playerMaxEnergy = 10; isPlayerTurn = true; selectedAttacker = null; selectedTarget = null;
    document.querySelectorAll('.card-pool .card').forEach(c => c.classList.remove('selected','attacker-selected','target-selected'));
    selectedCards = [];
    updateSelectionUI();
    updateBattleUI();
    detailEl.style.display='none'; detailEmpty.style.display='block';
    selectionDetail.style.display='none';
    lastDetailCard = null;
    buildCardPoolDOM();
  }

  // 初期 UI
  updateEnergyUI();
  updateSelectionUI();
  updateBattleUI();
});
</script>
</body>
</html>
