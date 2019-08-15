---
title: "LeetCode - 00004"
date: 2019-08-15 12:50:49
tags:
  - LeetCode
#intro : 'Enter intro here to display on home page'
comments: false
---

最开始只想到了 O(m+n)的算法，没想到二分该怎么分，看了一下题解，学习了。

要利用中位数的概念来做。

```cpp
// 对于数组a，m是中位数
if (a.size() % 2 == 0)
{
    m = (a[a.size() / 2] + a[a.size() / 2 + 1]) / 2;
}
else
{
    m = a[a.size() / 2];
}
```

假定有两个有序数组，a 和 b，将他们各分成两份，叫 `al/ar`，`bl/br`，使得`len(al + bl) = len(ar + br) || len(al + bl) = len(ar + br) + 1`，那么如果 `max(al) < min(br) && max(bl) < min(ar)`，则可以确定，已经分成了大小两份，然后我们根据情况取`al`和`bl`的最大值来确定中位数。

怎么去确认这个分割点，就是利用二分法去做的，因此整个算法的时间复杂度是 O(log(n + m))。

```cpp
class Solution
{
public:
	double findMedianSortedArrays(vector<int> &nums1, vector<int> &nums2)
	{
		int size1 = nums1.size(), size2 = nums2.size();
		if (size1 > size2)
		{
			return findMedianSortedArrays(nums2, nums1);
		}

		int pos = 0;
		int mid_pos = (size1 + size2 + 1) / 2;
		int l = 0, r = size1;
		while (l < r)
		{
			int c1 = l + (r - l) / 2;
			int c2 = mid_pos - c1;

			if (nums1[c1] < nums2[c2 - 1])
			{
				l = c1 + 1;
			}
			else
			{
				r = c1;
			}
		}

		int m1 = l;
		int m2 = mid_pos - l;
		int c1 = max(m1 <= 0 ? INT_MIN : nums1[m1 - 1], m2 <= 0 ? INT_MIN : nums2[m2 - 1]);
		if ((size1 + size2) % 2 == 1)
		{
			return c1;
		}
		int c2 = min(m1 >= size1 ? INT_MAX : nums1[m1], m2 >= size2 ? INT_MAX : nums2[m2]);
		return (c1 + c2) * 0.5;
	}
};
```
