---
title: "LeetCode - 00003"
date: 2019-08-14 12:03:49
tags:
  - LeetCode
#intro : 'Enter intro here to display on home page'
comments: false
---

利用动态规划来做，时间复杂度是O(n)，状态转移方程如下

```
# l - 末尾为当前位置的最大长度
# last - 某个字符上一次出现位置

l[n] = min(l[n-1] + 1, n - last[s[n]]);
```

所以其实代码还能再精简一点的，有时间再改吧，不需要那么多判断。

```cpp
class Solution
{
public:
	int lengthOfLongestSubstring(string s)
	{
		int last[255];
		memset(last, -1, sizeof(last));

		int longest[100000];

		byte prev = (byte)0;
		int i = 0;
		int max = 0;
		for (string::iterator it = s.begin(); it != s.end(); it++)
		{
			byte c = (byte)*it;

			if (i == 0)
			{
				longest[i] = 1;
			}
			else
			{
				if (prev == c)
				{
					longest[i] = 1;
				}
				else
				{
					longest[i] = longest[i - 1] + 1;
					if (last[(int)c] >= 0)
					{
						if (last[(int)c] >= i - longest[i] + 1)
						{
							longest[i] = i - last[(int)c];
						}
					}
				}
			}

			if (longest[i] > max)
			{
				max = longest[i];
			}
			last[(int)c] = i;
			i++;
		}

		return max;
	}
};
```
