<template>
  <div class="team-analyzer">
    <div class="header">
      <h2>⚔️ 洛克王国 · 双属性联防模拟器 ⚔️</h2>
      <p class="subtitle">支持双属性精灵，精确计算属性克制与联防</p>
    </div>

    <!-- 队伍管理 -->
    <div class="team-section card">
      <div class="section-header">
        <h3>✨ 我的精灵队伍 ✨</h3>
        <button class="btn-add" @click="addPokemon">+ 添加精灵</button>
      </div>
      <div class="pokemon-grid">
        <div v-for="(pokemon, idx) in team" :key="pokemon.id" class="pokemon-card">
          <div class="card-header">
            <span class="pokemon-index">#{{ idx + 1 }}</span>
            <button class="btn-delete" @click="deletePokemon(pokemon.id)">✖</button>
          </div>
          <input v-model="pokemon.name" placeholder="精灵名称" class="pokemon-name-input" />
          <div class="attribute-field">
            <label>主属性</label>
            <select v-model="pokemon.primaryAttr">
              <option v-for="attr in attributes" :key="attr" :value="attr">{{ attr }}</option>
            </select>
          </div>
          <div class="attribute-field">
            <label>次属性 (可选)</label>
            <select v-model="pokemon.secondaryAttr">
              <option value="">无</option>
              <option v-for="attr in attributes" :key="attr" :value="attr">{{ attr }}</option>
            </select>
          </div>
          <div class="defense-info">
            <div class="weakness" v-if="getFinalWeaknesses(pokemon).length">
              <span class="info-label">⚠️ 被克制 (最终倍率 > 1)</span>
              <div class="type-badges">
                <span v-for="w in getFinalWeaknesses(pokemon)" :key="w.attr" class="type-badge weak">
                  {{ w.attr }} <span v-if="w.multiplier === 4">(4x)</span><span v-else>({{ w.multiplier }}x)</span>
                </span>
              </div>
            </div>
            <div class="resistance" v-if="getFinalResistances(pokemon).length">
              <span class="info-label">🛡️ 抵抗 (最终倍率 < 1)</span>
              <div class="type-badges">
                <span v-for="r in getFinalResistances(pokemon)" :key="r.attr" class="type-badge resist">
                  {{ r.attr }} <span v-if="r.multiplier === 0.25">(0.25x)</span><span v-else>({{ r.multiplier }}x)</span>
                </span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 联防推荐表 -->
    <div class="synergy-section card" v-if="team.length">
      <div class="section-header">
        <h3>🔄 联防推荐 (针对每个弱点属性)</h3>
      </div>
      <div class="synergy-grid">
        <div v-for="attr in attributes" :key="attr" class="synergy-item">
          <div class="attack-attr">{{ attr }}</div>
          <div class="defenders">
            <span v-if="getSynergyPokemons(attr).length === 0" class="no-defense">⚠️ 无人抵抗</span>
            <span v-for="p in getSynergyPokemons(attr)" :key="p.id" class="defender-badge">
              {{ p.name || '未命名' }} <span class="defender-attr">({{ p.primaryAttr }}{{ p.secondaryAttr ? '+' + p.secondaryAttr : '' }})</span>
            </span>
          </div>
        </div>
      </div>
    </div>

    <!-- 威胁模拟器 -->
    <div class="simulator-section card">
      <div class="section-header">
        <h3>🎯 模拟对方技能属性</h3>
      </div>
      <div class="sim-controls">
        <select v-model="selectedAttackAttr">
          <option v-for="attr in attributes" :key="attr" :value="attr">{{ attr }}</option>
        </select>
        <button class="btn-simulate" @click="simulateAttack">分析谁上场最安全</button>
      </div>
      <div v-if="simulationResult.length" class="sim-result">
        <div v-for="res in simulationResult" :key="res.pokemon.id" class="result-row">
          <span class="result-poke">{{ res.pokemon.name || '无名' }} ({{ res.pokemon.primaryAttr }}{{ res.pokemon.secondaryAttr ? '+' + res.pokemon.secondaryAttr : '' }})</span>
          <span class="result-multiplier" :class="multiplierClass(res.multiplier)">
            伤害倍率: {{ res.multiplier }}x
          </span>
          <span class="result-tag">
            <span v-if="res.multiplier < 1" class="tag-good">✅ 推荐上场</span>
            <span v-else-if="res.multiplier > 1" class="tag-bad">❌ 被克制</span>
            <span v-else class="tag-normal">⚖️ 正常</span>
          </span>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

interface Pokemon {
  id: number
  name: string
  primaryAttr: string
  secondaryAttr: string   // 空字符串表示无次属性
}

const attributes = [
  '普通', '草', '火', '水', '光', '地', '冰', '龙', '电',
  '毒', '虫', '武', '翼', '萌', '幽', '恶', '机械', '幻'
] as const

// 属性克制表 (防御方索引，攻击方索引)
const typeChart: number[][] = [
  [1,1,1,1,1,1,1,1,1,1,1,2,1,1,0.5,1,1,1],
  [1,1,2,0.5,0.5,0.5,2,1,0.5,2,2,1,2,1,1,1,1,1],
  [1,0.5,1,2,1,2,0.5,1,1,1,0.5,1,1,0.5,1,1,0.5,1],
  [1,2,0.5,1,1,1,1,1,2,1,1,1,1,1,1,1,0.5,1],
  [1,2,1,1,1,1,1,1,1,1,1,1,1,1,2,0.5,1,0.5],
  [0.5,2,0.5,2,1,1,2,1,0.5,0.5,1,2,0.5,1,1,1,2,1],
  [1,1,2,0.5,0.5,2,0.5,1,1,1,1,1,2,1,1,1,2,1],
  [1,0.5,0.5,0.5,1,1,2,2,0.5,1,1,1,0.5,2,1,1,1,1],
  [1,1,1,1,1,2,1,2,0.5,1,1,1,0.5,1,1,1,0.5,1],
  [1,0.5,1,1,1,2,1,1,1,1,0.5,0.5,0.5,1,0.5,1,2,2],
  [1,0.5,2,1,1,1,1,1,1,1,1,1,0.5,2,1,1,1,1],
  [1,1,1,1,1,0.5,1,1,1,1,1,0.5,1,2,2,1,0.5,2],
  [1,0.5,1,1,1,1,2,1,2,1,0.5,0.5,1,1,1,1,1,1],
  [1,1,1,1,1,1,1,1,1,2,0.5,0.5,1,1,1,1,2,2],
  [0.5,1,1,1,2,1,1,1,1,0.5,0.5,0.5,1,1,2,2,1,1],
  [1,1,1,1,2,1,1,1,1,1,2,2,1,2,0.5,0.5,1,1],
  [0.5,0.5,2,2,1,1,0.5,0.5,1,0.5,0.5,2,0.5,0.5,1,1,0.5,0.5],
  [1,1,1,1,1,1,1,1,1,1,2,0.5,1,1,2,1,1,0.5]
]

const getDefIndex = (attr: string) => attributes.indexOf(attr as any)
const getAtkIndex = (attr: string) => attributes.indexOf(attr as any)

// 单属性伤害倍率
const getSingleDamageMultiplier = (defAttr: string, atkAttr: string): number => {
  const defIdx = getDefIndex(defAttr)
  const atkIdx = getAtkIndex(atkAttr)
  if (defIdx === -1 || atkIdx === -1) return 1
  return typeChart[defIdx][atkIdx]
}

// 双属性伤害倍率（两个属性相乘）
const getDamageMultiplier = (pokemon: Pokemon, atkAttr: string): number => {
  let mult = getSingleDamageMultiplier(pokemon.primaryAttr, atkAttr)
  if (pokemon.secondaryAttr) {
    mult *= getSingleDamageMultiplier(pokemon.secondaryAttr, atkAttr)
  }
  return mult
}

// 获取最终被克制的属性列表（倍率 > 1），带倍率值
const getFinalWeaknesses = (pokemon: Pokemon): { attr: string, multiplier: number }[] => {
  const result: { attr: string, multiplier: number }[] = []
  for (const atk of attributes) {
    const mult = getDamageMultiplier(pokemon, atk)
    if (mult > 1) {
      result.push({ attr: atk, multiplier: mult })
    }
  }
  // 按倍率从高到低排序
  result.sort((a, b) => b.multiplier - a.multiplier)
  return result
}

// 获取最终抵抗的属性列表（倍率 < 1）
const getFinalResistances = (pokemon: Pokemon): { attr: string, multiplier: number }[] => {
  const result: { attr: string, multiplier: number }[] = []
  for (const atk of attributes) {
    const mult = getDamageMultiplier(pokemon, atk)
    if (mult < 1) {
      result.push({ attr: atk, multiplier: mult })
    }
  }
  result.sort((a, b) => a.multiplier - b.multiplier)
  return result
}

// 默认6只精灵（部分双属性示例）
const defaultTeam: Pokemon[] = [
  { id: 1, name: '燃薪虫', primaryAttr: '火', secondaryAttr: '' },
  { id: 2, name: '水灵', primaryAttr: '水', secondaryAttr: '' },
  { id: 3, name: '小鹿', primaryAttr: '草', secondaryAttr: '' },
  { id: 4, name: '迪莫', primaryAttr: '光', secondaryAttr: '' },
  { id: 5, name: '贝古斯', primaryAttr: '机械', secondaryAttr: '火' },
  { id: 6, name: '海豹船长', primaryAttr: '水', secondaryAttr: '武' }
]

const team = ref<Pokemon[]>([...defaultTeam])
let nextId = 7

const addPokemon = () => {
  team.value.push({
    id: nextId++,
    name: `新精灵${nextId-1}`,
    primaryAttr: '普通',
    secondaryAttr: ''
  })
}

const deletePokemon = (id: number) => {
  team.value = team.value.filter(p => p.id !== id)
}

// 联防推荐：针对某个攻击属性，找出队伍中抵抗它（最终倍率<1）的精灵
const getSynergyPokemons = (attackAttr: string): Pokemon[] => {
  return team.value.filter(p => getDamageMultiplier(p, attackAttr) < 1)
}

// 威胁模拟器
const selectedAttackAttr = ref<string>('火')
const simulationResult = ref<{ pokemon: Pokemon, multiplier: number }[]>([])

const simulateAttack = () => {
  const attack = selectedAttackAttr.value
  const results = team.value.map(pokemon => ({
    pokemon,
    multiplier: getDamageMultiplier(pokemon, attack)
  }))
  results.sort((a, b) => a.multiplier - b.multiplier)
  simulationResult.value = results
}

const multiplierClass = (mult: number) => {
  if (mult > 1) return 'mult-bad'
  if (mult < 1) return 'mult-good'
  return 'mult-normal'
}
</script>

<style scoped>
/* 样式与之前相同，未做改动，保持美观 */
.team-analyzer {
  max-width: 1300px;
  margin: 0 auto;
  padding: 24px 20px;
  font-family: system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', sans-serif;
  background: transparent;
}
.header {
  text-align: center;
  margin-bottom: 32px;
}
.header h2 {
  font-size: 1.8rem;
  background: linear-gradient(135deg, #d45a2b, #f09819);
  background-clip: text;
  -webkit-background-clip: text;
  color: transparent;
  margin-bottom: 8px;
  letter-spacing: 1px;
}
.subtitle {
  color: #4a5568;
  font-size: 0.95rem;
}
.card {
  background: rgba(255, 255, 255, 0.92);
  backdrop-filter: blur(2px);
  border-radius: 32px;
  padding: 24px;
  margin-bottom: 28px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.08);
  transition: box-shadow 0.2s;
}
.section-header {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  margin-bottom: 20px;
  flex-wrap: wrap;
  gap: 12px;
}
.section-header h3 {
  font-size: 1.4rem;
  font-weight: 600;
  color: #2d3748;
  margin: 0;
}
.btn-add {
  background: #38a169;
  color: white;
  border: none;
  padding: 8px 20px;
  border-radius: 40px;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.2s;
}
.btn-add:hover {
  background: #2f855a;
}
.pokemon-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 24px;
  justify-content: flex-start;
}
.pokemon-card {
  background: #fffef7;
  border-radius: 24px;
  padding: 18px;
  width: 280px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
  transition: transform 0.1s, box-shadow 0.2s;
  border: 1px solid #f0e6d2;
}
.pokemon-card:hover {
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.1);
}
.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}
.pokemon-index {
  font-size: 0.85rem;
  font-weight: 600;
  color: #a0aec0;
  background: #edf2f7;
  padding: 2px 10px;
  border-radius: 30px;
}
.btn-delete {
  background: none;
  border: none;
  color: #fc8181;
  font-size: 1.2rem;
  cursor: pointer;
  width: 28px;
  height: 28px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.2s;
}
.btn-delete:hover {
  background: #fff5f5;
  color: #e53e3e;
}
.pokemon-name-input {
  width: 100%;
  padding: 10px 12px;
  border: 1px solid #e2e8f0;
  border-radius: 20px;
  font-size: 1rem;
  margin-bottom: 16px;
  box-sizing: border-box;
  background: #ffffff;
  transition: border 0.2s;
}
.pokemon-name-input:focus {
  outline: none;
  border-color: #f6ad55;
}
.attribute-field {
  margin-bottom: 14px;
}
.attribute-field label {
  display: block;
  font-size: 0.8rem;
  font-weight: 600;
  color: #4a5568;
  margin-bottom: 6px;
}
.attribute-field select {
  width: 100%;
  padding: 8px 12px;
  border-radius: 30px;
  border: 1px solid #e2e8f0;
  background: white;
  font-size: 0.9rem;
}
.defense-info {
  font-size: 0.8rem;
  margin-top: 12px;
}
.weakness, .resistance {
  margin-top: 12px;
}
.info-label {
  font-weight: 600;
  display: block;
  margin-bottom: 6px;
  font-size: 0.75rem;
  color: #4a5568;
}
.type-badges {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}
.type-badge {
  padding: 4px 10px;
  border-radius: 30px;
  font-size: 0.7rem;
  font-weight: 600;
}
.type-badge.weak {
  background: #fed7d7;
  color: #c53030;
}
.type-badge.resist {
  background: #c6f6d5;
  color: #276749;
}
.synergy-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
  gap: 12px;
  max-height: 500px;
  overflow-y: auto;
  padding-right: 6px;
}
.synergy-item {
  background: #f7fafc;
  border-radius: 20px;
  padding: 12px 16px;
  display: flex;
  align-items: center;
  gap: 14px;
  transition: background 0.2s;
}
.attack-attr {
  font-weight: 700;
  background: #fbbf24;
  color: #78350f;
  padding: 6px 14px;
  border-radius: 40px;
  min-width: 80px;
  text-align: center;
  font-size: 0.9rem;
}
.defenders {
  flex: 1;
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}
.defender-badge {
  background: #e2e8f0;
  padding: 4px 12px;
  border-radius: 30px;
  font-size: 0.8rem;
  font-weight: 500;
  color: #2d3748;
}
.defender-attr {
  font-size: 0.7rem;
  color: #718096;
}
.no-defense {
  color: #e53e3e;
  font-style: italic;
  font-size: 0.8rem;
}
.sim-controls {
  display: flex;
  gap: 16px;
  align-items: center;
  flex-wrap: wrap;
  margin-bottom: 24px;
}
.sim-controls select {
  padding: 10px 18px;
  border-radius: 40px;
  border: 1px solid #cbd5e0;
  font-size: 0.95rem;
  background: white;
}
.btn-simulate {
  background: #ed8936;
  border: none;
  padding: 10px 24px;
  border-radius: 40px;
  font-weight: 600;
  color: white;
  cursor: pointer;
  transition: background 0.2s;
}
.btn-simulate:hover {
  background: #dd6b20;
}
.sim-result {
  display: flex;
  flex-direction: column;
  gap: 12px;
}
.result-row {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 20px;
  padding: 12px 16px;
  background: #ffffff;
  border-radius: 20px;
  border: 1px solid #e9ecef;
}
.result-poke {
  font-weight: 600;
  width: 160px;
  font-size: 0.95rem;
}
.result-multiplier {
  font-weight: 700;
  width: 110px;
}
.mult-good {
  color: #2f855a;
}
.mult-bad {
  color: #c53030;
}
.mult-normal {
  color: #dd6b20;
}
.result-tag {
  font-size: 0.85rem;
}
.tag-good {
  background: #c6f6d5;
  color: #276749;
  padding: 2px 12px;
  border-radius: 30px;
}
.tag-bad {
  background: #fed7d7;
  color: #c53030;
  padding: 2px 12px;
  border-radius: 30px;
}
.tag-normal {
  background: #feebc8;
  color: #c05621;
  padding: 2px 12px;
  border-radius: 30px;
}
@media (max-width: 700px) {
  .team-analyzer {
    padding: 16px;
  }
  .pokemon-card {
    width: 100%;
  }
  .result-row {
    flex-direction: column;
    align-items: flex-start;
    gap: 8px;
  }
  .synergy-item {
    flex-direction: column;
    align-items: flex-start;
  }
  .attack-attr {
    align-self: flex-start;
  }
}
</style>