<script setup>
import { computed, reactive, ref } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { useTravelStore } from '../stores/travel'
import { TRIP_TYPES, TRANSPORTS } from '../constants'
import { daysBetween } from '../utils/format'
import ImageUpload from '../components/common/ImageUpload.vue'

const store = useTravelStore()
const route = useRoute()
const router = useRouter()

const isEdit = computed(() => route.name === 'plan-edit')
const isCopy = computed(() => route.name === 'plan-copy')
const sourcePlanId = computed(() => route.params.id)

const form = reactive({
  name: '',
  destination: '',
  tripType: '短途',
  startDate: '',
  endDate: '',
  memberCount: 1,
  memberNames: '',
  transport: '自驾',
  accommodation: '',
  budget: '',
  notes: '',
  photo: '',
})

const errors = ref({})

if (isEdit.value || isCopy.value) {
  const source = store.planById(sourcePlanId.value)
  if (source) {
    Object.assign(form, {
      name: isCopy.value ? `${source.name} 副本` : source.name,
      destination: source.destination,
      tripType: source.tripType,
      startDate: source.startDate,
      endDate: source.endDate,
      memberCount: source.memberCount,
      memberNames: source.members.map((m) => m.name).join('、'),
      transport: source.transport,
      accommodation: source.accommodation,
      budget: source.budget || '',
      notes: source.notes,
      photo: source.photo || '',
    })
  } else {
    router.replace('/plans')
  }
}

const days = computed(() => daysBetween(form.startDate, form.endDate))

function validate() {
  errors.value = {}
  if (!form.name.trim()) errors.value.name = '请输入出行名称'
  if (!form.destination.trim()) errors.value.destination = '请输入目的地'
  if (!form.startDate) errors.value.startDate = '请选择出发日期'
  if (!form.endDate) errors.value.endDate = '请选择返回日期'
  if (form.startDate && form.endDate && days.value <= 0) errors.value.endDate = '返回日期需晚于出发日期'
  return Object.keys(errors.value).length === 0
}

function submit() {
  if (!validate()) return
  const memberNames = form.memberNames.split(/[、,，\s]+/).filter(Boolean)
  const payload = {
    name: form.name.trim(),
    destination: form.destination.trim(),
    tripType: form.tripType,
    startDate: form.startDate,
    endDate: form.endDate,
    memberCount: Number(form.memberCount) || 1,
    memberNames,
    transport: form.transport,
    accommodation: form.accommodation,
    budget: form.budget,
    notes: form.notes,
    photo: form.photo,
  }

  if (isEdit.value) {
    store.updatePlan(sourcePlanId.value, payload)
    router.push(`/plans/${sourcePlanId.value}`)
  } else if (isCopy.value) {
    const id = store.duplicatePlan(sourcePlanId.value, payload)
    if (id) router.push(`/plans/${id}`)
  } else {
    const id = store.createPlan(payload)
    router.push(`/plans/${id}`)
  }
}
</script>

<template>
  <div class="card" style="max-width: 720px">
    <h2 class="card-title">{{ isEdit ? '编辑出行计划' : isCopy ? '复制出行计划' : '新建出行计划' }}</h2>

    <div v-if="isCopy" class="copy-tip">
      <p>将基于原计划生成一份新计划，请重新确认<strong>出行名称与日期</strong>。</p>
      <p class="text-muted">
        行李自定义物品、待办事项会带过来；打包勾选、行程记录与出行总结不会带入。
      </p>
    </div>

    <div class="form-row">
      <div class="form-group">
        <label class="form-label"><span class="required">*</span> 出行名称</label>
        <input v-model="form.name" class="input" placeholder="例如：杭州三日游" />
        <span v-if="errors.name" class="form-error">{{ errors.name }}</span>
      </div>
      <div class="form-group">
        <label class="form-label"><span class="required">*</span> 目的地</label>
        <input v-model="form.destination" class="input" placeholder="例如：杭州" />
        <span v-if="errors.destination" class="form-error">{{ errors.destination }}</span>
      </div>
    </div>

    <div class="form-row">
      <div class="form-group">
        <label class="form-label"><span class="required">*</span> 出行类型</label>
        <select v-model="form.tripType" class="select">
          <option v-for="t in TRIP_TYPES" :key="t" :value="t">{{ t }}</option>
        </select>
      </div>
      <div class="form-group">
        <label class="form-label">交通方式</label>
        <select v-model="form.transport" class="select">
          <option v-for="t in TRANSPORTS" :key="t" :value="t">{{ t }}</option>
        </select>
      </div>
    </div>

    <div class="form-row">
      <div class="form-group">
        <label class="form-label"><span class="required">*</span> 出发日期</label>
        <input v-model="form.startDate" type="date" class="input" />
        <span v-if="errors.startDate" class="form-error">{{ errors.startDate }}</span>
      </div>
      <div class="form-group">
        <label class="form-label"><span class="required">*</span> 返回日期</label>
        <input v-model="form.endDate" type="date" class="input" />
        <span v-if="errors.endDate" class="form-error">{{ errors.endDate }}</span>
      </div>
    </div>

    <p v-if="days > 0" class="text-secondary mb-16">行程共 {{ days }} 天</p>

    <div class="form-row">
      <div class="form-group">
        <label class="form-label">出行人数</label>
        <input v-model="form.memberCount" type="number" min="1" class="input" />
      </div>
      <div class="form-group">
        <label class="form-label">成员姓名（可选，逗号分隔）</label>
        <input v-model="form.memberNames" class="input" placeholder="例如：爸爸、妈妈、宝宝" />
      </div>
    </div>

    <div class="form-row">
      <div class="form-group">
        <label class="form-label">住宿信息</label>
        <input v-model="form.accommodation" class="input" placeholder="例如：西湖附近酒店" />
      </div>
      <div class="form-group">
        <label class="form-label">预算总额（元）</label>
        <input v-model="form.budget" type="number" min="0" class="input" placeholder="0" />
      </div>
    </div>

    <div class="form-group">
      <label class="form-label">备注</label>
      <textarea v-model="form.notes" class="textarea" placeholder="补充说明"></textarea>
    </div>

    <div class="form-group">
      <ImageUpload v-model="form.photo" label="目的地照片（可选）" />
    </div>

    <div class="flex gap-8">
      <button class="btn btn-primary" @click="submit">
        {{ isEdit ? '保存修改' : isCopy ? '创建副本' : '保存' }}
      </button>
      <button class="btn btn-ghost" @click="router.back()">取消</button>
    </div>
  </div>
</template>

<style scoped>
.form-error {
  color: var(--danger);
  font-size: 12px;
}

.copy-tip {
  background: var(--primary-light);
  border-radius: var(--radius-sm);
  padding: 12px 14px;
  margin-bottom: 16px;
  font-size: 13px;
  line-height: 1.7;
}

.copy-tip p {
  margin: 0;
}
</style>
