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
