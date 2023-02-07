一、等高虚拟滚动

   虚拟滚动的作用就是让浏览器不需要一次性渲染大量数据，而是根据用户滚动，来动态取出数据来渲染

   可以将虚拟滚动页面的构成这样理解：

   
   
   - 页面构成
   
     外部盒子：外部固定高度的盒子
   
     内部盒子：内部模拟全部列表元素高度的盒子
   
     真实的渲染区域:由渲染数组渲染出来的可视区和上下缓存区
   
   
   
   - 实现方法
   
     虚拟滚动的核心就是无论如何滚动，都要让可视区一直位于可视范围。
   
     假设我们什么都不做，那么如果此时我们滚动了`scrollTop`的高度，那么可视区就会向上滚动`scrollTop`的高度。当`scrollTop`大于渲染数组高度时，那么整个可视区就会向上滚动从而消失
   
     我们为了让可视区重新出现，那么只需要将可视区向下偏移`scrollTop`的高度即可，偏移我们用css的`transform:translateY`来实现
   
   
   
   - 代码实现
   
     先设计页面：页面主要由三个部分构成，分别是外盒子、内盒子以及可视区
   
     ```vue
     <template>
       <div>
         <!-- 外盒子 -->
         <div class="outSideBox" @scroll="scrollFn">
           <!-- 内盒子 -->
           <div class="inSideBox" :style="`height:${modelRef.height}px`">
             <!-- 可视区 -->
             <div :style="`transform:translateY(${modelRef.offsetY}px)`">
               <div
                 v-for="(item, index) in modelRef.curList"
                 :key="index"
                 class="list-item"
                 :style="{ height: `${modelRef.itemHight}px` }"
               >
                 {{ item }}
               </div>
             </div>
           </div>
         </div>
       </div>
     </template>
     
     <style lang="scss" scoped>
     .outSideBox {
       height: 600px;
       width: 375px;
       border: 1px solid red;
       margin: 0 auto;
       overflow: auto;
     
       .inSideBox {
         .list-item {
           border: 1px solid red;
           text-align: center;
         }
       }
     }
     </style>
     ```
   
     先定义一些基础变量
   
     ```js
     const modelRef = ref({
       //内部盒子高度
       height: 0,
       //可视区域每个item的高度
       itemHight: 100,
       //可视区域偏移高度
       offsetY: 0,
       //数据数组
       totalList: [],
       //渲染数组,
       curList: [],
     });
     //渲染条数
     const pageNum = ref(10);
     //上下缓存区高度
     const cacheNum = ref(5);
     //渲染数组起始位置
     const curIndex = ref(0);
     ```
   
     模拟数据
   
     ```js
     //模拟数据接口
     const listDataApi = new Promise((resolve, reject) => {
       setTimeout(() => {
         let arr = new Array(1000).fill(0).map((item, index) => {
           return index;
         });
         resolve({
           code: 200,
           data: arr,
         });
       }, 1000);
     });
     
     //模拟数据请求
     const getListData = async () => {
       let res = await listDataApi;
       if (res.code == 200) {
         //更新数据数组
         modelRef.value.totalList = res.data;
         //更新内盒子总高度
         modelRef.value.height =
           modelRef.value.totalList.length * modelRef.value.itemHight;
         //更新渲染数组
         modelRef.value.curList = modelRef.value.totalList.slice(
           curIndex.value,
           pageNum.value + cacheNum.value
         );
       }
     };
     getListData();
     ```
   
     滚动事件回调函数
   
     虚拟列表实现的核心就在滚动事件的回调函数中。通过事件event参数，我们可以拿到当前的`scrollTop`的值
   
     ```js
     //滚动事件回调函数，实现虚拟滚动逻辑
     const scrollFn = (e) => {
       //没有数据之前不执行
       if (modelRef.value.totalList.length == 0) return;
     
       let scrollTop = e.target.scrollTop;
     };
     ```
   
     根据`scrollTop`的值我们可以计算新渲染数组的起始下标应该是多少，等高虚拟滚动很简单，就是`scrollTop`除以单个元素的高度
   
     ```js
     //滚动事件回调函数，实现虚拟滚动逻辑
     const scrollFn = (e) => {
     	//...
     	let startIndex = Math.floor(scrollTop / modelRef.value.itemHight);
         
         //这里做个性能优化,如果起始位置没有变化，不需要执行下面的逻辑
         if (curIndex.value == startIndex) return;
     };
     ```
   
     有了新渲染数组的起始下标，就可以计算新渲染数组的终止下标，就是起始下标+渲染条数+缓存条数
   
     ```js
     //滚动事件回调函数，实现虚拟滚动逻辑
     const scrollFn = (e) => {
     	//...
         let endIndex = startIndex + pageNum.value + cacheNum.value;
     };
     ```
   
     接下来就是计算偏移量了，其实偏移量应该就是等于`scrollTop`，因为向上滚动多少，就应该偏移多少。但是为了增加平滑性，这里会这样写
   
     ```js
     //滚动事件回调函数，实现虚拟滚动逻辑
     const scrollFn = (e) => {
         //...
         modelRef.value.offsetY = scrollTop - (scrollTop % modelRef.value.itemHight);
     };
     ```
   
     当我们开启上下缓冲区时候，计算偏移量还需要考虑缓冲区的高度。举例子，渲染数组[0-4]是缓冲区，我们不需要看到缓冲区，只需要看到渲染数组的可视元素，因此当新渲染数组的起始下标大于缓冲条数时，偏移量要减去缓冲区的高度
   
     ```js
     //滚动事件回调函数，实现虚拟滚动逻辑
     const scrollFn = (e) => {
         //...
           //当卷起个数超过缓存个数
           if (startIndex > cacheNum.value) {
             modelRef.value.offsetY -= cacheNum.value * modelRef.value.itemHight;
             startIndex = startIndex - cacheNum.value;
           }
     };
     ```
   
     最后只需要根据新渲染数组的起始下标和终止下标，更新渲染数组即可，完整代码如下：
   
     ```js
     //滚动事件回调函数，实现虚拟滚动逻辑
     const scrollFn = (e) => {
       //没有数据之前不执行
       if (modelRef.value.totalList.length == 0) return;
     
       let scrollTop = e.target.scrollTop;
     
       //根据滚动高度更新渲染数组,startIndex = 距离顶部的高度/一个元素的高度
       let startIndex = Math.floor(scrollTop / modelRef.value.itemHight);
     
       //endIndex = startIndex + 渲染的条数 + 下缓冲区缓存条数
       let endIndex = startIndex + pageNum.value + cacheNum.value;
     
       //如果起始位置没有改变，则没必要执行下面逻辑
       if (curIndex.value == startIndex) return;
     
       //根据滚动高度，让内部盒子偏移
       modelRef.value.offsetY =
         scrollTop - (scrollTop % modelRef.value.itemHight);
     
       //当卷起个数超过缓存个数
       if (startIndex > cacheNum.value) {
         modelRef.value.offsetY -= cacheNum.value * modelRef.value.itemHight;
         startIndex = startIndex - cacheNum.value;
       }
     
       //记录最新的起始位置
       curIndex.value = startIndex;
     
       //更新渲染数组
       modelRef.value.curList = modelRef.value.totalList.slice(
         startIndex,
         endIndex
       );
     };
     ```
     
     

二、非等高虚拟滚动

   在等高虚拟滚动里，由于每一个item元素的高度是一定的，因此我们可以很容易的计算渲染数组的起始位置`startIndex`和终止位置`endIndex`，但是对于每个元素高度不等，就不能直接通过`Math.floor(scrollTop / modelRef.value.itemHight)`去计算了。因此在非等高虚拟滚动中，难点就是如何获得起始和终止索引以及偏移量

   

   解决方法：**以预估高度先行渲染，然后获取真实高度并缓存**。

   
   
   下面是html文件
   
   ```html
   <template>
     <div>
       <!-- 外盒子 -->
       <div class="outSideBox" @scroll="scrollFn">
         <!-- 内盒子 -->
         <div class="inSideBox" :style="`height: ${listHeight}px`">
           <!-- 可视区 -->
           <div :style="`transform:translateY(${modelRef.offsetY}px)`">
             <div
               v-for="(item, index) in curList"
               :key="item.key"
               class="list-item"
               :data-index="item.key"
               :style="`height:${item.height}px`"
             >
               {{ item.key }}
             </div>
           </div>
         </div>
       </div>
     </div>
   </template>
   ```
   
   先定义一些基础变量
   
   ```js
   const modelRef = ref({
     //可视区域偏移高度
     offsetY: 0,
     //缓存区长度,比例,
     cache: 1,
     //起始索引
     startIndex: 0,
     //结束索引
     endIndex: 0,
     //预估高度
     estimatedItemSize: 100,
     //列表项渲染后存储每一项的高度以及位置信息
     positions: [],
     //渲染区域高度
     screenHeight: 600
   })
   
   //完整的数组
   const totalList = ref([])
   
   //可视渲染条数：渲染区域高度/预估高度
   const visibleCount = computed({
     get: function () {
       return Math.ceil(
         modelRef.value.screenHeight / modelRef.value.estimatedItemSize
       )
     }
   })
   
   //上缓存区
   const aboveCount = computed({
     get: function () {
       return Math.min(
         modelRef.value.startIndex,
         modelRef.value.cache * visibleCount.value
       )
     }
   })
   
   //下缓存区
   const belowCount = computed({
     get: function () {
       return Math.min(
         totalList.value.length - modelRef.value.endIndex,
         modelRef.value.cache * visibleCount.value
       )
     }
   })
   
   //当前渲染数据：可视区+上下缓存区
   const curList = computed({
     get: function () {
       let start = modelRef.value.startIndex - aboveCount.value
       let end = modelRef.value.endIndex + belowCount.value
       return totalList.value.slice(start, end)
     }
   })
   
   //列表高度：positions数组中最后一个元素的底部位置
   const listHeight = computed({
     get: function () {
       return modelRef.value.positions[modelRef.value.positions.length - 1]
         ? modelRef.value.positions[modelRef.value.positions.length - 1].bottom
         : 0;
     },
   });
   ```
   
   首先模拟数据请求，拿到数据以后，对于每个元素，先给予一个预估高度`estimatedItemSize`，然后`positions`数组维护了整个列表的每一个元素的高度以及位置信息
   
   ```js
   //模拟数据接口
   const listDataApi = new Promise((resolved, rejected) => {
     setTimeout(() => {
       let arr = [];
       for (let i = 0; i < 1000; i++) {
         //随机高度
         let temp = parseInt(Math.random() * 200);
         temp = temp > 60 ? temp : 60;
   
         arr.push({
           key: i,
           height: temp,
         });
       }
       resolved({
         code: 200,
         data: arr,
       });
     }, 1000);
   });
   
   const getListData = async () => {
     let res = await listDataApi;
     if (res.code == 200) {
       totalList.value = res.data;
   
       //根据预估高度初始化每个列表的高度信息
       modelRef.value.positions = totalList.value.map((item, index) => {
         return {
           index,
           top: index * modelRef.value.estimatedItemSize,
           height: modelRef.value.estimatedItemSize,
           bottom: (index + 1) * modelRef.value.estimatedItemSize,
         };
       });
     }
   };
   
   //发起请求
   getListData();
   ```
   
   这时候，`positions`数组每一项的高度都是一样的，但是实际上每一个元素的高度是不一定的，是由元素内容动态决定的，我们先假设就根据当前`positions`数组来计算起始和终止索引以及偏移量
   
   在scrollFn中实现：
   
   ```js
   //滚动事件回调函数
   const scrollFn = (e) => {
     if (totalList.value.length == 0) return;
   
     let scrollTop = e.target.scrollTop;
   
     //更新可视区起始坐标：positions数组中第一个bottom值大于scrollTop元素的就是起始元素
     modelRef.value.startIndex = modelRef.value.positions.findIndex((item) => {
       return item.bottom > scrollTop;
     });
     //更新可视区终止坐标：起始坐标+可视个数
     modelRef.value.endIndex = modelRef.value.startIndex + visibleCount.value;
   };
   ```
   
   偏移量offsetY，理想偏移量应该是起始索引对应的元素的top再减去上缓存区的高度，但由于positions数组中保存的数据不一定是实际Dom数据，也就是说`modelRef.value.positions[modelRef.value.startIndex - 1].bottom != modelRef.value.positions[modelRef.value.startIndex].top`，因此这里进行了优化：
   
   ```js
   //获取偏移量
     if (modelRef.value.startIndex >= 1) {
       let size =
         modelRef.value.positions[modelRef.value.startIndex].top -
         (modelRef.value.positions[
           modelRef.value.startIndex - aboveCount.value
         ]
           ? modelRef.value.positions[
               modelRef.value.startIndex - aboveCount.value
             ].top
           : 0)
       
       //优化顺滑度
       modelRef.value.offsetY =
         modelRef.value.positions[modelRef.value.startIndex - 1].bottom - size
     } else {
       modelRef.value.offsetY = 0
     }
   ```
   
   现在实现的就是等高虚拟滚动
   
   但是实际上元素是非等高的，这样会导致startIndex和offsetY的计算问题，导致页面出现闪现效果。因此每次渲染数据更新，我们都要拿到渲染了的dom的实际高度和信息，去更新`positions`数组，这个方法可以在监听渲染数组中实现
   
   
   
   这里用了取巧的方式，直接生成模拟数据的时候，给每个元素添加了一个升序Key值，用来表示这个元素在数组中所处的位置，并且绑定到dom的dataset属性上，从而方便获取并更新positions数组
   
   ```js
   watch(curList, () => {
     let itemList = document.getElementsByClassName("list-item");
     for (let i = 0; i < itemList.length; i++) {
       //根据dataSet获取该元素对应的下标值
       let index = Number(itemList[i].dataset.index);
       //根据dom获取该元素的高度
       let domHeight = itemList[i].offsetHeight;
         
       //从列表高度信息数组取出数据对比
       let oldHeight = modelRef.value.positions[index].height;
       let dValue = oldHeight - domHeight;
   
       //如果高度不一致更新列表高度信息
       if (dValue != 0) {
         //元素bottom = oldbottom - 差值
         modelRef.value.positions[index].bottom =
           modelRef.value.positions[index].bottom - dValue;
         //元素高度 = dom元素实际高度
         modelRef.value.positions[index].height = domHeight;
   
         //遍历后面的所有元素，修改元素的top和bottom值
         for (let k = index + 1; k < modelRef.value.positions.length; k++) {
           modelRef.value.positions[k].top =
             modelRef.value.positions[k - 1].bottom;
           modelRef.value.positions[k].bottom =
             modelRef.value.positions[k].bottom - dValue;
         }
       }
     }
   });
   ```
   
   实际上就是不断地修正`positions`数组，从而实现非等高虚拟滚动。但是这里也出现了双重for循环带来的高时间复杂度，因此实际非等高虚拟滚动应该配合懒加载去实现，减小`positions`数组的长度



3. 面向未来

   由于scroll事件触发太频繁，所以会带来极大的性能损耗，因此可以使用`IntersectionObserver`来代替scroll事件，`IntersectionObserver`可以监听目标元素是否出现在可视区域内，在监听的回调事件中执行可视区域数据的更新，并且`IntersectionObserver`的监听回调是异步触发，不随着目标元素的滚动而触发，性能消耗极低。



4. 遗留问题

   我们虽然实现了根据列表项动态高度下的虚拟列表，但如果列表项中包含图片，并且列表高度由图片撑开，由于图片会发送网络请求，此时无法保证我们在获取列表项真实高度时图片是否已经加载完成，从而造成计算不准确的情况。

   这种情况下，如果我们能监听列表项的大小变化就能获取其真正的高度了。我们可以使用[ResizeObserver](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fzh-CN%2Fdocs%2FWeb%2FAPI%2FResizeObserver)来监听列表项内容区域的高度改变，从而实时获取每一列表项的高度。
