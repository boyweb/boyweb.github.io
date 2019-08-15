---
title: "LeetCode - 00001"
date: 2019-08-14 11:36:56
tags:
  - LeetCode
#intro : 'Enter intro here to display on home page'
comments: false
---

> 之前都是自己写写就完了，自己从来没有记录过，算是从头开始吧。

最开始的想法是将结果做了个排序，大于的 target 的值排除，然后头尾指针移动判断。这个时间复杂度依赖于排序的时间复杂度，因为指针移动判断是 O(n)，所以就算用上了快排也是 O(nlogn)。

后来发现没有重复的值，所以能够直接用 map 来做，用空间换时间，时间复杂度和空间复杂度都是 O(n)。

```cpp
class Solution
{
public:
	vector<int> twoSum(vector<int> &nums, int target)
	{
		map<int, int> num_map;
		vector<int> ret;
		int i = 0;
		for (auto num : nums)
		{
			map<int, int>::iterator iter;
			if (iter = num_map.find(target - num), iter != num_map.end())
			{
				ret.push_back(iter->second);
				ret.push_back(i);
				break;
			}

			num_map[num] = i;
			i++;
		}

		return ret;
	}
};
```
