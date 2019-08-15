---
title: "LeetCode - 00002"
date: 2019-08-14 12:01:00
tags:
  - LeetCode
#intro : 'Enter intro here to display on home page'
comments: false
---

纯模拟题，没仔细想，写的比较草率，有时间再来优化。

```cpp
class Solution
{
public:
	ListNode *addTwoNumbers(ListNode *l1, ListNode *l2)
	{
		int next = 0;
		ListNode *p, *q;
		ListNode *r = NULL, *n = NULL;
		p = l1;
		q = l2;
		while (true)
		{
			int val = 0;
			if (p != NULL || q != NULL)
			{
				if (p != NULL)
				{
					val += p->val;
					p = p->next;
				}
				if (q != NULL)
				{
					val += q->val;
					q = q->next;
				}
				int r_val = (val + next) % 10;
				next = (val + next) / 10;

				if (n == NULL)
				{
					n = new ListNode(r_val);
					r = n;
				}
				else
				{
					n->next = new ListNode(r_val);
					n = n->next;
				}
			}
			else
			{
				break;
			}
		}

		if (next > 0)
		{
			if (n == NULL)
			{
				n = new ListNode(next);
				r = n;
			}
			else
			{
				n->next = new ListNode(next);
				n = n->next;
			}
		}

		return r;
	}
};
```
