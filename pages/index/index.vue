<template>
	<view class="page-shell">
		<view class="ambient ambient-left"></view>
		<view class="ambient ambient-right"></view>

		<view class="container">
			<view class="hero-card fade-up delay-1" :style="heroThemeStyle">
				<view class="hero-gradient hero-gradient-sun"></view>
				<view class="hero-gradient hero-gradient-mint"></view>
				<view class="hero-gradient hero-gradient-rose"></view>
				<view class="hero-topline">
					<text class="hero-badge">{{ currentWeekdayEnglish }}</text>
					<text class="hero-date">{{ currentDate }}</text>
				</view>
				<view class="hero-main">
					<view class="hero-copy">
						<text class="app-title">学习打卡</text>
						<view class="quote-card" @click="refreshDailyQuote">
							<text :key="quoteAnimateKey" class="hero-subtitle quote-fade">{{ dailyQuote }}</text>
							<text class="quote-tip">{{ quoteLoading ? '正在获取新句子...' : '点这里换一句' }}</text>
							<text v-if="dailyQuoteSource" class="quote-source">{{ dailyQuoteSource }}</text>
						</view>
					</view>
				</view>
				<view class="hero-check-row">
					<button @click="checkIn" class="hero-check-btn" :class="{ 'success-pulse': checkInPulse }" :disabled="isCheckedToday">
						{{ isCheckedToday ? '今日已打卡了' : '立即打卡' }}
					</button>
				</view>
				<view class="hero-highlight">
					<view class="highlight-item compact-highlight">
						<text class="highlight-label">完成率</text>
						<text class="highlight-value">{{ completionRate }}%</text>
					</view>
					<view class="highlight-divider"></view>
					<view class="highlight-item compact-highlight">
						<text class="highlight-label">已打卡</text>
						<text class="highlight-value small">{{ checkedDays }}天</text>
					</view>
					<view class="highlight-divider"></view>
					<view class="highlight-item compact-highlight">
						<text class="highlight-label">当前聚焦</text>
						<text class="highlight-value small">{{ focusDateLabel }}</text>
					</view>
				</view>
				<view class="hero-actions">
					<button @click="backToToday" class="quick-btn light-btn">回到今天</button>
					<button v-if="focusDate" @click="openScheduleDialog(focusDate)" class="quick-btn dark-btn">新增日程</button>
				</view>
			</view>

			<view class="calendar-card fade-up delay-2">
				<view class="calendar-header">
					<button @click="switchMonth(-1)" class="month-btn ghost-btn">&lt;</button>
					<view @click="showDatePicker" class="current-month">
						<text class="month-title">{{ currentYear }}年{{ currentMonth }}月</text>
						<text class="dropdown-icon">▼</text>
					</view>
					<button @click="switchMonth(1)" class="month-btn ghost-btn">&gt;</button>
				</view>

				<view class="calendar-tips">
					<text class="calendar-tip">左右滑动切月，点击日期查看或创建日程</text>
				</view>

				<view v-if="showPicker" class="date-picker-mask" @click="showPicker = false">
					<view class="date-picker" @click.stop>
						<view class="picker-header">
							<text @click="showPicker = false" class="picker-close">取消</text>
							<text class="picker-title">选择日期</text>
							<text @click="confirmDate" class="picker-confirm">确定</text>
						</view>
						<view class="picker-body">
							<view class="picker-column">
								<text class="column-title">年份</text>
								<scroll-view scroll-y="true" class="scroll-view" :scroll-top="yearScrollTop">
									<text
										v-for="year in years"
										:key="year"
										class="picker-item"
										:class="{ active: selectedYear === year }"
										@click="selectYear(year)"
									>{{ year }}</text>
								</scroll-view>
							</view>
							<view class="picker-column">
								<text class="column-title">月份</text>
								<scroll-view scroll-y="true" class="scroll-view" :scroll-top="monthScrollTop">
									<text
										v-for="month in months"
										:key="month"
										class="picker-item"
										:class="{ active: selectedMonth === month }"
										@click="selectMonth(month)"
									>{{ month }}</text>
								</scroll-view>
							</view>
						</view>
					</view>
				</view>

				<view class="weekdays">
					<text v-for="day in weekdays" :key="day" class="weekday">{{ day }}</text>
				</view>

				<swiper class="calendar-swiper" :current="swiperCurrent" @change="handleCalendarSwiperChange" :duration="280">
					<swiper-item v-for="panel in calendarPanels" :key="panel.key">
						<view class="calendar-panel-wrap">
							<view class="calendar-days">
								<view
									v-for="(day, index) in panel.days"
									:key="panel.key + '-' + day.date + '-' + index"
									class="day-container"
									:class="{
										'other-month': day.month !== panel.month,
										checked: isChecked(day.date),
										today: isToday(day.date),
										missed: isMissed(day.date),
										selected: selectedDate === day.date
									}"
									@click="selectDate(day.date)"
								>
									<text class="day">{{ day.day }}</text>
									<view v-if="isChecked(day.date)" class="check-mark">✓</view>
									<view v-else-if="isMissed(day.date)" class="missed-mark">×</view>
									<view v-if="getSchedulesByDate(day.date).length > 0" class="schedule-mark"></view>
								</view>
							</view>
						</view>
					</swiper-item>
				</swiper>
			</view>

			<view class="focus-card fade-up delay-3">
				<view class="focus-header">
					<view>
						<text class="focus-title">{{ focusPanelTitle }}</text>
						<text class="focus-subtitle">{{ focusPanelSubtitle }}</text>
					</view>
					<view class="focus-meta-pill">未完成 {{ uncheckedDays }} 天</view>
				</view>

				<view class="schedule-list" v-if="selectedDaySchedules.length">
					<view v-for="schedule in selectedDaySchedules" :key="schedule.id" class="schedule-item">
						<view class="schedule-time-pill">{{ schedule.time || '全天' }}</view>
						<view class="schedule-body">
							<text class="schedule-title">{{ schedule.title }}</text>
							<text class="schedule-meta">{{ schedule.reminder ? '已开启提醒' : '未开启提醒' }}</text>
						</view>
					</view>
				</view>

				<view v-else class="empty-state">
					<text class="empty-title">暂无日程</text>
					<text class="empty-desc">点上方日期或右上按钮，直接补充今天的安排。</text>
				</view>
			</view>

			<view v-if="showMissedReminder" class="reminder-overlay reminder-overlay-enter" @click="closeMissedReminder">
				<view class="reminder-dialog reminder-dialog-enter">
					<text class="reminder-title">学习提醒</text>
					<text class="reminder-text">昨天忘记学习，那今天可要加倍学习哦</text>
					<text class="reminder-tip">点击任意位置关闭</text>
				</view>
			</view>

			<view v-if="showScheduleDialog" class="schedule-dialog" @click="closeScheduleDialog">
				<view class="dialog-content slide-up" @click.stop>
					<view class="dialog-header">
						<text class="dialog-title">添加日程</text>
						<text @click="closeScheduleDialog" class="dialog-close">×</text>
					</view>
					<view class="dialog-body">
						<view class="form-item">
							<text class="form-label">日期</text>
							<text class="form-value">{{ scheduleDate }}</text>
						</view>
						<view class="form-item vertical-item">
							<text class="form-label block-label">标题</text>
							<input type="text" v-model="scheduleTitle" placeholder="例如：英语听力 30 分钟" class="form-input" />
						</view>
						<view class="form-row">
							<view class="form-item vertical-item compact-item">
								<text class="form-label block-label">时间</text>
								<input type="time" v-model="scheduleTime" class="form-input" />
							</view>
							<view class="form-item toggle-item compact-item">
								<text class="form-label block-label">提醒</text>
								<switch :checked="scheduleReminder" @change="scheduleReminder = $event.detail.value" class="form-switch" color="#2d6df6" />
							</view>
						</view>
					</view>
					<view class="dialog-footer">
						<button @click="closeScheduleDialog" class="dialog-btn cancel-btn">取消</button>
						<button @click="saveSchedule" class="dialog-btn confirm-btn">保存</button>
					</view>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				currentDate: '',
				currentYear: 0,
				currentMonth: 0,
				weekdays: ['日', '一', '二', '三', '四', '五', '六'],
				calendarDays: [],
				checkedDates: [],
				checkedDays: 0,
				uncheckedDays: 0,
				showPicker: false,
				selectedYear: 0,
				selectedMonth: 0,
				years: [],
				months: Array.from({ length: 12 }, (_, index) => index + 1),
				yearScrollTop: 0,
				monthScrollTop: 0,
				fallbackQuotes: [
					'今天多走的一小步，会在未来接成一段长路。',
					'把重复过好的每一天，最后都会变成上升的轨迹。',
					'慢一点没关系，只要你还在持续靠近目标。',
					'真正拉开差距的，往往是那些不动声色的坚持。',
					'先完成今天，再去想远方，进步就是这样累积的。',
					'你为自己守住的每一次自律，都会变成底气。',
					'不用等状态最好，开始行动本身就会带来状态。',
					'把平凡的日子认真过，时间自然会给出答案。',
					'哪怕进步只有一点点，也比停在原地更接近理想。',
					'坚持不是一口气冲很远，而是每天都没有放弃。',
					'今天愿意坐下来学习的你，已经在超越昨天的自己。',
					'积累看似安静，却总会在某一天发出回响。'
				],
				dailyQuoteText: '正在加载今日名言...',
				dailyQuoteSource: '',
				quoteLoading: false,
				quoteAnimateKey: 0,
				checkInPulse: false,
				schedules: [],
				showScheduleDialog: false,
				showMissedReminder: false,
				currentSchedule: null,
				scheduleDate: '',
				scheduleTitle: '',
				scheduleTime: '',
				scheduleReminder: false,
				selectedDate: '',
				swiperCurrent: 1
			};
		},
		onLoad() {
			this.initData();
			this.generateCalendar();
			this.updateStats();
		},
		computed: {
			isCheckedToday() {
				return this.isChecked(this.currentDate);
			},
			currentWeekdayIndex() {
				const current = new Date(`${this.currentDate}T00:00:00`);
				return current.getDay();
			},
			currentWeekdayEnglish() {
				const weekdayMap = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'];
				return weekdayMap[this.currentWeekdayIndex] || 'Today';
			},
			heroThemeStyle() {
				const palette = [
					{
						baseA: 'rgba(255, 120, 86, 0.54)',
						baseB: 'rgba(255, 196, 92, 0.34)',
						baseC: 'rgba(255, 96, 145, 0.28)',
						sun: 'radial-gradient(circle, rgba(255, 210, 110, 0.90) 0%, rgba(255, 157, 86, 0.18) 72%, transparent 100%)',
						mint: 'radial-gradient(circle, rgba(255, 132, 172, 0.46) 0%, rgba(255, 132, 172, 0.08) 70%, transparent 100%)',
						rose: 'radial-gradient(circle, rgba(255, 236, 160, 0.42) 0%, rgba(255, 236, 160, 0.10) 72%, transparent 100%)'
					},
					{
						baseA: 'rgba(83, 112, 255, 0.56)',
						baseB: 'rgba(74, 217, 217, 0.34)',
						baseC: 'rgba(122, 129, 255, 0.28)',
						sun: 'radial-gradient(circle, rgba(114, 161, 255, 0.88) 0%, rgba(114, 161, 255, 0.18) 72%, transparent 100%)',
						mint: 'radial-gradient(circle, rgba(73, 232, 214, 0.52) 0%, rgba(73, 232, 214, 0.10) 70%, transparent 100%)',
						rose: 'radial-gradient(circle, rgba(176, 156, 255, 0.40) 0%, rgba(176, 156, 255, 0.10) 72%, transparent 100%)'
					},
					{
						baseA: 'rgba(62, 182, 145, 0.56)',
						baseB: 'rgba(111, 218, 146, 0.34)',
						baseC: 'rgba(83, 205, 232, 0.28)',
						sun: 'radial-gradient(circle, rgba(112, 233, 172, 0.86) 0%, rgba(112, 233, 172, 0.18) 72%, transparent 100%)',
						mint: 'radial-gradient(circle, rgba(102, 214, 255, 0.44) 0%, rgba(102, 214, 255, 0.10) 70%, transparent 100%)',
						rose: 'radial-gradient(circle, rgba(181, 247, 199, 0.38) 0%, rgba(181, 247, 199, 0.10) 72%, transparent 100%)'
					},
					{
						baseA: 'rgba(133, 92, 255, 0.54)',
						baseB: 'rgba(86, 146, 255, 0.34)',
						baseC: 'rgba(255, 122, 184, 0.26)',
						sun: 'radial-gradient(circle, rgba(159, 129, 255, 0.86) 0%, rgba(159, 129, 255, 0.18) 72%, transparent 100%)',
						mint: 'radial-gradient(circle, rgba(93, 184, 255, 0.46) 0%, rgba(93, 184, 255, 0.10) 70%, transparent 100%)',
						rose: 'radial-gradient(circle, rgba(255, 141, 208, 0.38) 0%, rgba(255, 141, 208, 0.10) 72%, transparent 100%)'
					},
					{
						baseA: 'rgba(255, 125, 92, 0.54)',
						baseB: 'rgba(255, 173, 80, 0.34)',
						baseC: 'rgba(255, 103, 125, 0.28)',
						sun: 'radial-gradient(circle, rgba(255, 182, 92, 0.90) 0%, rgba(255, 182, 92, 0.18) 72%, transparent 100%)',
						mint: 'radial-gradient(circle, rgba(255, 118, 99, 0.46) 0%, rgba(255, 118, 99, 0.10) 70%, transparent 100%)',
						rose: 'radial-gradient(circle, rgba(255, 221, 147, 0.38) 0%, rgba(255, 221, 147, 0.10) 72%, transparent 100%)'
					},
					{
						baseA: 'rgba(40, 177, 188, 0.54)',
						baseB: 'rgba(56, 132, 255, 0.34)',
						baseC: 'rgba(97, 223, 178, 0.28)',
						sun: 'radial-gradient(circle, rgba(88, 222, 229, 0.88) 0%, rgba(88, 222, 229, 0.18) 72%, transparent 100%)',
						mint: 'radial-gradient(circle, rgba(104, 155, 255, 0.48) 0%, rgba(104, 155, 255, 0.10) 70%, transparent 100%)',
						rose: 'radial-gradient(circle, rgba(133, 239, 198, 0.40) 0%, rgba(133, 239, 198, 0.10) 72%, transparent 100%)'
					},
					{
						baseA: 'rgba(255, 98, 154, 0.52)',
						baseB: 'rgba(137, 113, 255, 0.34)',
						baseC: 'rgba(255, 184, 96, 0.28)',
						sun: 'radial-gradient(circle, rgba(255, 132, 186, 0.88) 0%, rgba(255, 132, 186, 0.18) 72%, transparent 100%)',
						mint: 'radial-gradient(circle, rgba(157, 132, 255, 0.48) 0%, rgba(157, 132, 255, 0.10) 70%, transparent 100%)',
						rose: 'radial-gradient(circle, rgba(255, 208, 120, 0.40) 0%, rgba(255, 208, 120, 0.10) 72%, transparent 100%)'
					}
				][this.currentWeekdayIndex] || {};

				return {
					'--hero-base-a': palette.baseA,
					'--hero-base-b': palette.baseB,
					'--hero-base-c': palette.baseC,
					'--hero-sun-gradient': palette.sun,
					'--hero-mint-gradient': palette.mint,
					'--hero-rose-gradient': palette.rose
				};
			},
			dailyQuote() {
				return this.dailyQuoteText;
			},
			focusDate() {
				return this.selectedDate || this.currentDate;
			},
			focusDateLabel() {
				return this.selectedDate ? '已选日期' : '今天';
			},
			focusPanelTitle() {
				return this.focusDate === this.currentDate ? '今日安排' : '当日安排';
			},
			focusPanelSubtitle() {
				return this.focusDate || '请选择日期';
			},
			selectedDaySchedules() {
				return this.focusDate ? this.getSchedulesByDate(this.focusDate) : [];
			},
			completionRate() {
				const total = this.checkedDays + this.uncheckedDays;
				if (!total) {
					return 0;
				}
				return Math.round((this.checkedDays / total) * 100);
			},
			calendarPanels() {
				return [-1, 0, 1].map((offset) => {
					const targetDate = new Date(this.currentYear, this.currentMonth - 1 + offset, 1);
					const panelYear = targetDate.getFullYear();
					const panelMonth = targetDate.getMonth() + 1;
					return {
						key: `${panelYear}-${panelMonth}`,
						year: panelYear,
						month: panelMonth,
						days: this.buildCalendarDays(panelYear, panelMonth)
					};
				});
			}
		},
		methods: {
			initData() {
				const now = new Date();
				this.currentYear = now.getFullYear();
				this.currentMonth = now.getMonth() + 1;
				this.currentDate = this.formatDate(now);
				this.initDailyQuote();

				const startYear = this.currentYear - 10;
				const endYear = this.currentYear + 10;
				for (let year = startYear; year <= endYear; year++) {
					this.years.push(year);
				}

				const savedCheckInData = uni.getStorageSync('checkInData');
				if (savedCheckInData) {
					this.checkedDates = savedCheckInData;
				}

				const savedScheduleData = uni.getStorageSync('scheduleData');
				if (savedScheduleData) {
					this.schedules = savedScheduleData;
				}
			},
			getQuoteStorageKey() {
				return 'dailyQuoteStateV2';
			},
			getRandomFallbackQuote(excludeText = '') {
				if (!this.fallbackQuotes.length) {
					return {
						text: '今天也别停下，继续往前。',
						source: '本地备用'
					};
				}

				if (this.fallbackQuotes.length === 1) {
					return {
						text: this.fallbackQuotes[0],
						source: '本地备用'
					};
				}

				let nextText = this.fallbackQuotes[Math.floor(Math.random() * this.fallbackQuotes.length)];
				while (nextText === excludeText) {
					nextText = this.fallbackQuotes[Math.floor(Math.random() * this.fallbackQuotes.length)];
				}

				return {
					text: nextText,
					source: '本地备用'
				};
			},
			applyDailyQuote(quote, animate = false) {
				this.dailyQuoteText = quote.text;
				this.dailyQuoteSource = quote.source || '';
				if (animate) {
					this.quoteAnimateKey += 1;
				}
			},
			persistDailyQuote(quote) {
				uni.setStorageSync(this.getQuoteStorageKey(), {
					date: this.currentDate,
					text: quote.text,
					source: quote.source || '',
					updatedAt: Date.now()
				});
			},
			readCachedDailyQuote() {
				const savedQuoteState = uni.getStorageSync(this.getQuoteStorageKey());
				if (
					savedQuoteState &&
					savedQuoteState.date === this.currentDate &&
					typeof savedQuoteState.text === 'string' &&
					savedQuoteState.text.trim()
				) {
					return {
						text: savedQuoteState.text,
						source: savedQuoteState.source || ''
					};
				}
				return null;
			},
			requestQuoteFromApi() {
				return new Promise((resolve, reject) => {
					uni.request({
						url: 'https://v1.hitokoto.cn/',
						method: 'GET',
						timeout: 5000,
						success: (response) => {
							const { data, statusCode } = response;
							if (statusCode !== 200 || !data || !data.hitokoto) {
								reject(new Error('invalid quote response'));
								return;
							}

							const sourceName = data.from ? `《${data.from}》` : '';
							const sourceParts = [data.from_who, sourceName].filter(Boolean);
							resolve({
								text: data.hitokoto,
								source: sourceParts.length ? `来自 ${sourceParts.join(' ')}` : '来自 一言'
							});
						},
						fail: reject
					});
				});
			},
			async initDailyQuote() {
				const cachedQuote = this.readCachedDailyQuote();
				if (cachedQuote) {
					this.applyDailyQuote(cachedQuote);
					return;
				}

				await this.fetchDailyQuote(false);
			},
			async fetchDailyQuote(showToastOnFailure) {
				if (this.quoteLoading) {
					return;
				}

				this.quoteLoading = true;
				try {
					const quote = await this.requestQuoteFromApi();
					this.applyDailyQuote(quote, true);
					this.persistDailyQuote(quote);
				} catch (error) {
					const fallbackQuote = this.getRandomFallbackQuote(this.dailyQuoteText);
					this.applyDailyQuote(fallbackQuote, true);
					this.persistDailyQuote(fallbackQuote);
					if (showToastOnFailure) {
						uni.showToast({
							title: '网络异常，已切换备用名言',
							icon: 'none'
						});
					}
				} finally {
					this.quoteLoading = false;
				}
			},
			async refreshDailyQuote() {
				await this.fetchDailyQuote(true);
			},
			getSchedulesByDate(date) {
				return this.schedules.filter((schedule) => schedule.date === date);
			},
			selectDate(date) {
				const [, month] = date.split('-').map(Number);

				if (month > this.currentMonth) {
					this.switchMonth(1);
				} else if (month < this.currentMonth) {
					this.switchMonth(-1);
				}

				this.selectedDate = date;
			},
			openScheduleDialog(date) {
				this.scheduleDate = date;
				this.scheduleTitle = '';
				this.scheduleTime = '09:00';
				this.scheduleReminder = false;
				this.currentSchedule = null;
				this.showScheduleDialog = true;
			},
			saveSchedule() {
				if (!this.scheduleTitle) {
					uni.showToast({ title: '请输入日程标题', icon: 'none' });
					return;
				}

				const schedule = {
					id: Date.now().toString(),
					date: this.scheduleDate,
					title: this.scheduleTitle,
					time: this.scheduleTime,
					reminder: this.scheduleReminder,
					createdAt: new Date().toISOString()
				};

				this.schedules.push(schedule);
				uni.setStorageSync('scheduleData', this.schedules);
				this.showScheduleDialog = false;
				this.selectedDate = this.scheduleDate;
				uni.showToast({ title: '日程保存成功', icon: 'success' });
				this.generateCalendar();
			},
			closeScheduleDialog() {
				this.showScheduleDialog = false;
			},
			backToToday() {
				const now = new Date();
				this.currentYear = now.getFullYear();
				this.currentMonth = now.getMonth() + 1;
				this.selectedDate = '';
				this.refreshCalendar();
				uni.showToast({
					title: '已返回今天',
					icon: 'success'
				});
			},
			showDatePicker() {
				this.showPicker = true;
				this.selectedYear = this.currentYear;
				this.selectedMonth = this.currentMonth;
				const yearIndex = this.years.indexOf(this.currentYear);
				this.yearScrollTop = yearIndex * 60;
				this.monthScrollTop = (this.currentMonth - 1) * 60;
			},
			selectYear(year) {
				this.selectedYear = year;
			},
			selectMonth(month) {
				this.selectedMonth = month;
			},
			confirmDate() {
				this.currentYear = this.selectedYear;
				this.currentMonth = this.selectedMonth;
				this.selectedDate = '';
				this.showPicker = false;
				this.refreshCalendar();
			},
			formatDate(date) {
				const year = date.getFullYear();
				const month = (date.getMonth() + 1).toString().padStart(2, '0');
				const day = date.getDate().toString().padStart(2, '0');
				return `${year}-${month}-${day}`;
			},
			refreshCalendar() {
				this.generateCalendar();
				this.updateStats();
				this.resetSwiperPosition();
			},
			buildCalendarDays(year, month) {
				const firstDay = new Date(year, month - 1, 1);
				const firstDayOfWeek = firstDay.getDay();
				const startDate = new Date(firstDay);
				startDate.setDate(startDate.getDate() - firstDayOfWeek);

				const days = [];
				for (let index = 0; index < 42; index++) {
					const currentDate = new Date(startDate.getTime() + index * 24 * 60 * 60 * 1000);
					const currentYear = currentDate.getFullYear();
					const currentMonth = currentDate.getMonth() + 1;
					const currentDay = currentDate.getDate();
					const dateStr = `${currentYear}-${String(currentMonth).padStart(2, '0')}-${String(currentDay).padStart(2, '0')}`;
					days.push({
						date: dateStr,
						day: currentDay,
						month: currentMonth
					});
				}
				return days;
			},
			generateCalendar() {
				this.calendarDays = this.buildCalendarDays(this.currentYear, this.currentMonth);
			},
			switchMonth(offset) {
				this.currentMonth += offset;
				if (this.currentMonth < 1) {
					this.currentMonth = 12;
					this.currentYear -= 1;
				} else if (this.currentMonth > 12) {
					this.currentMonth = 1;
					this.currentYear += 1;
				}
				this.selectedDate = '';
				this.refreshCalendar();
			},
			resetSwiperPosition() {
				this.$nextTick(() => {
					this.swiperCurrent = 1;
				});
			},
			handleCalendarSwiperChange(event) {
				const { current, source } = event.detail;
				if (source !== 'touch') {
					return;
				}
				this.swiperCurrent = current;
				if (current === 2) {
					this.switchMonth(1);
				} else if (current === 0) {
					this.switchMonth(-1);
				}
			},
			isChecked(date) {
				return this.checkedDates.includes(date);
			},
			isToday(date) {
				return date === this.currentDate;
			},
			isMissed(date) {
				const dateObj = new Date(date);
				const today = new Date(this.currentDate);
				today.setHours(0, 0, 0, 0);
				dateObj.setHours(0, 0, 0, 0);
				return dateObj < today && !this.isChecked(date);
			},
			getYesterdayDate() {
				const current = new Date(`${this.currentDate}T00:00:00`);
				current.setDate(current.getDate() - 1);
				return this.formatDate(current);
			},
			closeMissedReminder() {
				this.showMissedReminder = false;
			},
			triggerCheckInPulse() {
				this.checkInPulse = false;
				this.$nextTick(() => {
					this.checkInPulse = true;
					setTimeout(() => {
						this.checkInPulse = false;
					}, 560);
				});
			},
			checkIn() {
				if (this.isCheckedToday) {
					return;
				}
				const yesterdayDate = this.getYesterdayDate();
				const shouldShowReminder = !this.isChecked(yesterdayDate);
				this.checkedDates.push(this.currentDate);
				uni.setStorageSync('checkInData', this.checkedDates);
				this.updateStats();
				this.generateCalendar();
				this.triggerCheckInPulse();
				if (shouldShowReminder) {
					this.showMissedReminder = true;
				}
				uni.showToast({
					title: '打卡成功',
					icon: 'success'
				});
			},
			updateStats() {
				const now = new Date();
				const startOfMonth = new Date(this.currentYear, this.currentMonth - 1, 1);
				const endOfMonth = new Date(this.currentYear, this.currentMonth, 0);
				const daysInMonth = endOfMonth.getDate();

				let checked = 0;
				for (let day = 1; day <= daysInMonth; day++) {
					const date = new Date(this.currentYear, this.currentMonth - 1, day);
					const dateStr = this.formatDate(date);
					if (this.checkedDates.includes(dateStr) && date <= now) {
						checked += 1;
					}
				}

				const timeDiff = now - startOfMonth;
				const daysDiff = Math.floor(timeDiff / (1000 * 60 * 60 * 24));
				const passedDays = Math.max(0, Math.min(daysInMonth, daysDiff + 1));
				const uncheckedDays = Math.max(0, passedDays - checked);

				this.checkedDays = checked;
				this.uncheckedDays = uncheckedDays;
			}
		}
	};
</script>

<style>
	page {
		background:
			radial-gradient(circle at top left, rgba(77, 123, 255, 0.20), transparent 35%),
			radial-gradient(circle at top right, rgba(43, 208, 201, 0.18), transparent 28%),
			linear-gradient(180deg, #f4f8ff 0%, #eef3fb 48%, #f8fbff 100%);
		min-height: 100%;
	}

	.page-shell {
		position: relative;
		min-height: 100vh;
		overflow-y: auto;
		overflow-x: hidden;
	}

	.ambient {
		position: absolute;
		width: 320rpx;
		height: 320rpx;
		border-radius: 50%;
		filter: blur(8rpx);
		opacity: 0.6;
	}

	.ambient-left {
		top: 60rpx;
		left: -120rpx;
		background: rgba(45, 109, 246, 0.16);
	}

	.ambient-right {
		top: 320rpx;
		right: -110rpx;
		background: rgba(56, 197, 168, 0.14);
	}

	.container {
		position: relative;
		min-height: 100vh;
		padding: 18rpx 20rpx 22rpx;
		box-sizing: border-box;
		display: flex;
		flex-direction: column;
		gap: 14rpx;
		overflow: visible;
		z-index: 1;
	}

	.fade-up {
		animation: fadeUp 0.6s ease both;
	}

	.delay-1 {
		animation-delay: 0.05s;
	}

	.delay-2 {
		animation-delay: 0.12s;
	}

	.delay-3 {
		animation-delay: 0.18s;
	}

	.delay-4 {
		animation-delay: 0.24s;
	}

	.delay-5 {
		animation-delay: 0.3s;
	}

	.hero-card,
	.calendar-card,
	.focus-card {
		background: rgba(255, 255, 255, 0.88);
		backdrop-filter: blur(14rpx);
		border: 1rpx solid rgba(255, 255, 255, 0.75);
		box-shadow: 0 22rpx 60rpx rgba(45, 82, 138, 0.12);
		border-radius: 32rpx;
	}

	.hero-card {
		position: relative;
		overflow: hidden;
		padding: 24rpx;
		background:
			linear-gradient(135deg, var(--hero-base-a), var(--hero-base-b)),
			linear-gradient(160deg, var(--hero-base-c), rgba(255, 255, 255, 0.10));
		color: #ffffff;
		border: 1rpx solid rgba(255, 255, 255, 0.28);
		box-shadow: 0 24rpx 58rpx rgba(77, 89, 168, 0.16);
	}

	.hero-gradient {
		position: absolute;
		border-radius: 50%;
		filter: blur(12rpx);
		opacity: 0.95;
	}

	.hero-gradient-sun {
		width: 220rpx;
		height: 220rpx;
		top: -70rpx;
		right: -10rpx;
		background: var(--hero-sun-gradient);
	}

	.hero-gradient-mint {
		width: 260rpx;
		height: 260rpx;
		left: -90rpx;
		bottom: -100rpx;
		background: var(--hero-mint-gradient);
	}

	.hero-gradient-rose {
		width: 180rpx;
		height: 180rpx;
		top: 120rpx;
		left: 160rpx;
		background: var(--hero-rose-gradient);
	}

	.hero-topline,
	.hero-main,
	.hero-highlight,
	.hero-actions {
		position: relative;
		z-index: 1;
	}

	.hero-main {
		display: block;
	}

	.hero-copy {
		flex: 1;
		min-width: 0;
	}

	.quote-card {
		margin-top: 4rpx;
		padding: 12rpx 16rpx;
		border-radius: 20rpx;
		background: rgba(255, 255, 255, 0.14);
		border: 1rpx solid rgba(255, 255, 255, 0.16);
	}

	.hero-check-row {
		position: relative;
		z-index: 1;
		display: flex;
		justify-content: center;
		margin-top: 16rpx;
	}

	.hero-topline {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 28rpx;
	}

	.hero-badge,
	.hero-date {
		font-size: 22rpx;
		padding: 10rpx 18rpx;
		border-radius: 999rpx;
		background: rgba(255, 255, 255, 0.14);
		color: rgba(255, 255, 255, 0.92);
	}

	.app-title {
		display: block;
		font-size: 42rpx;
		font-weight: 700;
		letter-spacing: 2rpx;
		margin-bottom: 8rpx;
	}

	.hero-subtitle {
		display: block;
		font-size: 22rpx;
		line-height: 1.5;
		color: rgba(255, 255, 255, 0.8);
	}

	.quote-tip {
		display: block;
		margin-top: 8rpx;
		font-size: 18rpx;
		color: rgba(255, 255, 255, 0.62);
	}

	.quote-source {
		display: block;
		margin-top: 6rpx;
		font-size: 18rpx;
		color: rgba(255, 255, 255, 0.72);
	}

	.reminder-overlay {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		z-index: 1200;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 32rpx;
		background: rgba(10, 18, 38, 0.34);
		backdrop-filter: blur(6rpx);
	}

	.reminder-overlay-enter {
		animation: overlayFadeIn 0.22s ease;
	}

	.reminder-dialog {
		width: 100%;
		max-width: 560rpx;
		padding: 34rpx 30rpx;
		border-radius: 30rpx;
		background: linear-gradient(180deg, rgba(255, 255, 255, 0.96), rgba(246, 249, 255, 0.94));
		text-align: center;
		box-shadow: 0 28rpx 60rpx rgba(23, 50, 92, 0.18);
	}

	.reminder-dialog-enter {
		animation: reminderPopIn 0.26s cubic-bezier(0.2, 0.8, 0.2, 1);
	}

	.reminder-title {
		display: block;
		font-size: 30rpx;
		font-weight: 700;
		color: #24467f;
		margin-bottom: 14rpx;
	}

	.reminder-text {
		display: block;
		font-size: 28rpx;
		line-height: 1.7;
		color: #213a61;
	}

	.reminder-tip {
		display: block;
		margin-top: 18rpx;
		font-size: 20rpx;
		color: #7b8daa;
	}

	.quote-fade {
		animation: quoteFade 0.24s ease;
	}

	.hero-check-btn {
		width: 280rpx;
		height: 88rpx;
		line-height: 88rpx;
		transform: scale(1);
		transition: transform 0.24s cubic-bezier(0.22, 1.2, 0.32, 1), box-shadow 0.24s ease, filter 0.2s ease, background 0.2s ease;
	}

	.quote-card:active {
		transform: scale(0.98);
		filter: brightness(0.99);
		box-shadow: inset 0 1rpx 0 rgba(255, 255, 255, 0.18), 0 10rpx 24rpx rgba(8, 20, 60, 0.12);
		margin: 0;
		border: none;
		border-radius: 999rpx;
		background: linear-gradient(180deg, #ffffff, #dfeaff);
		color: #1f58d8;
		font-size: 28rpx;
		font-weight: 700;
		box-shadow: 0 14rpx 32rpx rgba(8, 20, 60, 0.16);
		flex-shrink: 0;
		transform: scale(1);
		transition: transform 0.24s cubic-bezier(0.22, 1.2, 0.32, 1), box-shadow 0.24s ease, filter 0.2s ease, opacity 0.2s ease;
	}

	.hero-check-btn.success-pulse {
		animation: checkInPulse 0.52s cubic-bezier(0.22, 1.2, 0.32, 1);
	}

	.hero-check-btn[disabled] {
		background: linear-gradient(180deg, rgba(255, 255, 255, 0.54), rgba(219, 230, 255, 0.38));
		color: rgba(255, 255, 255, 0.86);
		box-shadow: none;
	}

	.hero-highlight {
		margin-top: 18rpx;
		display: flex;
		align-items: center;
		padding: 14rpx 18rpx;
		border-radius: 24rpx;
		background: rgba(10, 26, 74, 0.16);
	}

	.highlight-item {
		flex: 1;
	}

	.compact-highlight {
		min-width: 0;
	}

	.highlight-divider {
		width: 1rpx;
		height: 52rpx;
		background: rgba(255, 255, 255, 0.2);
		margin: 0 14rpx;
	}

	.highlight-label,
	.highlight-value,
	.stat-caption,
	.stat-desc,
	.calendar-tip,
	.focus-subtitle,
	.schedule-meta,
	.empty-desc {
		display: block;
	}

	.highlight-label {
		font-size: 18rpx;
		color: rgba(255, 255, 255, 0.68);
		margin-bottom: 4rpx;
	}

	.highlight-value {
		font-size: 30rpx;
		font-weight: 700;
	}

	.highlight-value.small {
		font-size: 24rpx;
	}

	.hero-actions {
		margin-top: 14rpx;
		display: grid;
		grid-template-columns: repeat(2, minmax(0, 1fr));
		gap: 12rpx;
	}

	.quick-btn {
		height: 68rpx;
		line-height: 68rpx;
		margin: 0;
		border: none;
		border-radius: 999rpx;
		font-size: 24rpx;
		font-weight: 700;
		transform: scale(1);
		transition: transform 0.24s cubic-bezier(0.22, 1.2, 0.32, 1), box-shadow 0.24s ease, filter 0.2s ease, opacity 0.2s ease;
	}

	.light-btn {
		background: rgba(255, 255, 255, 0.14);
		color: #ffffff;
	}

	.dark-btn {
		background: rgba(10, 26, 74, 0.22);
		color: #ffffff;
	}

	.calendar-card {
		padding: 20rpx 18rpx 18rpx;
		display: flex;
		flex-direction: column;
	}

	.calendar-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 8rpx;
	}

	.ghost-btn {
		width: 72rpx;
		height: 72rpx;
		line-height: 72rpx;
		padding: 0;
		border-radius: 50%;
		border: none;
		background: linear-gradient(180deg, #f7fbff, #e9f1ff);
		color: #2458b9;
		font-size: 28rpx;
		box-shadow: 0 10rpx 22rpx rgba(67, 111, 180, 0.12);
		transform: scale(1);
		transition: transform 0.24s cubic-bezier(0.22, 1.2, 0.32, 1), box-shadow 0.24s ease, filter 0.2s ease, opacity 0.2s ease;
	}

	.current-month {
		display: flex;
		align-items: center;
		gap: 10rpx;
		padding: 10rpx 18rpx;
		border-radius: 999rpx;
		background: #f4f8ff;
	}

	.month-title {
		font-size: 28rpx;
		font-weight: 700;
		color: #16396d;
	}

	.dropdown-icon {
		font-size: 18rpx;
		color: #5f7397;
	}

	.calendar-tips {
		margin-bottom: 10rpx;
	}

	.calendar-tip {
		font-size: 20rpx;
		color: #6c7e99;
	}

	.date-picker-mask,
	.schedule-dialog {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background: rgba(12, 20, 42, 0.34);
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 999;
		padding: 24rpx;
	}

	.date-picker,
	.dialog-content {
		width: 100%;
		max-width: 620rpx;
		background: #ffffff;
		border-radius: 28rpx;
		overflow: hidden;
		box-shadow: 0 24rpx 60rpx rgba(15, 31, 62, 0.18);
	}

	.slide-up {
		animation: slideUp 0.28s ease both;
	}

	.picker-header,
	.dialog-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 24rpx 26rpx;
		border-bottom: 1rpx solid #edf2fb;
	}

	.picker-close,
	.picker-confirm,
	.dialog-close {
		font-size: 26rpx;
		color: #2d6df6;
	}

	.picker-title,
	.dialog-title {
		font-size: 30rpx;
		font-weight: 700;
		color: #19355f;
	}

	.picker-body {
		display: flex;
		padding: 24rpx;
		gap: 16rpx;
	}

	.picker-column {
		flex: 1;
		padding: 18rpx;
		border-radius: 22rpx;
		background: #f7faff;
	}

	.column-title {
		display: block;
		font-size: 22rpx;
		color: #7485a3;
		margin-bottom: 14rpx;
		text-align: center;
	}

	.scroll-view {
		height: 300rpx;
	}

	.picker-item {
		display: block;
		height: 60rpx;
		line-height: 60rpx;
		font-size: 26rpx;
		text-align: center;
		color: #365177;
		border-radius: 16rpx;
	}

	.picker-item.active {
		background: #e7efff;
		color: #2d6df6;
		font-weight: 700;
	}

	.weekdays {
		display: flex;
		justify-content: space-between;
		margin-bottom: 8rpx;
	}

	.weekday {
		width: 14.28%;
		text-align: center;
		font-size: 20rpx;
		font-weight: 700;
		color: #7083a3;
	}

	.calendar-panel-wrap {
		overflow: hidden;
		height: 100%;
	}

	.calendar-swiper {
		height: 430rpx;
	}

	.calendar-days {
		display: flex;
		flex-wrap: wrap;
		height: 100%;
		align-content: flex-start;
	}

	.day-container {
		width: 14.28%;
		height: 72rpx;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		position: relative;
		margin-bottom: 4rpx;
		border-radius: 18rpx;
		transform: scale(1);
		transition: transform 0.24s cubic-bezier(0.22, 1.2, 0.32, 1), background 0.2s ease, box-shadow 0.24s ease, filter 0.2s ease;
	}

	.day-container:active {
		transform: scale(0.96);
		filter: brightness(0.985);
		box-shadow: 0 8rpx 18rpx rgba(67, 111, 180, 0.10);
	}

	.day {
		font-size: 24rpx;
		font-weight: 600;
		color: #213a61;
	}

	.other-month .day {
		color: #b1bdd0;
	}

	.today {
		background: linear-gradient(180deg, #edf4ff, #dce9ff);
	}

	.today .day {
		color: #2d6df6;
	}

	.checked {
		background: linear-gradient(180deg, #effaf7, #dbf5ee);
	}

	.missed {
		background: linear-gradient(180deg, #fff4f1, #ffe6df);
	}

	.selected {
		background: linear-gradient(180deg, #2d6df6, #1f58d8);
		box-shadow: 0 14rpx 28rpx rgba(45, 109, 246, 0.24);
	}

	.selected .day,
	.selected .check-mark,
	.selected .missed-mark {
		color: #ffffff;
	}

	.check-mark,
	.missed-mark {
		position: absolute;
		bottom: 8rpx;
		font-size: 16rpx;
		font-weight: 700;
	}

	.check-mark {
		color: #2ab37a;
	}

	.missed-mark {
		color: #eb6b5a;
	}

	.schedule-mark {
		position: absolute;
		top: 10rpx;
		right: 16rpx;
		width: 12rpx;
		height: 12rpx;
		border-radius: 50%;
		background: #ffb02e;
		box-shadow: 0 0 0 6rpx rgba(255, 176, 46, 0.16);
	}

	.focus-card {
		padding: 18rpx 18rpx 16rpx;
	}

	.focus-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		gap: 16rpx;
		margin-bottom: 12rpx;
	}

	.focus-title {
		display: block;
		font-size: 28rpx;
		font-weight: 700;
		color: #17325c;
		margin-bottom: 4rpx;
	}

	.focus-subtitle {
		font-size: 20rpx;
		color: #7385a2;
	}

	.focus-meta-pill {
		padding: 10rpx 18rpx;
		border-radius: 999rpx;
		background: #eef4ff;
		color: #315d9f;
		font-size: 20rpx;
	}

	.schedule-list {
		display: flex;
		flex-direction: column;
		gap: 10rpx;
	}

	.schedule-item {
		display: flex;
		align-items: center;
		gap: 16rpx;
		padding: 14rpx;
		border-radius: 18rpx;
		background: linear-gradient(180deg, #fbfdff, #f2f6fd);
	}

	.schedule-time-pill {
		min-width: 100rpx;
		padding: 8rpx 14rpx;
		border-radius: 999rpx;
		background: #e4edff;
		color: #2d6df6;
		font-size: 20rpx;
		text-align: center;
		font-weight: 700;
	}

	.schedule-body {
		flex: 1;
	}

	.schedule-title {
		display: block;
		font-size: 24rpx;
		font-weight: 700;
		color: #213a61;
		margin-bottom: 4rpx;
	}

	.schedule-meta {
		font-size: 18rpx;
		color: #7b8daa;
	}

	.empty-state {
		padding: 12rpx 6rpx 4rpx;
		text-align: center;
	}

	.empty-title {
		display: block;
		font-size: 24rpx;
		font-weight: 700;
		color: #284469;
		margin-bottom: 6rpx;
	}

	.empty-desc {
		font-size: 19rpx;
		line-height: 1.5;
		color: #7b8daa;
	}

	.dialog-btn {
		border: none;
		border-radius: 999rpx;
		font-size: 28rpx;
		font-weight: 700;
		transform: scale(1);
		transition: transform 0.24s cubic-bezier(0.22, 1.2, 0.32, 1), box-shadow 0.24s ease, filter 0.2s ease, opacity 0.2s ease;
	}

	.hero-check-btn:active,
	.quick-btn:active,
	.ghost-btn:active,
	.dialog-btn:active {
		transform: scale(0.96);
		filter: brightness(0.98);
	}

	.hero-check-btn:active {
		box-shadow: 0 8rpx 18rpx rgba(8, 20, 60, 0.14);
	}

	.quick-btn:active,
	.ghost-btn:active,
	.dialog-btn:active {
		box-shadow: 0 6rpx 14rpx rgba(67, 111, 180, 0.10);
	}

	.hero-check-btn[disabled]:active {
		transform: scale(1);
		filter: none;
	}

	.hero-check-btn[disabled].success-pulse {
		opacity: 1;
	}

	.dialog-body {
		padding: 24rpx 26rpx;
	}

	.form-item {
		display: flex;
		align-items: center;
		margin-bottom: 18rpx;
	}

	.vertical-item {
		flex-direction: column;
		align-items: stretch;
	}

	.compact-item {
		flex: 1;
		margin-bottom: 0;
	}

	.toggle-item {
		justify-content: space-between;
		background: #f7faff;
		padding: 16rpx 20rpx;
		border-radius: 20rpx;
	}

	.form-row {
		display: flex;
		gap: 16rpx;
	}

	.form-label {
		width: 100rpx;
		font-size: 24rpx;
		color: #6e80a0;
	}

	.block-label {
		width: auto;
		margin-bottom: 10rpx;
	}

	.form-value {
		flex: 1;
		font-size: 24rpx;
		color: #18355f;
	}

	.form-input {
		width: 100%;
		height: 78rpx;
		padding: 0 20rpx;
		border-radius: 20rpx;
		background: #f7faff;
		border: 1rpx solid #e9eff8;
		font-size: 24rpx;
		box-sizing: border-box;
	}

	.form-switch {
		transform: scale(0.85);
		transform-origin: right center;
	}

	.dialog-footer {
		display: flex;
		gap: 16rpx;
		padding: 0 26rpx 26rpx;
	}

	.dialog-btn {
		flex: 1;
		height: 82rpx;
		line-height: 82rpx;
	}

	.cancel-btn {
		background: #edf3fb;
		color: #5b7094;
	}

	.confirm-btn {
		background: linear-gradient(180deg, #2d6df6, #1f58d8);
		color: #ffffff;
	}

	@keyframes fadeUp {
		from {
			opacity: 0;
			transform: translateY(28rpx);
		}
		to {
			opacity: 1;
			transform: translateY(0);
		}
	}

	@keyframes slideUp {
		from {
			opacity: 0;
			transform: translateY(40rpx);
		}
		to {
			opacity: 1;
			transform: translateY(0);
		}
	}

	@keyframes quoteFade {
		from {
			opacity: 0;
			transform: translateY(8rpx);
		}
		to {
			opacity: 1;
			transform: translateY(0);
		}
	}

	@keyframes checkInPulse {
		0% {
			transform: scale(1);
			box-shadow: 0 14rpx 32rpx rgba(8, 20, 60, 0.16);
		}
		45% {
			transform: scale(1.04);
			box-shadow: 0 22rpx 42rpx rgba(45, 109, 246, 0.24);
		}
		100% {
			transform: scale(1);
			box-shadow: 0 14rpx 32rpx rgba(8, 20, 60, 0.16);
		}
	}

	@keyframes overlayFadeIn {
		from {
			opacity: 0;
		}
		to {
			opacity: 1;
		}
	}

	@keyframes reminderPopIn {
		from {
			opacity: 0;
			transform: scale(0.92) translateY(18rpx);
		}
		to {
			opacity: 1;
			transform: scale(1) translateY(0);
		}
	}

	@media screen and (max-width: 360px) {
		.container {
			padding-left: 14rpx;
			padding-right: 14rpx;
		}

		.hero-topline,
		.hero-main,
		.focus-header,
		.form-row {
			flex-direction: column;
			align-items: stretch;
		}

		.hero-actions {
			grid-template-columns: 1fr 1fr;
		}

		.hero-check-btn {
			width: 100%;
		}

		.schedule-item {
			flex-direction: column;
			align-items: flex-start;
		}
	}
</style>
