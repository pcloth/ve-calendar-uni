## vue easy calendar for uni-app版本

### 引用并内置了days.js作为基础日期库

### 用法
#### 下载代码，放到项目components目录下

#### 引入并在vue的components对象下完成引入
```js
import veCalendar from '@/components/ve-calendar/ve-calendar.vue';

export default {
	components: {
		veCalendar
	},
	data() {
		return {
			selectedDate:'',
			eventData:[],
		};
	}
	methods: {
		showDate() {
			this.$refs.veCalendar.open()
		},
		selectComplete(arr){
            console.log(arr,'>>')
			this.$refs.veCalendar.close()
		},
```

#### 在模板区域放置组件(推荐放置到uni-popup对话框里面)
```html
<uni-popup ref="veCalendar" type="bottom">
			<view style="background-color: #fff;margin: 20px;border-radius: 15px;height: 400px;">
				<veCalendar v-model="selectedDate"  @selectComplete="selectComplete" backColor="#fff" :eventData="eventData"></veCalendar>
			</view>
		</uni-popup>
```

### 参数
| prop | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
|value| string,array | '' | v-model绑定值 |
|min | string | '' | 最小可选日期，例如：2020-10-01，那么小于这个日期不可选 |
|max | string | '' | 最大可选日期 |
|eventData |array,object| {} | 传入array，表示事件日打圆点，传入object，value为显示的事件名称，例如 {'2020-11-20':'入住','2020-11-23':'离开'} 或者 ['2020-11-20','2020-11-22','2020-11-27']|
|mode| string | round | 模式：linkage 连锁模式 round 圆圈样式，leaf花瓣样式 |
|overClear | boolean | true | 超出limit的行为，true 清空全部选中，重新选；false移除第一个选中的 |
|backColor| string | #f8f8f8 | 背景颜色 |
|valueFormat| string | YYYY-MM-DD | 输出单个日期的格式，参见days.js文档，这个参数如果发生修改了，那么所有的传入值的日期字符串都要按照这个格式标准 |
|holidayWeeks| array | [0,6] | 每周几算假日0==周日 1-6为周一到周六|
|holidayDays| array | [] | 每月几号算假日 |
|holidayList| array | [] | 额为指定假日列表 |
|limit| number | 0 | 限制选中个数；传入value type is String的话，默认只选中一个；传入value type is Array，默认无限制；使用mode==linkage参数，默认可选中2 |
|disabled| boolean | false | 组件禁用，禁用后所有日期都不可点击 |
|disabledList| array | [] | 手动指定不可选中日期 |
|rangeList| array | [] | 手动指定可选日期，不要和disabledList冲突 |
|crossMonth| boolean | false | 允许跨月直接选中 |


### 事件
| event | 传入参数 | 说明 |
| --- | --- | --- |
|change | value | 当value发生变化时触发 |
|change-year-month| 年,月 | 改变年月的时候触发 |
|select-complete| 当前选中日期列表 | 当limit参数限定的选中个数被选满的时候触发，输出的日期列表会进行排序 |
|click-day| day,是否被选中 | 当某个日期被点中的时候触发 |

