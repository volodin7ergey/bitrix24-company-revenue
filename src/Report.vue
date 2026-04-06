<template>
	<!-- Загрузка приложения -->
	<div class="app-start-box" v-if="isAppLoading">
		<v-progress-circular indeterminate />
	</div>

	<template v-else>
		<div class="container">
			<div class="filters-box">
				<v-select
					label="Компания"
					v-model="selectedCompany"
					:items="companyItems"
					item-title="title"
					item-value="value"
					hint="Выберите компанию или оставьте 'Все'"
					persistent-hint
				/>

				<v-date-input
					v-model="dateFrom"
					label="Период анализа с"
					hint="Показывает сделки, закрытые с этой даты"
					persistent-hint
					autocomplete="off"
				/>

				<v-date-input
					v-model="dateTo"
					label="Период анализа по"
					hint="Показывает сделки, закрытые до этой даты"
					persistent-hint
					autocomplete="off"
				/>
			</div>

			<v-btn
				color="primary"
				@click="loadReport"
				:disabled="isLoading"
				:loading="isLoading"
			>
				Сформировать отчёт
			</v-btn>
		</div>

		<div class="content">
			<!-- Информационный alert перед формированием отчета -->
			<v-alert v-if="!reportLoaded" type="info" variant="tonal">
				Выберите нужные фильтры и нажмите «Сформировать отчёт».
			</v-alert>

			<!-- Таблица отчета -->
			<v-table v-else-if="rows.length" density="default">
				<thead>
					<tr>
						<th>
							Компания
							<v-tooltip>
								<template #activator="{ props }">
									<span v-bind="props">ℹ️</span>
								</template>
								<span
									>Название компании-клиента, по которому ведется аналитика
									сделок</span
								>
							</v-tooltip>
						</th>
						<th>
							Дата регистрации компании
							<v-tooltip>
								<template #activator="{ props }">
									<span v-bind="props">ℹ️</span>
								</template>
								<span>Дата добавления компании в CRM</span>
							</v-tooltip>
						</th>
						<th>
							Кол-во успешных сделок
							<v-tooltip>
								<template #activator="{ props }">
									<span v-bind="props">ℹ️</span>
								</template>
								<span>Сделки, закрытые с результатом успех</span>
							</v-tooltip>
						</th>
						<th>
							Сумма успешных сделок, ₽
							<v-tooltip>
								<template #activator="{ props }">
									<span v-bind="props">ℹ️</span>
								</template>
								<span>Общая сумма успешных сделок за выбранный период</span>
							</v-tooltip>
						</th>
						<th>
							Сумма потерянных сделок, ₽
							<v-tooltip>
								<template #activator="{ props }">
									<span v-bind="props">ℹ️</span>
								</template>
								<span>Сумма сделок, которые не завершились успешно</span>
							</v-tooltip>
						</th>
					</tr>
				</thead>

				<tbody>
					<tr v-for="row in rows" :key="row.companyId" class="row">
						<td class="task-title">
							<v-tooltip location="top">
								<template #activator="{ props }">
									<span
										class="company-title"
										v-bind="props"
										style="cursor: pointer"
										@click="onCompanyVisit(row.companyId)"
									>
										{{ row.title }}
									</span>
								</template>
								<span>Нажмите, чтобы открыть карточку компании</span>
							</v-tooltip>
						</td>
						<td>{{ formatDate(row.date) }}</td>
						<td>{{ row.successCount }}</td>
						<td>{{ formatMoney(row.successSum) }}</td>
						<td>{{ formatMoney(row.failSum) }}</td>
					</tr>

					<tr class="font-weight-bold bg-grey-lighten-4">
						<td>ИТОГО</td>
						<td></td>
						<td>{{ totals.successCount }}</td>
						<td>{{ formatMoney(totals.successSum) }}</td>
						<td>{{ formatMoney(totals.failSum) }}</td>
					</tr>
				</tbody>
			</v-table>

			<!-- Если компаний вообще нет -->
			<v-alert v-else type="warning" variant="tonal">
				Нет компаний для выбранного фильтра
			</v-alert>
		</div>
	</template>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

type Company = { ID: string; TITLE: string; DATE_CREATE: string }
type Deal = {
	ID: string
	COMPANY_ID: string
	OPPORTUNITY: number
	STAGE_SEMANTIC_ID: string
	CLOSEDATE: string
}
type Row = {
	companyId: string
	title: string
	date: string
	successCount: number
	successSum: number
	failSum: number
}

const isAppLoading = ref(true)
const isLoading = ref(false)
const reportLoaded = ref(false)

const companies = ref<Company[]>([])
const deals = ref<Deal[]>([])
const rows = ref<Row[]>([])

const totals = ref({ successCount: 0, successSum: 0, failSum: 0 })

const dateFrom = ref('')
const dateTo = ref('')
const selectedCompany = ref('')

// Выбор компаний для селекта
const companyItems = computed(() => [
	{ title: 'Все', value: '' },
	...companies.value.map(c => ({ title: c.TITLE, value: c.ID })),
])

// Форматирование
function formatMoney(val: number) {
	return new Intl.NumberFormat('ru-RU').format(val)
}
function formatDate(date: string) {
	return date ? new Date(date).toLocaleDateString() : ''
}

// Навигация к компании
const onCompanyVisit = (companyId: string) => {
	BX24.openPath(`/crm/company/details/${companyId}/`, () => {})
}

// Преобразование даты
function onDateChange(type: 'from' | 'to', value: string | Date) {
	if (value instanceof Date) {
		const d = value
		const formatted = `${d.getFullYear()}-${(d.getMonth() + 1)
			.toString()
			.padStart(2, '0')}-${d.getDate().toString().padStart(2, '0')}`
		if (type === 'from') dateFrom.value = formatted
		else dateTo.value = formatted
	}
}

// API
function api<T>(method: string, params: Record<string, any> = {}): Promise<T> {
	return new Promise((resolve, reject) => {
		BX24.callMethod(method, params, (result: any) => {
			if (result.error()) reject(result.error())
			else resolve(result.data())
		})
	})
}

function loadCompanies(): Promise<Company[]> {
	return new Promise(resolve => {
		BX24.callMethod(
			'crm.company.list',
			{ select: ['ID', 'TITLE', 'DATE_CREATE'] },
			(res: any) => resolve(res.data()),
		)
	})
}

function loadDeals(): Promise<Deal[]> {
	const filter: any = {}
	if (dateFrom.value) filter['>=CLOSEDATE'] = dateFrom.value + ' 00:00:00'
	if (dateTo.value) filter['<=CLOSEDATE'] = dateTo.value + ' 23:59:59'
	if (selectedCompany.value) filter['COMPANY_ID'] = selectedCompany.value

	return new Promise(resolve => {
		BX24.callMethod(
			'crm.deal.list',
			{
				select: [
					'ID',
					'COMPANY_ID',
					'OPPORTUNITY',
					'STAGE_SEMANTIC_ID',
					'CLOSEDATE',
				],
				filter,
			},
			(res: any) => resolve(res.data()),
		)
	})
}

// Построение отчета
function buildReport() {
	const map: Record<string, Row> = {}
	totals.value = { successCount: 0, successSum: 0, failSum: 0 }

	for (const company of companies.value) {
		if (selectedCompany.value && company.ID !== selectedCompany.value) continue
		map[company.ID] = {
			companyId: company.ID,
			title: company.TITLE,
			date: company.DATE_CREATE,
			successCount: 0,
			successSum: 0,
			failSum: 0,
		}
	}

	for (const deal of deals.value) {
		const row = map[deal.COMPANY_ID]
		if (!row) continue
		if (deal.STAGE_SEMANTIC_ID === 'S') {
			row.successCount++
			row.successSum += Number(deal.OPPORTUNITY)
			totals.value.successCount++
			totals.value.successSum += Number(deal.OPPORTUNITY)
		} else if (deal.STAGE_SEMANTIC_ID === 'F') {
			row.failSum += Number(deal.OPPORTUNITY)
			totals.value.failSum += Number(deal.OPPORTUNITY)
		}
	}

	rows.value = Object.values(map)
}

// Загрузка отчета
async function loadReport() {
	isLoading.value = true
	companies.value = await loadCompanies()
	deals.value = await loadDeals()
	buildReport()
	reportLoaded.value = true
	isLoading.value = false
}

// Загрузка компаний при старте
const sleep = (ms: number) => new Promise(res => setTimeout(res, ms))
onMounted(async () => {
	await sleep(500)
	try {
		companies.value = await api('crm.company.list', {
			select: ['ID', 'TITLE', 'DATE_CREATE'],
		})
	} catch (e) {
		console.log(e)
	} finally {
		isAppLoading.value = false
	}
})
</script>

<style scoped>
.app-start-box {
	width: 100%;
	height: 100dvh;
	display: grid;
	place-items: center;
}
.container {
	margin-top: 1rem;
	margin-bottom: 2rem;
}
.filters-box {
	display: flex;
	gap: 1rem;
	margin-bottom: 1rem;
}

.company-title {
	transition: all 0.2s ease;
	text-decoration: underline dotted;
	color: #1976d2;
}

.row {
	transition: 0.2s ease;
}

.row:active {
	transform: scale(0.98);
	background-color: #cfe8fc;
}

.row:hover {
	background-color: #cfe8fc;
}
</style>
