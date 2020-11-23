<template>
	<view class="ve_flex_col calendar">
		<view class="ve_flex_row header">
			<i @click="lastMonth" v-if="showArea!=='month'" class="icon-ve-calendar icon-arrow-left icon-btn"></i>
			<view style="flex-grow: 1;font-weight: bold;" >
				<view @click="showArea='year'">{{currentYear}}年</view>
				<view @click="showArea='month'">{{currentMonth}}月</view>
			</view>
			<view @click="goToday" v-if="showArea==='calendar'" style="width: 20%;">今天</view>
			<view @click="showArea='calendar'" v-else style="width: 20%;">返回</view>
			<i @click="nextMonth" v-if="showArea!=='month'" class="icon-ve-calendar icon-arrow-right icon-btn"></i>
		</view>
		<!-- 月视图 -->
		<view class="ve_flex_row months" v-if="showArea==='month'">
			<view class="ve_flex_col ve_month_box"
				:class="{today:(idx+1)==currentMonth}"
				:style="
				`border: 1px solid `+backColor
				"
				@click="gotoMonth(idx)"
				v-for="(month,idx) in months" :key="month">
				{{month}}
			</view>
		</view>
		<!-- 年视图 -->
		<view class="ve_flex_row years" v-if="showArea==='year'">
			<view class="ve_flex_col ve_year_box"
				:class="{today:year==currentYear}"
				:style="
				`border: 1px solid `+backColor
				"
				@click="gotoYear(year)"
				v-for="(year,idx) in years" :key="year">
				{{year}}
			</view>
		</view>
		
		<!-- 日历视图 -->
		<view class="ve_flex_row content" v-if="showArea==='calendar'">
			<view class="ve_flex_col ve_day_box"
				:style="
				`border: 1px solid `+backColor
				"
				v-for="week in weeks" :key="week">
				<view class="week_name">{{week}}</view>
			</view>
		</view>
		<view class="ve_flex_row days" v-if="showArea==='calendar'">
			<view class="ve_flex_col ve_day_box"
				:style="
				`border: 1px solid `+backColor
				"
				@click.stop="clickDay(day)"
				:class="{
					round:mode==='round',
					linkage:mode==='linkage',
					leaf:mode==='leaf',
					selected:isMid(day),
					first:isSelectedIndex(day,0),
					end:isSelectedIndex(day,-1),
					selected:findSelected(day),
					today:day.format(valueFormat)===today.format(valueFormat),
					holiday:holidayWeeks.includes(day.day())||holidayDays.includes(day.date())||holidayList.includes(day.strValue),
					is_disabled:disabled,
					disabled:isDisabled(day),
					disabled:notInMonth(day),
					}" 
				v-for="(day,idx) in randerMonthData" 
				:key="idx">
				<view class="day" >{{day.format('D')}}</view>
				<view class="event_dot" v-if="showDot && getEventDot(day)"></view>
				<view class="event_text" v-else>{{getEventText(day)}}</view>
			</view>
		</view>
	</view>
</template>

<script>
	import days from './days.min.js'
	
	export default {
		props:{
			value:{
				// 可以绑定一个日期字符串或者一个日期字符串列表
				type:[String,Array],
				default:''
			},
			min:{
				// 最小可选的日期
				type:String,
				default:''
			},
			max:{
				// 最大可选的日期
				type:String,
				default:''
			},
			eventData:{
				// 传入array，表示事件日打圆点，传入object，value为事件字符串
				// {'2020-11-20':'入住','2020-11-23':'离开'}
				// ['2020-11-20','2020-11-22','2020-11-27']
				type:[Array,Object],
				default(){
					return {}
				}
			},
			mode:{
				//linkage 连锁模式 round 圆圈样式，leaf花瓣样式
				type:String,
				default:'round', 
			},
			overClear:{
				// 超出limit的行为，true 清空全部选中，重新选；false移除第一个选中的
				type:Boolean,
				default:true
			},
			backColor:{
				// 背景颜色
				type:String,
				default:'#f8f8f8'
			},
			valueFormat:{
				// 输出格式和演算格式，如果改变，那么其他参数的列表格式也需要同步，如果错误会造成渲染不了
				type:String,
				default:'YYYY-MM-DD'
			},
			holidayWeeks:{
				// 每周星期几算假日
				type:Array,
				default(){
					return [0,6]
				}
			},
			holidayDays:{
				// 每月几号算假日
				type:Array,
				default(){
					return []
				}
			},
			holidayList:{
				// 指定一个假日列表
				type:Array,
				default(){
					return []
				}
			},
			limit:{
				// 限制选中个数，
				// 传入value type is String的话，默认只选中一个
				// 传入value type is Array，默认无限制
				// 使用mode==linkage参数，默认可选中2
				type:Number,
				default:0, 
			},
			disabled:{
				type:Boolean,
				default:false
			},
			disabledList:{
				// 禁止选中的列表
				type:Array,
				default(){
					return []
				}
			},
			rangeList:{
				// 只能从中选取的列表
				type:Array,
				default(){
					return []
				}
			},
			crossMonth:{
				type:Boolean,
				default:false
			},
		},
		data() {
			return {
				today:days(),
				currentYear:'',
				currentMonth:'',
				randerMonthData:[],
				weeks:['一','二','三','四','五','六','日'],
				months:['1月','2月','3月','4月','5月','6月','7月','8月','9月','10月','11月','12月'],
				currentSelected:[],
				copySelectedNotSort:[], // 保存了原始点击顺序的
				showDot:true,
				currLimit:0, // 内部使用的limit
				showArea:'calendar', // year / month
				years:[],
			};
		},
		watch:{
			currentSelected(){
				this.outputValue()
			},
			value(){
				this.inputValue()
			},
			eventData(){
				this.showDot = Array.isArray(this.eventData) // 事件传入是array就打原点
			},
			showArea(newVal,oldVal){
				if(newVal==='year'){
					this.makeYears(this.currentYear)
				}
			}
		},
		computed:{
			randerTitle(){
				return `${this.currentYear}年${this.currentMonth}月`
			},
		},
		mounted() {
			this.init();
		},
		methods:{
			init(){
				this.today = days();
				this.selectedMonth(this.today.format('YYYY'),this.today.format('MM'))
				this.showDot = Array.isArray(this.eventData) // 事件传入是array就打原点
				this.makeYears();
				this.inputValue();
			},
			makeYears(midYear){
				let year = midYear
				if(!year){
					year =  days().year()
				}
				year -= 7
				let years = []
				for(let i=0;i<15;i++){
					years.push(year + i)
				}
				this.years = years
				
			},
			inputValue(){
				// 输入值
				if(typeof this.value==='string'){
					if(this.value && days(this.value).format(this.valueFormat)==='Invalid Date'){
						console.error('ve-calendar组件 v-model 值必须是日期型字符串或者包和日期型字符串的数组。')
						this.currentSelected = []
					}else if(this.value){
						this.currentSelected = [this.value]
					}else{
						this.currentSelected = []
					}
					this.currLimit = 1
				}else{
					if(this.mode==='linkage'){
						this.currLimit = 2
					}else{
						this.currLimit = this.limit
					}
					this.currentSelected = JSON.parse(JSON.stringify(this.value))
				}
				
				if(this.currentSelected.length){
					// 视图切换到有值的第一个月
					let day = days(this.currentSelected[0])
					this.selectedMonth(day.format('YYYY'),day.format('MM'))
				}
				this.copySelectedNotSort = JSON.parse(JSON.stringify(this.currentSelected))
			},
			outputValue(){
				// 输出值
				let val = ''
				if(typeof this.value==='string'){
					if(this.currentSelected && this.currentSelected.length){
						val = this.currentSelected[0]
					}
				}else{
					val = this.currentSelected
				}
				this.$emit('input',val)
				this.$emit('change',val)
			},
			additionalAttributes(day){
				// 给日期对象附加属性，后期可以在这里添加农历、节气、节日等信息
				day.strValue = day.format(this.valueFormat)
			},
			goToday(){
				let today = days()
				this.selectedMonth(today.format('YYYY'),today.format('MM'))
			},
			selectedMonth(year,month){
				// 制作渲染数据
				let oneDay = days(`${year}-${month}-01`)
				let newYear = oneDay.format('YYYY');
				let newMonth = oneDay.format('MM');
				if(newYear===this.currentYear && newMonth===this.currentMonth){
					return
				}
				this.$emit('change-year-month',newYear,newMonth)
				this.currentYear = newYear
				this.currentMonth = newMonth
				let week 
				if(oneDay.day()===0){
					week = 6
				}else{
					week = oneDay.day() - 1
				}
				// 1\2\3\4\5\6\0
				// 0\1\2\3\4\5\6
				let firstDay = oneDay.add(-week,'day')
				this.additionalAttributes(firstDay)
				let randerMonthData = []
				randerMonthData.push(firstDay)
				for(let i=1;i<42;i++){
					let thisDay = firstDay.add(i,'day')
					this.additionalAttributes(thisDay)
					randerMonthData.push(thisDay)
				}
				this.randerMonthData = randerMonthData
				return randerMonthData
			},
			findSelected(day){
				return this.currentSelected.includes(day.strValue)
			},
			isDisabled(day){
				if(this.disabledList.includes(day.strValue)){
					// 有禁用清单
					return true
				}else if(this.min && day.isBefore(days(this.min))){
					// 小于开始日期
					return true
				}else if(this.max && day.isBefore(days(this.max))){
					// 大于结束日期
					return true
				}else if(this.rangeList.length){
					// 返回列表，只允许选择范围内的列表
					if(!this.rangeList.includes(day.strValue)){
						return true
					}
				}
				
			},
			getEventDot(day){
				if(!this.showDot) return false
				return this.eventData.includes(day.strValue)
			},
			getEventText(day){
				if(this.showDot) return ''
				if(this.eventData[day.strValue]){
					return this.eventData[day.strValue]
				}
				return ''
			},
			lastMonth(){
				if(this.showArea==='calendar'){
					let month = parseInt(this.currentMonth)
					let year = parseInt(this.currentYear)
					if(month===1){
						month = 12
						year -= 1
					}else{
						month -= 1
					}
					this.selectedMonth(year,month)
				}
				if(this.showArea==='year'){
					this.makeYears(this.years[0]-7)
				}
			},
			nextMonth(){
				if(this.showArea==='calendar'){
					let month = parseInt(this.currentMonth)
					let year = parseInt(this.currentYear)
					if(month===12){
						month = 1
						year += 1
					}else{
						month += 1
					}
					this.selectedMonth(year,month)
				}
				
				if(this.showArea==='year'){
					this.makeYears(this.years[this.years.length-1]+7)
				}
				
			},
			gotoYear(year){
				this.selectedMonth(year,this.currentMonth)
				this.showArea = 'calendar'
			},
			gotoMonth(idx){
				this.selectedMonth(this.currentYear,1+idx)
				this.showArea = 'calendar'
			},
			notInMonth(day){
				if(this.crossMonth){return false}
				
				let inmonth = day.isSame(`${this.currentYear}-${this.currentMonth}-01`, 'month')
				return !inmonth
				// console.log(day.strValue,inmonth)
			},
			isSelectedIndex(day,index){
				// 判断当前日期在选中表中的位置
				let length = this.currentSelected.length
				if(length){
					if(index===-1 && length>=2){
						return day.strValue == this.currentSelected[length-1]
					}
					else if(length>index){
						return day.strValue == this.currentSelected[index]
					}
				}
				return false
			},
			isMid(day){
				// linkage模式下，判断是否在选中日期中间
				if(this.mode==='linkage' && this.currentSelected.length>=2){
					return day.isAfter(this.currentSelected[0]) && day.isBefore(this.currentSelected[this.currentSelected.length-1])
				}
			},
			clickDay(day){
				// 点击某一天
				let copyIndex = this.copySelectedNotSort.indexOf(day.strValue)
				
				let index = this.currentSelected.indexOf(day.strValue)
				let length = this.copySelectedNotSort.length
				let limit = this.currLimit

				
				if(limit && length>=limit && index<0){
					// 超出限制
					if(this.overClear===false){
						// 移除第一个选中值
						let first = this.copySelectedNotSort[0]
						let findFirst = this.currentSelected.indexOf(first)
						this.currentSelected.splice(findFirst,1)
						this.copySelectedNotSort.splice(0,1)
					}else {
						// 超出限制，清空之前选的
						this.currentSelected = []
						this.copySelectedNotSort = []
					}
				}
				if(copyIndex>=0){
					// 取消选中
					this.copySelectedNotSort.splice(copyIndex,1)
				}else{
					// 添加选中
					this.copySelectedNotSort.push(day.strValue)
				}
				if(index>=0){
					// 取消选中
					this.currentSelected.splice(index,1)
				}else{
					// 添加选中
					this.currentSelected.push(day.strValue)
				}
				this.currentSelected.sort()
				if(limit>0 && this.currentSelected.length===limit){
					// 选满
					this.outputValue()
					this.$emit('select-complete',this.currentSelected)
				}
				this.$emit('click-day',day,this.currentSelected.includes(day.strValue))
			}
		}
	}
</script>

<style lang="scss" scoped>
	.calendar {
		height: 100%;
		width: 100%;
	}
	
	.months,.years,.days {
		flex:1;
		width: 100%;
		flex-wrap: wrap;
	}
	
	// .years {
	// 	flex:1;
	// 	width: 100%;
	// 	flex-wrap: wrap;
	// }
	
	// .days {
	// 	flex:1;
	// 	width: 100%;
	// 	flex-wrap: wrap;
	// }
	.ve_flex_row {
		display: flex;
		flex-direction: row;
	}
	
	.ve_flex_col {
		display: flex;
		flex-direction: column;
		
	}
	
	.content {
		flex-wrap:wrap;
		width: 100%;
	}
	.header {
		justify-content: center;
		align-items: center;
		line-height: 2.5;
		view {
			display: flex;
			justify-content: center;
			align-items: center;
		}
	}
	.ve_year_box {
		width: calc(( (100% - 14px) / 3));
		height: calc(( (100% - 14px) / 7));
		justify-content: center;
		align-items: center;
	}
	.ve_month_box {
		width: calc(( (100% - 14px) / 3));
		height: calc(( (100% - 8px) / 4));
		justify-content: center;
		align-items: center;
	}
	.ve_day_box {
		width: calc(( (100% - 14px) / 7));
		height: calc(( (100% ) / 7));
		justify-content: center;
		align-items: center;
		border: 1px solid #fff;
		.week_name {
			font-weight: bold;
			color:#C1C1C9;
		}
		.day {
			display: flex;
			justify-content: center;
			width: 100%;
		}
		
	}
	.ve_day_box.round {
		border-radius: 50%;
	}
	.today {
		border: 1px solid #A5ACB1!important;
	}
	.selected {
		background: #70CE99;
		border: 1px solid #70CE99;
		color:#fff;
	}
	.selected.first.linkage {
		border-radius: 25% 0 0 25%;
	}
	.selected.end.linkage {
		border-radius: 0 25% 25% 0;
	}
	.selected.leaf {
		border-radius: 50% 0 50% 0;
	}
	.holiday {
		color:#fb6052;
	}
	.disabled {
		// color: #C1C1C9!important;
		opacity: 0.5;
		pointer-events: none;
	}
	.is_disabled {
		pointer-events: none;
	}
	.event_dot {
		width: 6px;
		height: 6px;
		background: #F3CA4F;
		border-radius: 50%;
		position: relative;
		bottom: 0;
	}
	
	.event_text {
		position: relative;
		// bottom: 0;
		// top:30px;
		margin-top: -5px;
		height: 10px;
		font-size: xx-small;
	}
	
	
	.icon-btn {
	    width: 20%;
	    display: flex;
	    justify-content: center;
	}
	@font-face {font-family: "ve_calendar";
	  src: url('data:application/x-font-woff2;charset=utf-8;base64,d09GMgABAAAAAAKQAAsAAAAABmwAAAJGAAEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAHEIGVgCCfgpQYgE2AiQDDAsIAAQgBYRtB0IbrAXILrBt2JMS7RwdWCgCvk2wg/AUD/9/32+fe+97f5qzomHEfF4eRdNAWWBZVASBxwG3fASK/n5z743YTef5shwQ6VKuvdL1pRmS01PqC7lpwJ7gGQLUX5fM9XqIKBu9z3M5vQl0djS/nZ3jbNKisZYf9QKMAwpsT4wiK6GMU6jdQUFY+VePCbQtMbQdLW0ciF5FuCoQDyoJRG8nrSiq1hTqiYNFvKo003OK8FJ8P/540UtSZeHG4/vFWMx8+So/32lGyBDo8QoZ24RCnEyWj3UIxXVolzstE++aFXw1n/MlC4m38on46+LuLlinlcSXRCX46ogEMqi71S4WGDu9NkRdPx5NYzx1OhXj9NHO7ftbo2GmJe7SMvv+Lnw2Jb0lIT0VvwgE/yH+M+LNTgGfu/gggQL3N98TCf6JAQ4UzwqDuTh6ogfRrAZKCW0vZi6Mj73t98/gfkLTbEe0YbFB1rRMFnYblY4d1Jp20bZl+XjHBOkQpYVNzwRh6AVJ3zuyoRZZ2FdUFvygNowOtJ0EF3ashuqPMGCMITwHFelKxmHuL3v3MDkuAmpbhXCE1KgcuJbTry1jhXTGluYk8ZglSNIlLJHnsCg01KQzjNhKmesx25ZTb7IiXYp+hwgKMBQDoXNAiWgVmY1m/YPP70GJY4UALTi62iOINNTkwGVxAHLZW0GOb3mlcSLhYUwCEtFKYInMQ4WCBurpeRkUYZZ0R7E2xo6GSdRr3e8vv+8YtIWP5UiRo+hcEelTk1SScn/TUqBkIQA=') format('woff2');
	}
	
	.icon-ve-calendar {
	  font-family: "ve_calendar" !important;
	  font-size: 24px;
	  font-style: normal;
	  font-weight: bold;
	  -webkit-font-smoothing: antialiased;
	  -moz-osx-font-smoothing: grayscale;
	}
	
	.icon-arrow-right:before {
	  content: "\e743";
	}
	
	.icon-arrow-left:before {
	  content: "\e744";
	}
	

</style>
