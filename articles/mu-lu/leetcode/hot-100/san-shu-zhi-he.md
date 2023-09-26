# 三数之和

```markdown
给你一个整数数组 nums ，判断是否存在三元组 [nums[i], nums[jmark], nums[k]]
满足 i != j、i != k 且 j != k ，同时还满足 nums[i] + nums[j] + nums[k] == 0 。
请你返回所有和为 0 且不重复的三元组。

注意：答案中不可以包含重复的三元组。
```

```
先对数组从小到大进行排序，一层for循环，用左右双指针，左指针往右，右指针往左
所以就形成了三个数nums[i], nums[l], nums[j]，l初始值为i+1，r初始值为legth-1，以下几种情况
1. nums[i] > 0 ，表示后续所有元素都大于0，不存在和为0的情况，查找结束
2. nums[i] + nums[l] + nums[j] > 0，表示右指针元素过大，r--
3. nums[i] + nums[l] + nums[j] < 0，表示左指针元素过小，l++
4. nums[i] + nums[l] + nums[j] == 0，是符合要求的一组元素，但是为了保持三元组不重复
   需要做去重操作，如果nums[l+1] == nums[l] l++，nums[r-1] == nums[r] r--
```



```java
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {
        Arrays.sort(nums);
        //从小到大排序，用左右双指针，如果元素重复直接跳过
        List<List<Integer>> res = new ArrayList<>();
        for (int i=0;i<nums.length;i++) {
            int left = i+1;
            int right = nums.length-1;
            if (nums[i] > 0) {
                return res;
            }
            if (i>0&&nums[i]==nums[i-1]){
                continue;
            }
            while (left < right) {
                if (nums[i] + nums[left] + nums[right] == 0) {
                    while (left < right && nums[left] == nums[left + 1]) {
                        left ++;
                    }
                    while (left < right && nums[right] == nums[right -1]) {
                        right--;
                    }
                    res.add(Arrays.asList(nums[i],nums[left],nums[right]));
                    left ++;
                    right --;
                } else if (nums[i] + nums[left] + nums[right] > 0) {
                    right--;
                } else {
                    left ++;
                }
            }
        }
        return res;
    }
}
```
