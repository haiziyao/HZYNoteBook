
#### 哈希(OK)

##### 1.两数之和

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
    
	   // 直接自己搞一个capacity,防止扩充
       int capacity = (int) (nums.length / 0.75f) + 1;
       Map<Integer,Integer> map = new HashMap<>(capacity);
       
       //  这里用到一个非常厉害的思路
       // 思考一个问题:有没有必要开始的时候就把所有数据加进去
       // 这样来说: 答案其实是两种可能, int1已知,找int2或者反过来
       // 那么我们就可以刚开始找一个放一个数,这样就可以省资源
       for(int i=0;i<nums.length;i++){
            int another = target - nums[i];
            if(map.containsKey(another)){
                return new int[]{map.get(another),i};
            }
            map.put(nums[i],i);
       }
        throw new AssertionError("没有解");
    }
}
```


##### 2.
我的初步思路

```java
class Solution {

    public List<List<String>> groupAnagrams(String[] strs) {

  

        // 处理字符串,全部把字符串按照unicode排列
        // new hashmap<str,string arr[]> 存上面处理好的字符串和索引
        // 返回


        int  apicity = (int) (strs.length / 0.75f)+1;

        Map<String,List<String>> groups = new HashMap<>(capicity);

        for(String str: strs){

            char[] chars = str.toCharArray();

            Arrays.sort(chars);

            String key = new String(chars);

            groups.computeIfAbsent(key,value->new ArrayList<>())

                        .add(str);

        }
        return new ArrayList<>(groups.values());

    }

}
```

最终题解
```java
class Solution {

    public List<List<String>> groupAnagrams(String[] strs) {


        int capicity = (int) (strs.length / 0.75f)+1;

        Map<String,List<String>> groups = new HashMap<>(capicity);

  

        for(String str: strs){
        
        //  相比于我自己的想法,AI给出了一个不同的见解
        // 我之前是用到了一个排序 总复杂度是 n * k * logk
       // AI 用时间换空间  搞了一个计数  复杂度为 n *  k  
	// 但是测试出来的效果一般,不如上面的排序,应该是 字符串长度问题
	
            int[] counts = new int[26];

            char[] chars = str.toCharArray();

            for(char ch:chars){

                counts[ch-'a']++;

            }

            StringBuilder key = new StringBuilder();

            for(int count:counts){

                key.append('#').append(count);

            }

  

            groups.computeIfAbsent(key.toString(),value->new ArrayList<>())

                        .add(str);

  

        }

        return new ArrayList<>(groups.values());

    }

}
```

##### 最长连续序列 (noll)
```java
class Solution {

    public int longestConsecutive(int[] nums) {

        // if 0 需要处理  

        // 这个我思来想去没思路,主要是不知道如何处理'连续'这个东西

  

        // 我想了一下,我主要是有以下想法

        // map保存 (k,v) = (head,tail)

        // 然后每次来一个新int ,就插到head前面,没有就默认new一个(int,int)

        // 上面这个思路我叫做前插, 如果保存(tail,head) ,那就得用后插

        // 这个满足 n的复杂度,但是我不种地如何处理 合并这个方法

  

        //我想的方法是搞另一个 链表,存储每一 head前面的一个head

    }

}
```


```java
class Solution {

    public int longestConsecutive(int[] nums) {

        // if 0 需要处理  

        // AI 的方法,用空间换时间

        // 我不保存head tail

        // 保存 (int,length) ,length是当前 区间的长度

        // 只要我在这个区间,我就知道这个区间长度

  

        int capacity = (int) (nums.length / 0.75f) + 1;

        Map<Integer, Integer> map = new HashMap<>(capacity);

        int maxlen = 0;

  

        for(int num:nums){

  

            if(map.containsKey(num)) continue;

            int llen = map.containsKey(num-1)?map.get(num-1):0;

            int rlen = map.containsKey(num+1)?map.get(num+1):0;

            int clen = llen+rlen+1;

            map.put(num,clen);

  

            // 更新,中间可以不管,边界一定更新

            map.put(num-llen,clen);

            map.put(num+rlen,clen);

		// 请你思考以下这里能不能把用过的 旧 的head 和tail删除
		// 也就是合并两个区间之后,我们只更新了大区间的头尾
		// 中间俩是没有删除的,为什么?
		// 原因在于 这个题会给出可能重复的数字,如果重复数字来了,我们要检测重复
		// 不过可以把值设置为null 省内存

  

            maxlen = Math.max(maxlen,clen);

  

        }

        return maxlen;

  

    }

}
```


```java

// 最简单的解法,直接用set+hash

class Solution {

    public int longestConsecutive(int[] nums) {

        int capacity = (int) (nums.length / 0.75f) + 1;

        Set<Integer> set = new HashSet<>(capacity);

        int maxlen = 0;

  

        for(int num:nums){

            set.add(num);

        }

  

        for(int num: set){

            if(set.contains(num-1))continue;

            int length = 1;

            while(set.contains(++num)){

                length++;

            }

            maxlen = Math.max(maxlen,length);
			
		// 这里可以加上一个剪枝操作,爽爽爽	
	    if(maxlen>nums.length/2) return maxlen;
	    

        }

        return maxlen;

    }

}
```


#### 双指针

##### 移动零

```java
class Solution {

    public void moveZeroes(int[] nums) {

        // 秒了

        // 快慢指针

        // 快指针遇到0就跳过

        // 慢指针重新对非0赋值

        //快指针到头,剩下全赋值0

  

        int slow = 0;

        for(int fast = 0; fast<nums.length;fast++){

            if(nums[fast]==0){

                continue;

            }

            if(slow != fast){

                nums[slow] = nums[fast];

            }

            slow++;

        }

  

        for(;slow<nums.length;slow++){

            nums[slow] = 0;

        }

  

    }

}
```

##### 小插曲笑死我了

原来我们可以搞一个static重新构造,去刷榜
这里主要用到的就是一个  JIT 即时编译, 先static执行,搞成热点代码,再调用函数,开始执行



##### 盛最多水的容器

```java
class Solution {

    public int maxArea(int[] height) {

        // 上来没啥思路

  

        // 但是可以看到,越远,是结果的概率更高

        // 所以我们这道题先用双指针试试

        // letf right

        // 我们每次找 都找比上个高的

        // 复杂度为 n

        // 秒杀

        // 这道题与接雨水不同,道理就在这里

  

        int left = 0,right = height.length-1,max =0;;

        while(left<right){

            int lh = height[left];

            int rh = height[right];

            int s = 0,gap=right-left;

              //  这道题唯一需要考虑的地方就在这里,让谁移动
              // 这里很好理解的

            if(lh<=rh){

                s = lh*gap;

                left++;

            }else{

                s = rh*gap;

                right--;

            }

            max = Math.max(max,s);

        }

        return max;

    }

}
```



##### 三数之和


```java
class Solution {

    public List<List<Integer>> threeSum(int[] nums) {

        // 对于这种题,刚开始没思路肯定的

        // 不妨先想象twoSum

        // 肯定用双指针,对吧.找到结果直接两个指针都移动,找下一个

        // 那么threeSum是不是就可以是twoSum的结果是 three_num

        // 至少苯方法这道题解出来了

        // 在for里面套一层双指针, n 平方复杂度,可以接受

        // 怎么优化呢,很容易想到,上面加for就多了很多重复工作

        // 我们思考一下,应该没有 nlogn方法吧,那就用上面方法得了

	//   下面是错误示例,错的地方太多了,太典型了

        List<List<Integer>> result = new ArrayList<>();

  

        Arrays.sort(nums);

  

        for(int i = 0;i<nums.length-2;i++){

            int left = i+1,right = nums.length-1;

            while(left<right){

                // 我们先去重

                if(nums[left]==nums[left+1]){

                    left++;

                    continue;

                }

  

                if(nums[right] == nums[right-1]){

                    right--;

                    continue;

                }

  
  

                int sum = nums[left]+nums[right]+nums[i];

  

                if(sum==0){

                    result.add(Arrays.asList(nums[i],nums[left],nums[right]));

                    right--;

                    left++;

                }else if(sum>0){right--;}

                else{left++;}

            }

        }

        return result;

    }

}
```


```java
class Solution {

    public List<List<Integer>> threeSum(int[] nums) {

        // 对于这种题,刚开始没思路肯定的

        // 不妨先想象twoSum

        // 肯定用双指针,对吧.找到结果直接两个指针都移动,找下一个

        // 那么threeSum是不是就可以是twoSum的结果是 three_num

        // 至少苯方法这道题解出来了

        // 在for里面套一层双指针, n 平方复杂度,可以接受

        // 怎么优化呢,很容易想到,上面加for就多了很多重复工作

        // 我们思考一下,应该没有 nlogn方法吧,那就用上面方法得了

  

        List<List<Integer>> result = new ArrayList<>();

  

        Arrays.sort(nums);

  

        for(int i = 0;i<nums.length-2;i++){

            // 剪枝叶

            if(nums[i]>0)break;

            if(i>0 && nums[i] == nums[i-1])continue;

  

            int left = i+1,right = nums.length-1;

            while(left<right){

                // 绝对不能在这里去重,会丢掉 0,0,0这种情况

  

                int sum = nums[left]+nums[right]+nums[i];

  

                if(sum==0){

                    result.add(Arrays.asList(nums[i],nums[left],nums[right]));

                    right--;

                    left++;

                    while(left<right && nums[left] == nums[left-1]){

                        left++;

                    }

                    while(left<right && nums[right]==nums[right+1]){

                        right--;

                    }

                }else if(sum>0){right--;}

                else{left++;}  

            }

        }

        return result;

    }

}
```


##### 接雨水

再做接雨水,我感觉好简单

```java
class Solution {

    public int trap(int[] height) {

  

        // 我觉得用双指针很简单

        // left 和 right

        // 取低的为 height_gap

        // 然后把长度×height_gap是所有格子 sum

        // 然后把左右指针低的换next

        // 继续计算 height_gap(不过要记得减去上一个gap,因为已经计算过了)

        // 重复过程

        // 最后把柱子求和 得到 add(这里多减了,比如如图,有的柱子超出部分没碰到水也被减去了)

        // sum - add 得到结果

        // 上面思路是有问题的

        // 所以应该把计算雨水,搞到每次移动中,这就对了

        // 具体意思就是:

        int[] gaps = new int[height.length];

  

        int left = 0 ,right = height.length-1, sum = 0;

  

        while(left<right){

            int lv = height[left],rv = height[right];

            int level = Math.min(lv,rv);

            //int len = right-left+1;

            //int sum = gap*len;

            //for 遍历中间的格子,计算出add

  

            // 但是我们能可以换个方法,我们用时间换空间

            // 上面for里面的复杂度应该是logk,在坏情况退化到 n

            // 所以我们用时间换空间,用gaps

  

            int last_v,last;

            if(lv>rv){

                last = right;

                last_v = rv;

                gaps[right] = level;

                while(left<right && height[--right]<= last_v){

                    gaps[right] = level;

                }

            }else{

                last = left;

                last_v = lv;

                gaps[left] = level;

                while(left<right && height[++left] <= last_v){

                    gaps[left] = level;

                }

  

            }

        }

        for(int i = 0;i<height.length;i++){

                sum += gaps[i]>height[i]?gaps[i] - height[i]:0;

        }

        return sum;          

    }

}
```



更好的题解
```java
//其实根本不需要自己去额外维护一个数组,我们当时懒得写
// 思路很简单还很好做

class Solution {

    public int trap(int[] height) {

        int left=0,right = height.length-1;

        int lmax = height[left];

        int rmax = height[right];

        int water = 0;

  

        while(left<right){

            if(lmax<=rmax){

                left++;

                lmax = Math.max(lmax,height[left]);

                water += lmax - height[left];

            }else{

                right--;

                rmax = Math.max(rmax,height[right]);

                water += rmax-height[right];

            }

        }

        return water;

    }

}
```



#### 滑动窗口


##### 无重复字符的最长字串(习惯不好 )

```java
class Solution {

    public int lengthOfLongestSubstring(String s) {

        // 这一眼左右指针滑动窗口

        int left = 0, right = 0;

        char[] chars = s.toCharArray();

        int max = 0;

        Set<Character> set = new HashSet<>();

        while(right<s.length()){

            char rc = chars[right];

          while(set.contains(rc)){

            set.remove(chars[left]);

            left++;

          }

          set.add(rc);

          right++;

          max = Math.max(max,set.size());

        }

        return max;

    }

}
```


AI给的奇思妙想

```java
class Solution {

    public int lengthOfLongestSubstring(String s) {

        Map<Character,Integer> map = new HashMap<>();

        int left = 0;

        int max = 0;

  

        for(int right =0;right<s.length();right++){

            char c = s.charAt(right);

  

            if(map.containsKey(c)){

                //这个方法很难想明白这里会有问题

                // 就是 last_index可能比目前的left要小2

                int index = map.get(c);

                if(left<=index)left = index+1;

            }

  

            map.put(c,right);

            max = Math.max(max,right-left+1);

        }

        return max;

  

    }

}
```


##### 找到字符串所有字母的异位词

```java
class Solution {

    public List<Integer> findAnagrams(String s, String p) {

        // 依旧一眼滑动窗口

        List<Integer> result = new ArrayList<>();

  

        // 先处理 goal

        Map<Character,Integer> goal = new HashMap<>();

        Map<Character,Integer> current = new HashMap<>();

        for(char c: p.toCharArray()){

            goal.put(c, goal.getOrDefault(c, 0) + 1);

        }

  

        // 滑动窗口

        int left = 0;

        for(int right = 0;right<s.length();right++){

            // 加入窗口,goal有而且数量没超,直接加入

            char c = s.charAt(right);

            current.put(c,current.getOrDefault(c,0)+1);

            // 不合法处理

            // goal里面有,或者数量超了

            // 直接从左边开始移动去除

            if(right-left+1>p.length()){

                char remove = s.charAt(left);

                int count = current.get(remove)-1;

                if(count==0){current.remove(remove);}

                else{current.put(remove,count);}

                left++;

            }

  

            // 合法处理

            // 记录当前left位置

            // current去除,left++

            if (right - left + 1 == p.length()

                    && current.equals(goal)) {

                result.add(left);

            }

        }

        return result;

    }

}
```

```java
class Solution {

    public List<Integer> findAnagrams(String s, String p) {

        List<Integer> result = new ArrayList<>();

  

        int[] goal = new int[26];

        int[] current = new int[26];

  

        int left=0,len = 0,glen = p.length();

  

        for(char c: p.toCharArray()){

            goal[c-'a']++;

        }

        for(int right=0;right<s.length();right++){

            // 加入

            char c = s.charAt(right);

            current[c-'a']++;

            len++;

  

            if(len>glen){

                char remove = s.charAt(left);

                current[remove-'a']--;

                len--;

                left++;

            }

  

            if(len==glen && Arrays.equals(goal,current)){

                result.add(left);

            }

        }

        return result;

    }

  

}
```



#### 子串

##### 和为k的子数组

```java
class Solution {

    public int subarraySum(int[] nums, int k) {

        // 这个题实际上可以叫做动态规划

        // 我们使用类似排列树

        // 但是这样复杂度就不能保障了

        // 这种情况不适合动态规划

  

        // 这种情况实际是子串等

        // 我好像没啥思路了

        // 刚才忽略了一个条件,就是"连续"

        // 那直接上前缀和

  

         int capacity =(int) ((nums.length + 1) / 0.75f) + 1;

         Map<Integer,Integer> map = new HashMap<>(capacity);

         map.put(0,1);

         int pre_count = 0;

         int result = 0;

  

         for(int num: nums){

            pre_count += num;

            // 计算结果

            result += map.getOrDefault(pre_count-k,0);

            map.put(pre_count,map.getOrDefault(pre_count,0)+1);

         }

         return result;

    }
```


#####  滑动窗口最大值(noll)

自己的思路复杂度太高了,不好

```java
class Solution {

    public int[] maxSlidingWindow(int[] nums, int k) {

        // 其实我太会做这道题

        // 但我有点印象,用什么单调栈,最小栈啥的

        //  1 3 -1

        // 好,我有思路了

        // 直接把k个元素入栈,然后出栈一个,入栈一个,取top

         Stack<Integer> stack = new Dequue<>();

         Stack<Integer> temp = new Dequue<>();

         int[] result = new int[nums.length-k+1];

         int ref = 0;

         for(int num : num){

            if(stack.size() == k){

                result[ref] = stack.peek()

                //这里拿出来

            }

            while(stack.peek()>num){

                // 这里放进去

            }

  

         }

    }

}

```

AI help

```java
class Solution {

    public int[] maxSlidingWindow(int[] nums, int k) {

        int[] result = new int[nums.length - k + 1];

        // 这道题用双端队列

        // 思路

        // 每次入队一个,就去把所有比他小的扔了

  

        Deque<Integer> queue = new ArrayDeque<>();

        for(int i = 0; i< nums.length;i++){

            // 清理旧元素

            while(!queue.isEmpty() && queue.peekFirst() <= i-k){

                queue.pollFirst();    

            }

            // 加上新元素

            while(!queue.isEmpty() && nums[queue.peekLast()]<=nums[i] ){

                queue.pollLast();

            }

            queue.offerLast(i);

  

            // 存结果

            if(i>=k-1){

                result[i-k+1] = nums[queue.peekFirst()];

            }

        }

        return result;

  

    }

}
```



##### 最小覆盖串(review)


自己写的坠机了,出错有很多地方,而且,还有: 题目没说只有小写
```java
class Solution {

    public String minWindow(String s, String t) {

        // 我感觉这道题就是一个滑动窗口

        // 就类似滑动窗口做就行了

        // O(n) 把 t 存入 map

        // O(m) 处理s的滑动窗口

        // 唯一麻烦的就是需要自己去重写equals?

        // 但是更好的方法就是我们不用map,用数组,因为题目只有小写字母

  

        int[] need = new int[26];

        int[] window = new int[26];

  

        for(char c : t.toCharArray()){

            need[c-'a']++;

        }

  

        int left = 0, min = Integer.MAX_VALUE,cur = 0;

        for(int right = 0;right<s.length();right++){

            // 进入窗口

            char c = s.charAt(right);

            window[c-'a']++;

  

            // 太小的直接不要

            if(right-left<t.length()){

                continue;

            }

            //

            while(left<right){

                boolean flag = true;

                for(int i =0;i<26;i++){

                    if(need[i]>window[i]){

                        flag = false;

                        break;

                    }

                }

  

                if(flag){

                    cur = Math.min(min,right-left+1)==min?cur:left;

                    window[s.charAt(left)-'a']--;

                    left++;

                }

            }    

        }

        return s.substring(cur,cur+min+1);

    }

}
```

按照我们的思路正确修改
```java

class Solution {

    public String minWindow(String s, String t) {

        int[] need = new int[128];

        int[] window = new int[128];

  

        for (char c : t.toCharArray()) {

            need[c]++;

        }

  

        int left = 0;

        int bestStart = -1;

        int bestLength = Integer.MAX_VALUE;

  

        for (int right = 0; right < s.length(); right++) {

            char add = s.charAt(right);

            window[add]++;

  

            // 窗口合法时持续缩小

            while (covers(window, need)) {

                int currentLength = right - left + 1;

  

                if (currentLength < bestLength) {

                    bestLength = currentLength;

                    bestStart = left;

                }

  

                char remove = s.charAt(left);

                window[remove]--;

                left++;

            }

        }

  

        if (bestStart == -1) {

            return "";

        }

  

        return s.substring(

                bestStart,

                bestStart + bestLength

        );

    }

  

    private boolean covers(int[] window, int[] need) {

        for (int i = 0; i < 128; i++) {

            if (window[i] < need[i]) {

                return false;

            }

        }

  

        return true;

    }

}
```

但是其实还有好方法,我们使用了两个数组,能不能用一个? 
其实是可以用一个missing去检测.这种耗费资源更少的方法


#### 普通数组

##### 最大子数组和
```java
class Solution {

    public int maxSubArray(int[] nums) {

        //  我想用前缀和去做,但是发现一些问题

        // 我找出最大前缀max和最小前缀min

        // 如果max的指针在min的前面,就不能用max-min了

        // 那么我们可以不可以搞一个前缀和 与 后缀和两个

        // 这俩是不是能互补

  

        //上面不行的画用分治很简单

        // 整体最大等于 左边最大/右边最大/跨越两个区间

        // 但是跨越两个区间好像不好写

        // 其实也算好写,直接用前缀和与后缀和

  

        int min = 0;

        int pre_sum = 0;

        int max_sum = Integer.MIN_VALUE;

        for(int num:nums){

            pre_sum+=num;

            max_sum = Math.max(max_sum,pre_sum-min);

            min = Math.min(min,pre_sum);

        }

        return max_sum;

    }

}
```



##### 合并区间(9/1)

```java
class Solution {

    public int[][] merge(int[][] intervals) {

        // 刚开始没有思路

        // 这道题的难点在于我怎么存,怎么查

        // 后来我想了一下,我打算用双端队列去做

        Deque<Integer> deque = new ArrayDeque<>();

        for(int[] nums:intervals){

            int left = nums[0];

            int right = nums[1];

            if(!deque.size()>1){

                deque.addLast(right);

                deque.addFirst(left);

                continue;

            }

            int last = deque.peekLast();

            int first = deque.peekFirst();

  

            // 这里根据 大小比较就有非常多种情况了

            // 感觉这个方法好复杂

        }

    }

}
```


```java
class Solution {

    public int[][] merge(int[][] intervals) {

  

        // 之前那个方法我觉得我能做出来 n d复杂度

        // 但是看了一下好像不行

  

        Arrays.sort(

            intervals,

            (a,b) -> Integer.compare(a[0],b[0])

        );

  

        List<int[]> merged = new ArrayList<>();

        for(int[] current: intervals){

            if(merged.isEmpty()){

                 merged.add(

                        new int[]{current[0], current[1]}

                );

                continue;

            }

  

            int[] last = merged.get(merged.size() - 1);

  

            if(current[0]>last[1]){

                 merged.add(

                        new int[]{current[0], current[1]}

                );

            }else{

                last[1] = Math.max(last[1],current[1]);

            }

        }

  

        return merged.toArray(new int[merged.size()][]);

    }

}
```


##### 轮转数组

直接跳过了

```java
class Solution {

    public void rotate(int[] nums, int k) {

        // 这个我记得是数学方法

        //  reverse三次就好了

        // 整体reverse一次

        // 在0~(k-1) reverse一次

        // 在k~(length-1) revers一次

    }

}
```

##### 除了自身以外数组的乘积

```java
class Solution {

    public int[] productExceptSelf(int[] nums) {

        // 这个挺好做的,一下就想到了

        // 0 的数量 对结果的影响很大

        int mul = 1;

        int zero_count = 0;

        int zero_bit = -1;

        int[] result = new int[nums.length];

  

        for(int i =0;i<nums.length;i++){

            if(nums[i] != 0){

                mul *= nums[i];

                continue;

            }

  

            zero_count++;

            if(zero_count > 1){

                break;

            }

            zero_bit = i ;

        }

  

        if(zero_count>1){

            return result;

        }

        if(zero_count == 1){

            result[zero_bit] = mul;

            return result;

        }

  

        for(int i=0;i<nums.length;i++){

            result[i] = mul / nums[i];

        }

        return result;

    }

}
```


```java
class Solution {

    public int[] productExceptSelf(int[] nums) {

        // 做了半天发现人家不让用除法

        //那么就是另外一种思路

        //从左到右边扫一遍 result变成前缀积

        //再从右到左扫一遍,result变成后缀积

        // 俩乘起来

        int len = nums.length;

        int[] result = new int[len];

  

        int pre_mul = 1;

        for(int i = 0; i<len;i++){

            result[i] =  pre_mul;

            pre_mul *= nums[i];

        }

        int suf_mul = 1;

        for(int i = len-1;i >=0;i--){

            result[i] *= suf_mul;

            suf_mul *= nums[i];

        }

        return result;

    }

}
```


##### 缺失的第一个正数(9/1)

```java
class Solution {

    public int firstMissingPositive(int[] nums) {

        // 这道题实在是没有思路

        // 后面我觉得也有方法

        // 使用 Hashset的 O(1) 查找方法

        Set<Integer> set = new HashSet<>(nums.length);

        for(int num : nums){

            set.add(num);

        }  

  

        for(int i = 1; i<= nums.length+1;i++){

            if(!set.contains(i)){

                return i;

            }

        }

        return -1;

    }

}
```


```java
class Solution {

    public int firstMissingPositive(int[] nums) {

        // 但是题目要求要使用 常数级别的 空间

        // 我们就采用一个原地hash

        int n = nums.length;

        for(int i =0;i < n;i++){

            //我们拿到 一个数字,我们就要给他找到一个位置

            //我们把 [0] 拿出来了,如果他要去i,那我们就让他去

            // 但是不能把i扔了,还要继续处理

            //只有当i出线了,我们才扔了
            
            // 这个判断条件有点离谱,第三个条件尤其需要注意,这是因为会出现重复的数字

            while(nums[i]>0 && nums[i]<=n && nums[nums[i]-1]!=nums[i]){

                int temp = nums[i];

                int target_index = nums[i] -1;

                nums[i] = nums[target_index];

                nums[target_index] = temp;

            }

        }

  

        for (int i = 0; i < n; i++) {

            if (nums[i] != i + 1) {

                return i + 1;

            }

        }

  

        return n + 1;

  

    }

}
```


#### 矩阵 (这些题见过就忘不了了)

##### 矩阵置 0 (empty)

empty标记表示我直接没做,跳过了
因为我觉得好做,所以直接跳了
但是还有很重要的东西,就是这道题我刚开始忘记了 标记行和标记列要单独考虑是否置为 0 

```java
class Solution {
    public void setZeroes(int[][] matrix) {
        int rows = matrix.length;
        int columns = matrix[0].length;

        boolean firstRowHasZero = false;
        boolean firstColumnHasZero = false;

        // 检查第一行
        for (int column = 0; column < columns; column++) {
            if (matrix[0][column] == 0) {
                firstRowHasZero = true;
                break;
            }
        }

        // 检查第一列
        for (int row = 0; row < rows; row++) {
            if (matrix[row][0] == 0) {
                firstColumnHasZero = true;
                break;
            }
        }

        // 使用第一行和第一列记录标记
        for (int row = 1; row < rows; row++) {
            for (int column = 1; column < columns; column++) {
                if (matrix[row][column] == 0) {
                    matrix[row][0] = 0;
                    matrix[0][column] = 0;
                }
            }
        }

        // 根据标记处理内部区域
        for (int row = 1; row < rows; row++) {
            for (int column = 1; column < columns; column++) {
                if (matrix[row][0] == 0
                        || matrix[0][column] == 0) {
                    matrix[row][column] = 0;
                }
            }
        }

        // 最后处理第一行
        if (firstRowHasZero) {
            for (int column = 0; column < columns; column++) {
                matrix[0][column] = 0;
            }
        }

        // 最后处理第一列
        if (firstColumnHasZero) {
            for (int row = 0; row < rows; row++) {
                matrix[row][0] = 0;
            }
        }
    }
}
```


##### 螺旋矩阵(waiting)

后面再写,需要review

```java
class Solution {
    public List<Integer> spiralOrder(int[][] matrix) {
        int rows = matrix.length;
        int columns = matrix[0].length;

        List<Integer> result = new ArrayList<>(rows * columns);

        int top = 0;
        int bottom = rows - 1;
        int left = 0;
        int right = columns - 1;

        while (top <= bottom && left <= right) {
            // 上边：从左到右
            for (int column = left; column <= right; column++) {
                result.add(matrix[top][column]);
            }
            top++;

            // 右边：从上到下
            for (int row = top; row <= bottom; row++) {
                result.add(matrix[row][right]);
            }
            right--;

            // 可能只剩一行，防止重复访问
            if (top <= bottom) {
                for (int column = right; column >= left; column--) {
                    result.add(matrix[bottom][column]);
                }
                bottom--;
            }

            // 可能只剩一列，防止重复访问
            if (left <= right) {
                for (int row = bottom; row >= top; row--) {
                    result.add(matrix[row][left]);
                }
                left++;
            }
        }

        return result;
    }
}
```


##### 旋转图像(empty)

明白思路,懒得写了 

```java
class Solution {
    public void rotate(int[][] matrix) {
        int n = matrix.length;

        // 沿主对角线转置
        for (int row = 0; row < n; row++) {
            for (int column = row + 1; column < n; column++) {
                int temp = matrix[row][column];
                matrix[row][column] = matrix[column][row];
                matrix[column][row] = temp;
            }
        }

        // 每一行左右反转
        for (int row = 0; row < n; row++) {
            int left = 0;
            int right = n - 1;

            while (left < right) {
                int temp = matrix[row][left];
                matrix[row][left] = matrix[row][right];
                matrix[row][right] = temp;

                left++;
                right--;
            }
        }
    }
}
```


##### 探索二维矩阵 2

```java
class Solution {

    public boolean searchMatrix(int[][] matrix, int target) {

        int m = matrix.length, n = matrix[0].length;

        int x = 0, y = n - 1;

        while (x < m && y >= 0) {

            if (matrix[x][y] == target) {

                return true;

            }

            if (matrix[x][y] > target) {

                --y;

            } else {

                ++x;

            }

        }

        return false;

    }

}
```


#### 链表

##### 相交链表(3)

```java
public class Solution {

    public ListNode getIntersectionNode(ListNode headA, ListNode headB) {

        // 这个思路很简单

        // A走完去走B

        // B走完去走A

        // 如果在结束之前 a 和 b所在节点 相等

        // 如果节点不是null,那就有香蕉


        ListNode a = headA, b =headB;

        while(a != b && (a.next !=null || b.next !=null)){
        
            if(a.next == null){a= headB;}else{a = a.next;}

            if(b.next == null){b = headA;}else{b = b.next;}

        }

        return a==b?a:null;

  

    }

}
```


##### 反转链表(3)

对于链表,我可能开始就学的不是很好,然后就一直不会写,
(虽然说链表很简单,但是我绝对是轻视它了,导致处理不好前几个节点)

```java
class Solution {

    public ListNode reverseList(ListNode head) {

        if(head == null || head.next == null) return head;

        ListNode curr = head;

        ListNode next = head.next;

        curr.next = null;

        ListNode temp;

        while(next != null){

            temp = curr;

            curr = next;

            next = next.next;

            curr.next =temp;

        }

        return curr;

  

    }

}
```


##### 回文链表(3)

```java

// 第三次做了,直接开背
class Solution {

    public boolean isPalindrome(ListNode head) {

        ListNode mid_node = midNode(head);

        ListNode head2 = reveser(mid_node);

        while(head2 != null){

            if(head.val != head2.val){

                return false;

            }

            head = head.next;

            head2 = head2.next;

        }

        return true;

    }

  
  

    ListNode midNode(ListNode head){

        ListNode fast = head, slow =  head;

        while(fast != null && fast.next != null){

            fast = fast.next.next;

            slow = slow.next;

        }

        return slow;

    }

  

    ListNode reveser(ListNode head){

        ListNode cur = head;

        ListNode pre = null;

        while(cur != null){

            ListNode nxt = cur.next;

            cur.next = pre;

            pre = cur;

            cur = nxt;

        }

        return pre;

    }

}
```


##### 环形链表 1 (3)

```java
public class Solution {

    public ListNode detectCycle(ListNode head) {

           ListNode fast = head, slow = head;

        while(fast != null && fast.next != null){

            fast = fast.next.next;

            slow = slow.next;

            // 最重要就是这一步,记住顺口溜

            // head slow 决定

            if(fast == slow){

                while(head != slow){

                    slow = slow.next;

                    head = head.next;

                }

                return slow;

            }

        }

        return null;

    }

}
```

```java
public class Solution {

    public boolean hasCycle(ListNode head) {

        ListNode fast = head, slow = head;

        while(fast != null && fast.next != null){

            fast = fast.next.next;

            slow = slow.next;

            if(fast == slow) return true;

        }

        return false;

    }

}
```
##### 环形连表2 (3)

```java
public class Solution {

    public ListNode detectCycle(ListNode head) {

        ListNode slow = head;

        ListNode fast = head;

        // 把自己的改了吧,还是AI写的好

        while (fast != null && fast.next != null) {

            slow = slow.next;

            fast = fast.next.next;

  

            if (slow == fast) {

                ListNode finder = head;

  

                while (finder != slow) {

                    finder = finder.next;

                    slow = slow.next;

                }

  

                return finder;

            }

        }

  

        return null;

    }

}
```


##### 合并两个有序链表

```java
class Solution {

    public ListNode mergeTwoLists(ListNode list1, ListNode list2) {

        // 这个很经典了

        ListNode head = new ListNode(-1);

        ListNode next= head;

        while(list1 != null && list2 != null){

            next.next = list1.val<=list2.val?list1:list2;

            next = next.next;

            if(next == list1){

                list1 = list1.next;

            }else{

                list2 = list2.next;

            }

        }

        if(list1 == null){

            next.next = list2;

        }else{

            next.next = list1;

        }

        return head.next;

    }

}
```


```java

//还有递归写法

class Solution {

    public ListNode mergeTwoLists(ListNode list1, ListNode list2) {

        if(list1 == null) return list2;

        if(list2 == null) return list1;

        if(list1.val<list2.val){

            list1.next = mergeTwoLists(list1.next,list2);

            return list1;

        }else{

            list2.next = mergeTwoLists(list1,list2.next);

            return list2;

        }

    }

}
```
##### 两数相加

```java
class Solution {

    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {

        // 这个题我有点不理解

        // 好像变成了一个 list ->num -> list

        // 我好像没有其他方法了

        // 因为不可能在一遍遍历下,能够进行运算的

        ListNode answer = new ListNode(-1);

        ListNode tail = answer;

        int carry = 0;

        while(!(l1 == null && l2 == null) ){

            int v1 = l1 == null ? 0: l1.val;

            int v2 = l2 == null ? 0: l2.val;

            int count = v1 + v2 + carry;

            carry = count / 10;

            count %= 10;

            tail.next = new ListNode(count);

            tail = tail.next;

            if(l1 != null) l1=l1.next;

            if(l2 != null) l2 = l2.next;

        }    

        if(carry == 1) tail.next = new ListNode(carry);

        return answer.next;

    }

}
```

```java

class Solution {

    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {

        ListNode dummy = new ListNode();

        ListNode cur = dummy;

        int carry = 0;

        while(l1 != null || l2 != null || carry != 0){

            int sum = carry;

            if(l1 != null){

                sum += l1.val;

                l1 = l1.next;

            }

            if(l2 != null){

                sum += l2.val;

                l2 = l2.next;

            }

            cur.next = new ListNode(sum%10);

            cur = cur.next;

            carry = sum/10;

        }

        return dummy.next;

    }

}
```

```java

//  用递归,还行
class Solution {

    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {

        return addTwo(l1,l2,0);

    }

  

    ListNode addTwo(ListNode l1, ListNode l2, int carry){

        if(l1 == null && l2 == null && carry == 0){

            return null;

        }

  

        int add = carry;

        if(l1 != null){

            add += l1.val;

            l1 = l1.next;

        }

        if(l2 != null){

            add += l2.val;

            l2 = l2.next;

        }

        return new ListNode(add %10, addTwo(l1,l2,add/10));

    }

}
```
##### 删除链表的倒数第n个节点

```java


class Solution {

    public ListNode removeNthFromEnd(ListNode head, int n) {

        // easy

        // 这个题过于简单,做一个快慢指针,然后跑一趟就OK了

        ListNode dummy = new ListNode(-1,head);

        ListNode fast = dummy, slow = dummy;

        while(n-->0){

            fast = fast.next;

        }

  

        while(fast.next != null){

            fast = fast.next;

            slow = slow.next;

        }

        slow.next = slow.next.next;

        return dummy.next;

    }

}
```


##### 两两交换链表中的节点(review)

```java
class Solution {
    public ListNode swapPairs(ListNode head) {
        ListNode dummy = new ListNode(-1, head);
        ListNode pre = dummy;

        while (pre.next != null && pre.next.next != null) {
            ListNode left = pre.next;
            ListNode right = left.next;

            // pre -> left -> right -> next
            // 变成
            // pre -> right -> left -> next

            pre.next = right;
            left.next = right.next;
            right.next = left;

            pre = left;
        }

        return dummy.next;
    }
}
```

```java
//迭代

class Solution {

    public ListNode swapPairs(ListNode head) {

        ListNode dummy = new ListNode(0,head);

        ListNode node0 = dummy;

        ListNode node1 = head;

        while(node1 != null && node1.next != null){

            ListNode node2 = node1.next;

            ListNode node3 = node2.next;

  
            // 牢记 0->2, 2->1, 1->3

            node0.next = node2;

            node2.next = node1;

            node1.next = node3;

  

            node0 = node1;

            node1 = node3;

        }

        return dummy.next;

    }

}
```


```java
// 这是递归
class Solution {

    public ListNode swapPairs(ListNode head) {

        if(head == null || head.next == null){

            return head;

        }

        ListNode node1 = head;

        ListNode node2 = node1.next;

        ListNode node3 = node2.next;

  

        node2.next = node1;

        node1.next = swapPairs(node3);

        return node2;

    }

}
```



##### K 个 一组翻转链表(这个得多复习两遍)

这个解法不是最优解,而且经常犯这种错误,就是太麻烦了
就是不太理解这道题是怎么换过来的

```java
class Solution {

    public ListNode reverseKGroup(ListNode head, int k) {

        ListNode dummy = new ListNode(-1);

        dummy.next = head;

        ListNode[] nodes = new ListNode[k];

        ListNode prev = dummy;

        while(true){

            boolean flag = true;

            ListNode temp = prev.next;

            for(int i=0;i<k;i++){

                if(temp == null){

                    flag = false;

                    break;

                }

                nodes[i] = temp;

                temp = temp.next;

            }

            if(!flag){break;}

            prev.next = nodes[k-1];

            nodes[0].next = nodes[k-1].next;

            for(int i = k-1;i> 0;i--){

                nodes[i].next = nodes[i-1];

            }

            prev = nodes[0];

  

        }

        return dummy.next;

    }

}
```


##### 随机链表的复制

```java
class Solution {

    public Node copyRandomList(Node head) {

        // 这个就很难绷

        // 其实用一个map就可以轻松解决

        // 先存入map

        // 构造的时候通过val取出来就ok了

        // 但是前提得是这道题的val不同

        // 但是很遗憾啊不能用

        // 那我们直接用Node当key

        if(head == null ) return null;

        Map<Node,Node> map = new HashMap<>();

        Node temp = head;

        while(temp != null){

            map.put(temp,new Node(temp.val));

            temp =  temp.next;

        }

        Node dummy = new Node(-1);

        Node curr = dummy;

        temp = head;

        while(temp != null){

            curr.next = map.get(temp);

            curr = curr.next;

            if(temp.random != null)curr.random = map.get(temp.random);

            temp = temp.next;

        }

        return dummy.next;

    }

}

```


```java

// 这个是个好方法
class Solution {

    public Node copyRandomList(Node head) {

        Node cur = head;

        while(cur != null){

            Node temp = new Node(cur.val);

            temp.next = cur.next;

            cur.next = temp;

            cur = cur.next.next;

        }

        cur = head;

        while(cur != null){

            // 这里后半句的next是指, 不要把旧的random给新链表

            if(cur.random != null)

            cur.next.random = cur.random.next;

            cur = cur.next.next;

        }

        cur = head;

        Node dummy = new Node(0);

        Node tail = dummy;

        while(cur != null){

            Node copy = cur.next;

            tail.next = copy;

            cur.next = copy.next;

            tail = tail.next;

            cur = cur.next;

        }

        return dummy.next;

    }

}
```
##### 排序链表(没有完成进阶要求)

```java
class Solution {

    public ListNode sortList(ListNode head) {

        //这道题最省事的方法就是

        // 把数字提取出来,排序

        // 然后再赋值

        // 但是如果不取巧的话

        // 我们就只能用比较排序了

        // 最妙可以上希尔这样子?

         int length = 0;

  

        for (ListNode current = head;

                current != null;

                current = current.next) {

            length++;

        }

  

        int[] values = new int[length];

        ListNode current = head;

  

        for (int i = 0; i < length; i++) {

            values[i] = current.val;

            current = current.next;

        }

  

        Arrays.sort(values);

  

        current = head;

  

        for (int value : values) {

            current.val = value;

            current = current.next;

        }

  

        return head;

    }

}
```

```java


class Solution {

    public ListNode sortList(ListNode head) {

        if(head == null || head.next == null) return head;

  

        ListNode head2 = midNode(head);

        // 分治

        head = sortList(head);

        head2 = sortList(head2);

        // 合并

        return mergeTwoList(head,head2);

    }

  
  

    ListNode midNode(ListNode head){

        ListNode pre = head;

        ListNode slow = head;

        ListNode fast = head;

        while(fast != null && fast.next != null){

            pre = slow;

            slow = slow.next;

            fast = fast.next.next;

        }

        pre.next = null;

        return slow;  

    }

  

    ListNode mergeTwoList(ListNode l1,ListNode l2){

        ListNode dummy = new ListNode();

        ListNode cur = dummy;

        while(l1 != null && l2 != null){

            if(l1.val< l2.val){

                cur.next = l1;

                l1 = l1.next;

            }else{

                cur.next = l2;

                l2 = l2.next;

            }

            cur = cur.next;

        }

        if(l1 == null) cur.next = l2;

        if(l2 == null) cur.next = l1;

        return dummy.next;

    }

}
```
##### 合并K个升序链表(偷懒取巧的方法)

```java
class Solution {

    public ListNode mergeKLists(ListNode[] lists) {

        // 如果真要手动去合并,那真的写死人

        // 那种逻辑真得需要慢慢判断

        // 或者使用分治思想,两个两个合并,最后合并到一起

        // 但是,直接作弊,把数字读出来再排序创建放回

        PriorityQueue<Integer> pq = new PriorityQueue<>();

        for(ListNode node : lists){

            while(node != null){

                pq.offer(node.val);

                node = node.next;

            }

        }

        ListNode dummy = new ListNode(-1);

        ListNode tail = dummy;

        while(!pq.isEmpty()){

            tail.next = new ListNode(pq.poll());

            tail = tail.next;

        }

        return dummy.next;

    }

}
```

```java

class Solution {

    public ListNode mergeKLists(ListNode[] lists) {

        PriorityQueue<ListNode> pq = new PriorityQueue<>((a,b)->a.val-b.val);

        for(ListNode head: lists){

            if(head != null){

                pq.offer(head);

            }

        }

        ListNode dummy = new ListNode();

        ListNode cur = dummy;

        while(!pq.isEmpty()){

            ListNode temp  = pq.poll();

            if(temp.next != null) pq.offer(temp.next);

            cur.next = temp;

            cur = cur.next;

        }

        return dummy.next;

    }

}
```
##### LRU(需要额外做带过期时间的LRU)

```java
class LRUCache {

  

    int capacity;

    Node head, tail;

    HashMap<Integer, Node> map;

  

    public LRUCache(int capacity) {

        this.capacity = capacity;

        head = new Node(0,-1);

        tail = new Node(0,-1);

        head.next = tail;

        tail.prev = head;

        map = new HashMap<Integer, Node>();

    }

  

    // 就拿到,然后放到队首

    public int get(int key) {

        if (map.containsKey(key)) {

            Node n = map.get(key);

            resetToHead(n);

            return n.value;

        }

        return -1;

    }

  
  
  

    // 没有就加,有就改

    public void put(int key, int value) {

        if (map.containsKey(key)) {

            Node n = map.get(key);

            n.value = value;

            resetToHead(n);

            return;

        }

        // 没有就加

        Node n = new Node(key, value);

        map.put(key, n);

        addHead(n);

        if (map.size() > capacity) {

            Node rubbish = tail.prev;

            map.remove(rubbish.key);

            rubbish.prev.next = tail;

            tail.prev = rubbish.prev;

            rubbish.prev = null;

            rubbish.next = null;

        }

  

    }

  

    private void addHead(Node n) {

        n.next = head.next;

        n.next.prev = n;

        n.prev = head;

        head.next = n;

    }

  

    private void resetToHead(Node n) {

        n.prev.next = n.next;

        n.next.prev = n.prev;

        addHead(n);

    }

  
  

}

  

class Node {

    int key;

    int value;

    Node next;

    Node prev;

  

    public Node(int key, int value) {

        this.key = key;

        this.value = value;

    }

}
```


#### 二叉树

##### 中序遍历 (太简单了)

```java
class Solution {

  

    List<Integer> result;

  

    public List<Integer> inorderTraversal(TreeNode root) {

        result = new ArrayList<>();

        dfs(root);

        return result;

    }

  

    public void dfs(TreeNode node){

        if(node == null) return;

        dfs(node.left);

        result.add(node.val);

        dfs(node.right);

    }

}
```

##### 二叉树的最大深度

```java
class Solution {
    public int maxDepth(TreeNode root) {
        if (root == null) {
            return 0;
        }

        return Math.max(
                maxDepth(root.left),
                maxDepth(root.right)
        ) + 1;
    }
}
```


##### 翻转二叉树

```java
class Solution {

    public TreeNode invertTree(TreeNode root) {

        reverse(root);

        return root;

    }

  

    void  reverse(TreeNode root){

        if(root == null) return;

        TreeNode temp = root.left;

        root.left = root.right;

        root.right = temp;

        reverse(root.left);

        reverse(root.right);

    }

}
```



##### 对称二叉树

```java
class Solution {

    public boolean isSymmetric(TreeNode root) {

        return check(root.left,root.right);

    }

    public boolean check(TreeNode left,TreeNode right){

        if(left == null || right == null ){

            if(!(left==null && right == null)){

                return false;

            }

            return true;

        }

  

        return left.val == right.val && check(left.right,right.left) &&

                    check(left.left,right.right);

  

    }

}
```


##### 二叉树的直径

```java
class Solution {

  

    int max;

    public int diameterOfBinaryTree(TreeNode root) {

        // 左子树深度

        // 右子树深度

        // 返回左子树的直径,右子树的直径, root的直径中最大的

  

        // 但是上卖的递归,会重复遍历子树很多遍

        // 如果用map存储起来直径会不会更好.

        max = 0;

        depth(root);

        return max;

  

    }

    public int depth(TreeNode root){

        if(root == null){

            return 0 ;

        }

        int ld = depth(root.left);

        int rd = depth(root.right);

        // 这里没有加1 是因为 节点数量也就是深度 和 直径的计算方式不同

        max = Math.max(max,ld+rd);

        return Math.max(ld,rd)+1;

    }

}

```


##### 层序遍历

```java
class Solution {

    public List<List<Integer>> levelOrder(TreeNode root) {

       List<List<Integer>> result = new ArrayList<>();

        if(root == null) return result;

       Queue<TreeNode> queue = new LinkedList<>();

       queue.offer(root);

  

       while(!queue.isEmpty()){

            int size = queue.size();

            List<Integer> temp = new ArrayList<>();

            result.add(temp);

            while(size-->0){

                TreeNode node = queue.poll();

                temp.add(node.val);

                if(node.left != null) queue.offer(node.left);

                if(node.right != null) queue.offer(node.right);

            }

       }

       return result;

    }

}

```


##### 有序数组转二叉树

```java
class Solution {

    int[] data;

    public TreeNode sortedArrayToBST(int[] nums) {

        data = nums;

        return addNode(0,nums.length-1);

    }

  

    public TreeNode addNode(int left,int right){

        int mid = (left + right) / 2;

        TreeNode node = new TreeNode(data[mid]);

        if(left>right) return null;

        if(left==right){

            return node;

        }

        node.left = addNode(left,mid-1);

        node.right = addNode(mid+1,right);

        return node;

    }

}
```

```java
class Solution {

  

    private long pre = Long.MIN_VALUE;

  

    public boolean isValidBST(TreeNode root) {

        if(root == null) return true;

  

        if(!isValidBST(root.left)){

            return false;

        }

  

        if(root.val<= pre){

            return false;

        }

        pre = root.val;

  

        if(!isValidBST(root.right)){

            return false;

        }

        return true;

    }

}
```


```java
class Solution {

    public boolean isValidBST(TreeNode root) {

        return dfs(root,Long.MIN_VALUE,Long.MAX_VALUE);

    }

  

    boolean dfs(TreeNode root, Long min, Long max){

        if(root == null) return true;

        long val = root.val;

        return val<max &&

                val > min &&

                dfs(root.left,min,val) &&

                dfs(root.right,val, max);

    }

}
```



##### 验证二叉搜索树

```java

class Solution {

    long last_val;

    public boolean isValidBST(TreeNode root) {

        last_val = Long.MIN_VALUE;

        // 最简单的方法,中序遍历必须是有序的

        return dfs(root);

    }

    public boolean dfs(TreeNode root){

        if(root == null) return true;

  

        boolean con1 = dfs(root.left);

        boolean con2 = root.val > last_val;

        last_val = root.val;

        boolean con3 = dfs(root.right);

        return con1 && con2 && con3;

    }

}
```



``
##### 二叉搜索树 第K小的数字

```java
class Solution {

    int K;

    int GOAL;

    int rank = 0;

    public int kthSmallest(TreeNode root, int k) {

        K = k;

        dfs(root);

        return GOAL;

    }

  

    public void dfs(TreeNode node){

        if(node == null) return;

  

        dfs(node.left);

        rank++;

        if(rank == K) {

            GOAL = node.val;

            return;

        }

        dfs(node.right);

    }

}
```


##### 二叉树的右视图

```java
class Solution {

    public List<Integer> rightSideView(TreeNode root) {

        // 如果从dfs这是根本不行的

        // 这得用bfs

        // 用bfs就和层序遍历没啥区别了

  

        List<Integer> result = new ArrayList<>();

        if(root == null) return result;

        Deque<TreeNode> queue = new LinkedList<>();

        queue.offer(root);

        while(!queue.isEmpty()){

            int size = queue.size();

            while(size-->0){

                TreeNode n = queue.poll();

                if(n.left != null) queue.offer(n.left);

                if(n.right != null) queue.offer(n.right);

                if(size == 0) result.add(n.val);

            }

        }

        return result;

    }

}
```

```java
class Solution {
	//这道题其实可以DFS, 优先右子树遍历, 如果遇到当前深度的第一个就加入结果集
    public List<Integer> rightSideView(TreeNode root) {
        List<Integer> result = new ArrayList<>();
        dfs(root, 0, result);
        return result;
    }

    private void dfs(
            TreeNode node,
            int depth,
            List<Integer> result) {

        if (node == null) {
            return;
        }

        // 当前深度第一次被访问
        if (depth == result.size()) {
            result.add(node.val);
        }

        // 必须先访问右子树
        dfs(node.right, depth + 1, result);
        dfs(node.left, depth + 1, result);
    }
}
```

```java
class Solution {

    // dfs试试

    int height;

    int maxh;

    List<Integer> result;

    public List<Integer> rightSideView(TreeNode root) {

        height = 0;

        maxh = 0;

        result = new ArrayList<>();

        dfs(root);

        return result;

    }

  

    void dfs(TreeNode root){

        if(root == null) return;

        height++;

        if(height>maxh){

            result.add(root.val);

            maxh = height;

        }

  

        dfs(root.right);

        dfs(root.left);

        height--;

    }

}
```
##### 二叉树展开为链表(没看懂第二种方法的最优解)


```java
class Solution {

    public void flatten(TreeNode root) {

        // 先序遍历一遍就能做

        // 但是题目要求用 原地

  

        // 那也是dfs

        //

        if(root == null) return;

        dfs(root);

  

    }

  

    public void dfs(TreeNode root){

        if(root == null) return;

        if(root.left == null){

            dfs(root.right);

            return;

        }

        if(root.right == null){

            root.right = root.left;

            root.left = null;

            dfs(root.right);

            return;

        }

        dfs(root.left);

        dfs(root.right);

        TreeNode n = root.left;

        while(n.right != null){

            n  = n.right;

        }

        n.right = root.right;

        root.right = root.left;

        root.left = null;

        return;

    }

}
```

```java

class Solution {

    // 使用头插法秒了

    TreeNode preNode;

    public void flatten(TreeNode root) {

        dfs(root);

    }

  

    void dfs(TreeNode root){

        if(root == null) return;

        dfs(root.right);

        dfs(root.left);

        root.left = null;

        root.right = preNode;

        preNode = root;

    }

}
```



```java
class Solution {
    public void flatten(TreeNode root) {
        TreeNode current = root;

        while (current != null) {
            if (current.left != null) {
                // 找到左子树最右侧节点
                TreeNode predecessor = current.left;

                while (predecessor.right != null) {
                    predecessor = predecessor.right;
                }

                // 原右子树接到左子树末尾
                predecessor.right = current.right;

                // 左子树移动到右边
                current.right = current.left;
                current.left = null;
            }

            current = current.right;
        }
    }
}
```

##### 前序中序构造二叉树

```java
class Solution {

  

    private int[] preorder;

    private int preorderIndex;

    private Map<Integer, Integer> inorderIndex;

  

    public TreeNode buildTree(int[] preorder, int[] inorder) {

        this.preorder = preorder;

        this.preorderIndex = 0;

  

        int capacity = (int) (inorder.length / 0.75f) + 1;

        inorderIndex = new HashMap<>(capacity);

  

        for (int i = 0; i < inorder.length; i++) {

            inorderIndex.put(inorder[i], i);

        }

  

        return dfs(0, inorder.length - 1);

    }

  

    private TreeNode dfs(int begin, int end) {

        if(begin>end) return null;

  

        int root_value  = preorder[preorderIndex++];

        TreeNode root = new TreeNode(root_value);

  

        int index = inorderIndex.get(root_value);

  

        root.left = dfs(begin,index-1);

        root.right = dfs(index+1,end);

        return root;

    }

}
```

##### 路径总和2(noll,不会写,抄AI)


```java
class Solution {

  

    private Map<Long, Integer> prefixCount;

    private long target;

  

    public int pathSum(TreeNode root, int targetSum) {

        prefixCount = new HashMap<>();

        target = targetSum;

  

        // 虚拟的根节点之前，前缀和为 0，出现一次

        prefixCount.put(0L, 1);

  

        return dfs(root, 0L);

    }

  
	// dfs这里手写,但是确实自己debug很慢,AI debug更快
    private int dfs(TreeNode root, long previousSum) {

        if(root == null){

            return 0;

        }

  

        long sum = previousSum +   root.val;

        int result = prefixCount.getOrDefault(sum-target,0);

  

        prefixCount.put(sum,prefixCount.getOrDefault(sum,0)+1);

        result += dfs(root.left,sum);

        result += dfs(root.right,sum);

  

        int count = prefixCount.get(sum);

        if(count == 1){

            prefixCount.remove(sum);

        }else{

            prefixCount.put(sum,count-1);

        }

        return result;

    }

}
```

```java
class Solution {

    private Map<Long, Integer> prefixCount;
    private long target;

    public int pathSum(TreeNode root, int targetSum) {
        prefixCount = new HashMap<>();
        target = targetSum;

        // 虚拟的根节点之前，前缀和为 0，出现一次
        prefixCount.put(0L, 1);

        return dfs(root, 0L);
    }

    private int dfs(TreeNode root, long previousSum) {
        if (root == null) {
            return 0;
        }

        long currentSum = previousSum + root.val;

        // 必须先统计，再把 currentSum 放进去
        int result = prefixCount.getOrDefault(
                currentSum - target,
                0
        );

        // 当前前缀和进入递归路径
        prefixCount.put(
                currentSum,
                prefixCount.getOrDefault(currentSum, 0) + 1
        );

        result += dfs(root.left, currentSum);
        result += dfs(root.right, currentSum);

        // 离开当前节点，进行回溯
        int count = prefixCount.get(currentSum) - 1;

        if (count == 0) {
            prefixCount.remove(currentSum);
        } else {
            prefixCount.put(currentSum, count);
        }

        return result;
    }
}
```


##### 二叉树的最近公共祖先

```java
class Solution {

    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {

        // 这个题确实没有好的思路

        // 如果直接用dfs

        if(root == null || root == p || root == q){

            return root;

        }

        TreeNode left = lowestCommonAncestor(root.left,p,q);

        TreeNode right = lowestCommonAncestor(root.right,p,q);

        if(left != null && right != null){

            return root;

        }

        return left  != null?left:right;

    }

}
```


##### 二叉树中的最大路径和

```java
class Solution {

    int max;

    public int maxPathSum(TreeNode root) {

        max = Integer.MIN_VALUE;

        dfs(root);

        return max;

    }

  

    int dfs(TreeNode root){

        if(root == null) return 0;

  

        int left_val = Math.max(0, dfs(root.left));

        int right_val  = Math.max(0, dfs(root.right));

        int sum = root.val + left_val + right_val;

        max = Math.max(max,sum);

        return root.val + Math.max(left_val,right_val);

    }

}
```



#### 图论

##### 岛屿数量

```java
class Solution {

  

    char[][] nums;

    public int numIslands(char[][] grid) {

        // 这道题我想用BFS,但是我感觉不太好用

        // 算了,管他好不好用,直接干了

        nums = grid;

        int result = 0;

        for(int i=0;i<grid.length;i++){

            for(int j=0;j<grid[0].length;j++){

               if(nums[i][j] == '1'){

                result++;

                bfs(i,j);

               }

            }

        }

        return result;

    }

  

    public void bfs(int i,int j){

        if(i<0 || j < 0 || i>=nums.length || j >= nums[0].length){

            return;

        }

        int num = nums[i][j];

        if(num== '0') return;

        nums[i][j] = '0';

        bfs(i-1,j);

        bfs(i+1,j);

        bfs(i,j-1);

        bfs(i,j+1);

    }

}
```


##### 腐烂的橘子

```java

class Solution {

  

    class Node {

        int i;

        int j;

  

        Node(int i, int j) {

            this.i = i;

            this.j = j;

        }

    }

  

    public int orangesRotting(int[][] grid) {

        Queue<Node> queue = new ArrayDeque<>();

  

        // 剩余新鲜橘子的数量

        int count = 0;

  

        // 所有初始腐烂橘子同时入队

        for (int i = 0; i < grid.length; i++) {

            for (int j = 0; j < grid[0].length; j++) {

                if (grid[i][j] == 1) {

                    count++;

                } else if (grid[i][j] == 2) {

                    queue.offer(new Node(i, j));

                }

            }

        }

  

        int minutes = 0;

  

        // 每轮循环代表一分钟

        while (!queue.isEmpty() && count > 0) {

            int size = queue.size();

  

            while (size-- > 0) {

                Node node = queue.poll();

  

                int i = node.i;

                int j = node.j;

  

                // 上

                if (i - 1 >= 0 && grid[i - 1][j] == 1) {

                    grid[i - 1][j] = 2;

                    count--;

                    queue.offer(new Node(i - 1, j));

                }

  

                // 下

                if (i + 1 < grid.length && grid[i + 1][j] == 1) {

                    grid[i + 1][j] = 2;

                    count--;

                    queue.offer(new Node(i + 1, j));

                }

  

                // 左

                if (j - 1 >= 0 && grid[i][j - 1] == 1) {

                    grid[i][j - 1] = 2;

                    count--;

                    queue.offer(new Node(i, j - 1));

                }

  

                // 右

                if (j + 1 < grid[0].length && grid[i][j + 1] == 1) {

                    grid[i][j + 1] = 2;

                    count--;

                    queue.offer(new Node(i, j + 1));

                }

            }

  

            minutes++;

        }

  

        return count == 0 ? minutes : -1;

    }

}
```


##### 课程表(抄AI的,还得重做)

```java

class Solution {

    public boolean canFinish(int numCourses, int[][] prerequisites) {

        // 这个题主要就是判断有没有环

        //

        List<Integer>[] graph = new List[numCourses];

        int[] indegree = new int[numCourses];

        for (int i = 0; i < numCourses; i++) {

            graph[i] = new ArrayList<>();

        }

        for (int[] prerequisite : prerequisites) {

            int course = prerequisite[0];

            int previous = prerequisite[1];

  

            // previous → course

            graph[previous].add(course);

            indegree[course]++;

        }

  

         Deque<Integer> queue = new ArrayDeque<>(numCourses);

          // 没有先修课程，可以直接学习

        for (int course = 0; course < numCourses; course++) {

            if (indegree[course] == 0) {

                queue.offer(course);

            }

        }

        int completed = 0;

        while (!queue.isEmpty()) {

            int course = queue.poll();

            completed++;

  

            for (int nextCourse : graph[course]) {

                indegree[nextCourse]--;

  

                if (indegree[nextCourse] == 0) {

                    queue.offer(nextCourse);

                }

            }

        }

        return completed == numCourses;

    }

}
```



#####  实现前缀树(review)


```java
class Trie {

    class Node{

        Node[] son = new Node[26];

        boolean end = false;

    }

    // 这个我确实不知道怎么实现

    // 肯定用多边图

    // a -> p -> p -> l -> e

    // 然后把第一个字母hash一下?

    // 找的时候按照第一个字母找

    private final Node root = new Node();

  

    public Trie() {

    }

    public void insert(String word) {

        Node cur = root;

        for(char c : word.toCharArray()){

             c -= 'a';

             if( cur.son[c] == null){

                cur.son[c] = new Node();

             }

             cur = cur.son[c];

        }

        cur.end = true;

    }

    public boolean search(String word) {

        return find(word) == 2;

    }

    public boolean startsWith(String prefix) {

        return find(prefix) != 0;

    }

  

    private int find(String word) {

        Node cur = root;

        for (char c : word.toCharArray()) {

            c -= 'a';

            if (cur.son[c] == null) {  

                return 0;

            }

            cur = cur.son[c];

        }

        return cur.end ? 2 : 1;

    }

  

}

  

/**

 * Your Trie object will be instantiated and called as such:

 * Trie obj = new Trie();

 * obj.insert(word);

 * boolean param_2 = obj.search(word);

 * boolean param_3 = obj.startsWith(prefix);

 */
```


#### 回溯

##### 全排列


```java
class Solution {

  

     List<List<Integer>> result;

     int[] nums;

     int length;

     boolean[] used;

  

    public List<List<Integer>> permute(int[] nums) {

        // 对于回溯法的题,我们就分为两种

        // 全排列 还是 子集树

        // 对于这两种方法是不同的实现

  

        result = new ArrayList<>();

        this.nums = nums;

        length = nums.length;

        used = new boolean[nums.length];

        backtrace(new ArrayList<Integer>());

        return result;

  

    }

  

    void backtrace(List<Integer> list){

  

        if(list.size() == length){

            result.add(new ArrayList<>(list));

            return;

        }

  

        for(int i = 0;i<length;i++){

            if(used[i]){continue;}

            used[i] = true;

            int num = nums[i];

            list.add(num);

            backtrace(list);

            list.remove(list.size()-1);

            used[i]= false;

        }

  

    }

}
```


##### 子集

```java
class Solution {

  

    List<List<Integer>> result;

    int[] nums;

  

    public List<List<Integer>> subsets(int[] nums) {

        // 这个是子集树,和排列树不一样

        // 左子树表示选,右子树表示不选

        result = new ArrayList<>();

        this.nums = nums;

        backtrace(new ArrayList<Integer>(),0);

        return result;

    }

  

    void backtrace(List<Integer> list,int j){

        if(j == nums.length){

            result.add(new ArrayList(list));

        }

        if(j>=nums.length) return;

  

        // 不要当前节点

        backtrace(list,j+1);

  

        // 要当前节点

        list.add(nums[j]);

        backtrace(list,j+1);

        list.remove(list.size() - 1);

  

    }

}
```


##### 电话号码

```java
class Solution {

  

    private static final String[] LETTERS = {

            "",

            "",

            "abc",

            "def",

            "ghi",

            "jkl",

            "mno",

            "pqrs",

            "tuv",

            "wxyz"

    };

    private List<String> result;

    private String digits;

  

    public List<String> letterCombinations(String digits) {

        // 这是一个排列树的变种

        // 主要是在初始化的时候我们要存储这个对应关系

        // backtrace中也就是

        result = new ArrayList<>();

        this.digits = digits;

  

        backtrace(0, new StringBuilder());

        return result;

    }

  

    void backtrace(int index,StringBuilder sb){

        if(index == digits.length()){

            result.add(sb.toString());

            return;

        }

  

        int digit = digits.charAt(index) - '0';

        String cur_str = LETTERS[digit];

        for(char c: cur_str.toCharArray()){

            sb.append(c);

            backtrace(index+1,sb);

            sb.deleteCharAt(sb.length()-1);

        }

    }

}
```


##### 组合总和

```java
class Solution {

  

    // 这道题是纯粹的子集树

  

    List<List<Integer>> list = new ArrayList<>();

    int GOAL;

    int[] Arr;

    public List<List<Integer>> combinationSum(int[] candidates, int target) {

        GOAL = target;

        Arr = candidates;

        backtrace(new ArrayList<>(),0,0);

        return list;

    }

  

    public void backtrace(List<Integer> temp,int add,int rank){

        if(add == GOAL) {

            list.add(new ArrayList<>(temp));

            return;

        }

        if(add > GOAL) return;

        for(int i = rank;i<Arr.length;i++){

            int a = Arr[i];

            temp.add(a);

            backtrace(temp,add+a,i);

            temp.remove(temp.size()-1);

        }

    }

}
```


##### 括号生成

```java
 
/*
这道题我真的不想写了,我感觉思路大概就是
backtrace的参数有左括号的数量和右括号的数量
括号够了就可以返回
括号不够
只要左括号比右括号多(低于最大值)
左右都可以放
否则,只能放左或者不能放
*/*

class Solution {

    private List<String> result;
    private char[] path;
    private int n;

    public List<String> generateParenthesis(int n) {
        this.n = n;
        result = new ArrayList<>();
        path = new char[2 * n];

        backtrack(0, 0, 0);

        return result;
    }

    private void backtrack(
            int index,
            int left,
            int right
    ) {
        if (index == path.length) {
            result.add(new String(path));
            return;
        }

        if (left < n) {
            path[index] = '(';
            backtrack(index + 1, left + 1, right);
        }

        if (right < left) {
            path[index] = ')';
            backtrack(index + 1, left, right + 1);
        }
    }
}
```


##### 单词搜索(先跳过了)


##### 分割回文串(先跳过了)



##### N皇后(先跳过了)





#### 二分查找

##### 搜索插入位置(经常搞不清楚这个逻辑,我也是醉了)

```java
class Solution {
    public int searchInsert(int[] nums, int target) {
        int left = 0;
        int right = nums.length;

        while (left < right) {
            int mid = left + (right - left) / 2;

            if (nums[mid] < target) {
                left = mid + 1;
            } else {
                right = mid;
            }
        }

        return left;
    }
}
```


##### 搜索二位矩阵

```java
// 这个很简单,先查每行第一个,binaryseacrh看看是那一行

        // binarySearch 看看是哪一个

        // 问题是,我是不是把每列第一个拿出来才能二分查找
        
        // g高级做法,直接把二维看作是一维数组
        class Solution {

    public boolean searchMatrix(int[][] matrix, int target) {

        int rows = matrix.length;

        int cols = matrix[0].length;

        int size = rows * cols;

  

        int left = 0;

        int right = size;

  

        while (left < right) {

            int middle = left + (right - left) / 2;

  

            int row = middle / cols;

            int col = middle % cols;

            int value = matrix[row][col];

  

            if (value < target) {

                left = middle + 1;

            } else {

                right = middle;

            }

        }

  

        if (left == size) {

            return false;

        }

  

        int row = left / cols;

        int col = left % cols;

  

        return matrix[row][col] == target;

    }

}
```


##### 在排序数组中找第一个和最后一个

```java
class Solution {

    public int[] searchRange(int[] nums, int target) {

        // 这道题最简单的思路就是

        // 二分查找,然后双while

        // 但是很显然,不满足题意, 是 n 复杂度的

        // 其实也很简单

        // 就是二分查找找 target 和 target+1

        // 这里绝对不能用java的binarySearch,这个api在有多个target不保证index

        int left = binarySearch(nums,target);

        if(left == nums.length || nums[left] != target){

            return new int[]{-1,-1};

        }

        int right = binarySearch(nums,target+1)-1;

        return new int[]{left,right};

    }

  

    int binarySearch(int[] nums,int target){

        int left = 0;

        int right = nums.length;

        while(left<right){

            int mid = (right-left)/2+left;

            int number = nums[mid];

            if(number<target){

                left = mid + 1;

            }else{

                right = mid;

            }

        }

        return left;

    }

}
```


##### 搜索旋转排序数组(noll)


```java
class Solution {
    public int search(int[] nums, int target) {
        int left = 0;
        int right = nums.length;
        int last = nums[nums.length - 1];

        while (left < right) {
            int mid = left + (right - left) / 2;
            int number = nums[mid];

            boolean targetInRight = target <= last;
            boolean numberInRight = number <= last;

            if (targetInRight == numberInRight) {
                // target 和 number 在旋转点同一侧
                // 可以按照普通二分比较
                if (number < target) {
                    left = mid + 1;
                } else {
                    right = mid;
                }
            } else if (targetInRight) {
                // target 在右侧小数区
                // number 在左侧大数区
                // target 一定在 mid 的右边
                left = mid + 1;
            } else {
                // target 在左侧大数区
                // number 在右侧小数区
                // target 一定在 mid 的左边
                right = mid;
            }
        }

        if (left < nums.length && nums[left] == target) {
            return left;
        }

        return -1;
    }
}
```



##### 寻找旋转数组的最小值

```java

class Solution {

    public int findMin(int[] nums) {

        int length = nums.length;

        int lear = nums[length-1];

        int left = 0;

        int right = length - 1;

        while(left<right){

            int mid = (right -left)/2+left;

            if(nums[mid] > nums[right]){

                left = mid + 1;

            }else{

                right = mid;

            }

        }

        return nums[right];

    }

}
```


##### 寻找两个正序数组的中位数(未完全满足要求)

```java
class Solution {

    public double findMedianSortedArrays(int[] nums1, int[] nums2) {

        // 这个题,我说他是链表题都不为过

        // 就拿两个指针,跟合并链表似的

        // 找出固定数字

        // 分奇偶
        
        // 但是这个方法不满足题意,没有完成 log N 的复杂度

        int length = nums1.length + nums2.length;

        int i = 0;

        int j = 0;

        int previous = 0;

        int current = 0;

        for (int index = 0; index <= length / 2; index++) {

            previous = current;

  

            if (i < nums1.length&& (j >= nums2.length || nums1[i] <= nums2[j])) {

                current = nums1[i];

                i++;

            } else {

                current = nums2[j];

                j++;

            }

        }

        if (length % 2 == 1) {

            return current;

        }

  

        return (previous + current) / 2.0;

  

    }

}
```


##### 寻找两个正序数组的中位数(这道题太难了,mark一下,只会不符合题目的解答)

```java

class Solution {

    public double findMedianSortedArrays(int[] nums1, int[] nums2) {

        // 这个题,我说他是链表题都不为过

        // 就拿两个指针,跟合并链表似的

        // 找出固定数字

        // 分奇偶

        int length = nums1.length + nums2.length;

        int i = 0;

        int j = 0;

        int previous = 0;

        int current = 0;

        for (int index = 0; index <= length / 2; index++) {

            previous = current;

  

            if (i < nums1.length&& (j >= nums2.length || nums1[i] <= nums2[j])) {

                current = nums1[i];

                i++;

            } else {

                current = nums2[j];

                j++;

            }

        }

        if (length % 2 == 1) {

            return current;

        }

  

        return (previous + current) / 2.0;

  

    }

}
```

```java

class Solution {
    public double findMedianSortedArrays(int[] nums1, int[] nums2) {
        int length = nums1.length + nums2.length;
        int rank = length / 2;

        int[] lnums;
        int[] rnums;

        if (nums1.length <= nums2.length) {
            lnums = nums1;
            rnums = nums2;
        } else {
            lnums = nums2;
            rnums = nums1;
        }

        int left = 0;
        int right = lnums.length;

        // 寻找最小的分割位置：
        // rnums 左侧最大值 <= lnums 右侧最小值
        while (left < right) {
            int mid = left + (right - left) / 2;

            // lnums 左边取 mid 个
            int cut1 = mid;

            // rnums 左边取 rank - mid 个
            int cut2 = rank - mid;

            int right1 = cut1 == lnums.length
                    ? Integer.MAX_VALUE
                    : lnums[cut1];

            int left2 = cut2 == 0
                    ? Integer.MIN_VALUE
                    : rnums[cut2 - 1];

            if (left2 > right1) {
                // lnums 左边取少了
                left = mid + 1;
            } else {
                // 当前分割可能正确，或者 lnums 左边取多了
                right = mid;
            }
        }

        // 循环结束后，left 才是最终分割位置
        int cut1 = left;
        int cut2 = rank - left;

        int left1 = cut1 == 0
                ? Integer.MIN_VALUE
                : lnums[cut1 - 1];

        int right1 = cut1 == lnums.length
                ? Integer.MAX_VALUE
                : lnums[cut1];

        int left2 = cut2 == 0
                ? Integer.MIN_VALUE
                : rnums[cut2 - 1];

        int right2 = cut2 == rnums.length
                ? Integer.MAX_VALUE
                : rnums[cut2];

        int lmax = Math.max(left1, left2);
        int rmin = Math.min(right1, right2);

        if (length % 2 == 0) {
            return (lmax + rmin) / 2.0;
        } else {
            return rmin;
        }
    }
}
```


#### 栈

##### 有效的括号


```java
class Solution {

    public boolean isValid(String s) {

  

        if(s.length() %2 != 0) return false;

  

        Deque<Character> stack = new ArrayDeque<>();

        for(char c: s.toCharArray()){

            if(c == '(' || c == '{' || c == '['){

                stack.push(c);

            }else{

                if(stack.isEmpty()) return false;

  

                char top = stack.pop();

                if (c == ')' && top != '(') return false;

                if (c == ']' && top != '[') return false;

                if (c == '}' && top != '{') return false;

  

            }

        }

        return stack.isEmpty();

  

    }

  

}
```


##### 最小栈

```java
class MinStack {

  

    Deque<Integer> stack;

    Deque<Integer> min;

  
  

    public MinStack() {

        stack = new ArrayDeque<>();

        min = new ArrayDeque<>();

    }

    public void push(int value) {

        if(stack.isEmpty()){

            min.push(value);

        }

        min.push(Math.min(value,min.peek()));

        stack.push(value);

    }

    public void pop() {

        stack.pop();

        min.pop();

    }

    public int top() {

        return stack.peek();

    }

    public int getMin() {

        return min.peek();

    }

}

  

/**

 * Your MinStack object will be instantiated and called as such:

 * MinStack obj = new MinStack();

 * obj.push(value);

 * obj.pop();

 * int param_3 = obj.top();

 * int param_4 = obj.getMin();

 */
```


##### 字符串解码(review)

```java
class Solution {

    public String decodeString(String s) {

        Deque<Integer> countStack = new ArrayDeque<>();

        Deque<StringBuilder> prefixStack = new ArrayDeque<>();

  

        StringBuilder current = new StringBuilder();

        int repeat = 0;

        for (char c : s.toCharArray()) {

            if (c >= '0' && c <= '9') {

                repeat = repeat * 10 + c - '0';

            } else if (c == '[') {

                countStack.push(repeat);

                prefixStack.push(current);

                repeat = 0;

                current = new StringBuilder();

            } else if (c == ']') {

                int count = countStack.pop();

                StringBuilder previous = prefixStack.pop();

                while(count-->0){

                    previous.append(current);

                }

                current = previous;

            } else {

                 current.append(c);

            }

        }

  

        return  current.toString();

    }

}
```


##### 每日温度

刚开始不会写,看了2min题解,直接搞定

```java
class Solution {

    public int[] dailyTemperatures(int[] temperatures) {

        // 这题使用单调栈

        int[] answer = new int[temperatures.length];

        Deque<Integer> stack = new ArrayDeque<>();

        for(int i = 0;i<temperatures.length;i++){

            int value = temperatures[i];

            while(!stack.isEmpty()){

                int point  = stack.peek();

                if(temperatures[point]<value){

                    stack.pop();

                    answer[point] = i - point;

                    continue;

                }

                break;

            }

            stack.push(i);

        }

  

        return answer;

    }

}
```



##### 柱状图最大的矩形

```java
class Solution {

    public int largestRectangleArea(int[] heights) {

        // 一开始真的一点思路都没有啊

        // 后面看了题解感觉还行,就是一个最小栈的题

        int n = heights.length;

        Deque<Integer> stack = new ArrayDeque<>();

        stack.push(-1);

        int ans = 0;

        for(int right = 0;right<=n;right++){

            int h  = right < n ? heights[right] : -1;

            while(stack.size() > 1 && heights[stack.peek()] >h){

                int left  = stack.pop();

                ans = Math.max(ans, heights[left] * (right - left));

            }

            // 这里有问题,如果是 2,1,2 这种情况

            // 我们单调栈只能记录 到最值是 1,2的而不是 2,1,2

            stack.push(right);

        }

           return ans;

    }

}
```


修改后

```java
class Solution {

    public int largestRectangleArea(int[] heights) {

        // 一开始真的一点思路都没有啊

        // 后面看了题解感觉还行,就是一个最小栈的题

        int n = heights.length;

        Deque<Integer> stack = new ArrayDeque<>();

        stack.push(-1);

        int ans = 0;

        for(int right = 0;right<=n;right++){

            int h  = right < n ? heights[right] : -1;

            while(stack.size() > 1 && heights[stack.peek()] >h){

                // 重点是修改这里的代码

                int index  = stack.pop();

                int left = stack.peek();

                ans = Math.max(ans, heights[index] * (right - left-1));

            }

            // 这里有问题,如果是 2,1,2 这种情况

            // 我们单调栈只能记录 到最值是 1,2的而不是 2,1,2

            stack.push(right);

        }

           return ans;

    }

}
```


#### 堆

##### 数组中第K个最大元素

```java
public class Solution {

    private int quickSelect(List<Integer> nums, int k) {

        // 随机选择基准数

        Random rand = new Random();

        int pivot = nums.get(rand.nextInt(nums.size()));

        // 将大于、小于、等于 pivot 的元素划分至 big, small, equal 中

        List<Integer> big = new ArrayList<>();

        List<Integer> equal = new ArrayList<>();

        List<Integer> small = new ArrayList<>();

        for (int num : nums) {

            if (num > pivot)

                big.add(num);

            else if (num < pivot)

                small.add(num);

            else

                equal.add(num);

        }

        // 第 k 大元素在 big 中，递归划分

        if (k <= big.size())

            return quickSelect(big, k);

        // 第 k 大元素在 small 中，递归划分

        if (nums.size() - small.size() < k)

            return quickSelect(small, k - nums.size() + small.size());

        // 第 k 大元素在 equal 中，直接返回 pivot

        return pivot;

    }

    public int findKthLargest(int[] nums, int k) {

        List<Integer> numList = new ArrayList<>();

        for (int num : nums) {

            numList.add(num);

        }

        return quickSelect(numList, k);

    }

}
```


##### 前K个高频元素(两个方法没手敲,看了看)

方法一: 维护一个最小堆,这里要想清楚为啥要最大元素,但是维护最小堆

```java
class Solution {
    public int[] topKFrequent(int[] nums, int k) {
        Map<Integer, Integer> frequency = new HashMap<>();

        for (int num : nums) {
            frequency.put(
                    num,
                    frequency.getOrDefault(num, 0) + 1
            );
        }

        // [数字, 出现次数]
        PriorityQueue<int[]> heap =
                new PriorityQueue<>((a, b) -> a[1] - b[1]);

        for (Map.Entry<Integer, Integer> entry
                : frequency.entrySet()) {

            heap.offer(new int[]{
                    entry.getKey(),
                    entry.getValue()
            });

            if (heap.size() > k) {
                heap.poll();
            }
        }

        int[] result = new int[k];

        for (int i = 0; i < k; i++) {
            result[i] = heap.poll()[0];
        }

        return result;
    }
}
```

```java
class Solution {

    public int[] topKFrequent(int[] nums, int k) {

        // 最简单就是做一个大顶堆

        // 然后不停地改节点

        // 这个算法是 n log n

        // 如果要 n 复杂度

        // 使用map记录次数,然后得排序,也是 n log n

        // n 复杂度不好做

        // 其实这道题可以用桶排序优化,就可以 n 复杂度了

  

        // 先放元素

        Map<Integer,Integer> map = new HashMap<>();

        for(int num: nums){

            // 这个方法有人会用merge,但是我不会

            map.put(num,map.getOrDefault(num,0)+1);

        }

  

        List<Integer>[] buckets = new List[nums.length + 1];

        for (Map.Entry<Integer, Integer> entry : map.entrySet()) {

            int number = entry.getKey();

            int frequency = entry.getValue();

  

            if (buckets[frequency] == null) {

                buckets[frequency] = new ArrayList<>();

            }

  

            buckets[frequency].add(number);

        }

  
  

        //  倒序遍历,拿k个

        int[] result = new int[k];

        int index = 0;

  

        for (int frequency = nums.length;

             frequency >= 1 && index < k;

             frequency--) {

  

            if (buckets[frequency] == null) {

                continue;

            }

  

            for (int number : buckets[frequency]) {

                result[index++] = number;

  

                if (index == k) {

                    break;

                }

            }

        }

        return result;

    }

}
```


##### 数据流中的中位数

```java
class MedianFinder {

  

    // 其实这道题,我更推荐使用双向链表

    // 为什么说呢,

    // 使用链表,addNum的复杂度可以提升到 n

    // 顺便把 中位数的 复杂度提升到 O(1)

    // 但是这个插入是我们不能忍受的

    // 所以使用堆,把插入改造为 log n

  

    // 保存较小的一半，是大顶堆

    private final PriorityQueue<Integer> left;

  

    // 保存较大的一半，是小顶堆

    private final PriorityQueue<Integer> right;

  

    public MedianFinder() {

        left = new PriorityQueue<>(Comparator.reverseOrder());

        right = new PriorityQueue<>();

    }

    public void addNum(int num) {

        left.offer(num);

        right.offer(left.poll());

        if (right.size() > left.size()) {

            left.offer(right.poll());

        }

    }

    public double findMedian() {

         if (left.size() > right.size()) {

            return left.peek();

        }

  

        return ((long) left.peek() + right.peek()) / 2.0;

    }

}

  

/**

 * Your MedianFinder object will be instantiated and called as such:

 * MedianFinder obj = new MedianFinder();

 * obj.addNum(num);

 * double param_2 = obj.findMedian();

 */
```




#### 贪心

##### 买卖股票的最佳时机

这道题太简单了,直接最简单方法做了,不过确实没有体会到这道题的贪心思想在哪里
```java
class Solution {

    public int maxProfit(int[] prices) {

        // 我觉得这道题就是,当前最小值与当前值的差

        // 贪心我倒是没看出来

        int earn = 0 ;

        int min = Integer.MAX_VALUE;

  

        for(int price: prices){

            min = Math.min(min,price);

            earn = Math.max(earn,price-min);

        }

        return earn;

    }

}
```



##### 跳跃游戏

思路解析
```java
class Solution {

    public boolean canJump(int[] nums) {

        // 这道题确实没有思路

        // 我其实想从后面开始往前贪心的

        // 不过在前面也可以

        // 每次选择自己可以去到的地方内最大的数

        // 我们就做bfs或者dfs

  

        int position = 0;

        int step = nums[0];

        PriorityQueue<Node> pq = new PriorityQueue<>();

        pq.offer(new Node(position,step));

        // 这种有点类似层序遍历了,类似bfs

        while(!pq.isEmpty()){

            pq.

        }

        // 也可以用dfs

        backtrace();

    }

  

    void backtrace(){}

  

    class Node{

        int index;

        int value;

    }

}
```


最简单贪心实现

```java
```
