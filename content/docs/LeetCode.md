## 1、两数之和

[两数之和](https://leetcode.cn/problems/two-sum/)

```markdown
* 给定一个整数数组 nums 和一个整数目标值 target，请你在该数组中找出 和为目标值 target  的那 两个 整数，并返回它们的数组下标。
* 你可以假设每种输入只会对应一个答案。但是，数组中同一个元素在答案里不能重复出现
* 你可以按任意顺序返回答案。
```

---

> unordered_map

```markdown
** 这题需要使用到unordered_map即哈希表，为了提前准备好已经配对的值，将target-nums[i]，i的值放入哈希表中，如果后面找到了以后就知道之前有过配对的值，就可以直接返回，这样做优化了时间
```

```c++
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        vector<int> res;
        unordered_map<int,int> mp;
        for(int i = 0 ; i < nums.size(); i++)
        {
            int other = target - nums[i];
            if(mp.count(other))
            {
                res = vector<int>({mp[other], i});
                break;
            }
            mp[nums[i]] = i;
        }
        return res;
    }
};
```



## 2、两数相加

[两数相加](https://leetcode.cn/problems/add-two-numbers/)

```markdown
* 给你两个 非空 的链表，表示两个非负的整数。它们每位数字都是按照**逆序**的方式存储的，并且每个节点只能存储**一位**数字。
* 请你将两个数相加，并以相同形式返回一个表示和的链表。
* 你可以假设除了数字 0 之外，这两个数都不会以 0 开头。
```

---

> 链表

```markdown
** 这题一开始做的时候想得不好，想要将真实的值先求出来相加，最后再放进去，这样以后导致两个问题。
* 1、复杂度提升了
* 2、long long 可能都不够表达原本数字的大小
所以，其实只需要按照加法器的逻辑进行实现就可以了，增加一个carry位（进位）再遍历两个链表，每位相加，进一等等。
```

```c++
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode* addTwoNumbers(ListNode* l1, ListNode* l2) {
        
        ListNode* head = new ListNode(-1);
        int carry = 0;
        ListNode* now = head;
        while(l1 || l2){
            int v1 = l1 ? l1 -> val : 0;
            int v2 = l2 ? l2 -> val : 0;
            int sum = v1 + v2 + carry;

            carry = sum / 10;
            now->next = new ListNode(sum % 10);
            now = now -> next;
            
            if(l1){ 
                l1 = l1 -> next;
            }
            if(l2){
                l2 = l2 -> next;
            } 
        }

        if(carry){
            now->next = new ListNode(carry);
        }

        return head->next;
    }
};
```



## 3、无重复字符的最长子串

[无重复字符的最长子串](https://leetcode.cn/problems/longest-substring-without-repeating-characters/)

```markdown
* 给定一个字符串 s ，请你找出其中不含有重复字符的 最长子串 的长度。
```

---

> unordered_map，子串的思想：i固定，j动，因为j不满足条件再移动i

```markdown
** 这题比较难想。
逻辑如下，有i，j两个指针和一个哈希表
1、每次将s[j]放入哈希表
2、如果hash[s[j]]，即s[j]出现次数大于1了，将i往后移动，一直到s[j]出现次数等于1
3、每次得到的i，j来求一个长度，更新最大值
```

```c++
class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        unordered_map<char,int> hash;
        int res = 0;
        for(int i = 0 , j = 0 ; j < s.size() ; j++)
        {
            hash[s[j]]++;
            while(hash[s[j]] > 1) hash[s[i++]]--;
            res = max(res,j - i + 1);
        }
        return res;
    }
};
```



## 4、寻找两个正序数组的中位数

[寻找两个正序数组的中位数](https://leetcode.cn/problems/median-of-two-sorted-arrays)

```markdown
* 给定两个大小分别为 m 和 n 的正序（从小到大）数组 nums1 和 nums2。请你找出并返回这两个正序数组的 中位数 。
* 算法的时间复杂度应该为 O(log (m+n)) 。
```

---

> 第k大数，递归方法实现

```markdown
** 思路：首先是用递归写出找到第k大的数，这样以后只要讨论nums1和nums2的个数总和的奇偶性问题了。
找出第k大的数步骤如下：(i,j是递归的时候的左端)
1、 先考虑m，n > k的情况：
	首先在num1和num2中各自取出k/2个数字，然后比较num1[k/2 - 1]与num2[k/2 - 1]谁大。
	如果num1[k / 2]更大，那么说明num2的前k / 2个数字一定在前k个数之内，直接放进去，缩小了一半的大小，即只需要num1在[i,num1.size()]和num2在[j + k/2,num2.size()]里找到第k/2个数。将范围直接减少了一半
	同理，如果num1对应的更大的话就在num1在[i + k/2,num1.size()]和num2在[j,num2.size()]里找到第k/2个数
2、 再考虑m < k的情况，因为m + n > k , 所以m < k的时候n必然>k
	这里我们让num1永远是小的那个。
	也就是说我们考虑num1[m - 1]和num2[k / 2 - 1]哪个更大。如果num1[m - 1]大，num1在[i,num1.size()]和num2在[j + k / 2,num2.size()]找第k / 2个数
	如果num2[k / 2 - 1]更大，我们就拿到结果num2[j + k - m - 1]
3、 考虑出口：
	1、将**如果num2[k / 2 - 1]更大，我们就拿到结果num2[j + k - m - 1]**这一块包装成
	   if(i == num1.size()) return num2[j + k - 1];
	2、考虑k = 1的时候，因为k = 1的时候 k / 2 为 0
```

```c++
class Solution {
public:
    double findMedianSortedArrays(vector<int>& nums1, vector<int>& nums2) {
       int total = nums1.size() + nums2.size();
       if(total % 2){
           return findKthNumber(nums1,0,nums2,0,total / 2 + 1);
       }else{
           int left = findKthNumber(nums1,0,nums2,0,total / 2);
           int right = findKthNumber(nums1,0,nums2,0,total / 2 + 1);
           return (left + right) / 2.0;
       }

    }

    double findKthNumber(vector<int>& num1, int i, vector<int>& num2, int j, int k){
       if(num1.size() - i > num2.size() - j) return findKthNumber(num2,j,num1,i,k);
       if(i == num1.size()) return num2[j + k - 1];
       if(k == 1) return min(num1[i],num2[j]);

        int si = min(i + k / 2 , int(num1.size()));
        int sj = j + k / 2;

        if(num1[si - 1] > num2[sj - 1]){
            return findKthNumber(num1,i,num2,j + k / 2 , k - k / 2);
        }else{
            return findKthNumber(num1,si,num2,j,k - (si - i));
        }

    }
};
```



## 5、 最长回文子串

[最长回文子串](https://leetcode.cn/problems/longest-palindromic-substring/)

```markdown
* 给你一个字符串 s，找到 s 中最长的回文子串。
* 如果字符串的反序与原始字符串相同，则该字符串称为回文字符串。
```

---

> 遍历中间值，奇偶的情况

```markdown
** 首先遍历奇数情况 如：aba这种情况，i是遍历中间的位置，然后j往两端移动，满足条件就更新大小。
接着遍历偶数情况，如：abba这种情况，也是遍历中间位置，j从i开始，k从i+1开始，然后j--，k++，满足条件就更新大小。
```

```c++
class Solution {
public:
    string longestPalindrome(string s) {
        int res = 0;
        string str;
        for(int i = 0 ; i < s.size() ; i++)
        {
            // 奇数情况
            for(int j = 0 ; j <= i && i + j < s.size() ; j++){

                if(s[i - j] == s[i + j]){
                    if(j * 2 + 1 > res){
                        res = j * 2 + 1;
                        str = s.substr(i - j,j * 2 + 1);
                    }   
                }else{
                    break;
                }
            }

            // 偶数情况
            for(int j = i , k = i + 1; j >= 0 && k < s.size() ; j-- , k++){
                if(s[j] == s[k]){
                    if(k - j + 1 > res){
                        res = k - j + 1;
                        str = s.substr(j,k - j + 1);    
                    }
                }else{
                    break;
                }
            }
        }
        return str;
    }
};
```



## 6、 Z字形变换

[Z字形变换](https://leetcode.cn/problems/zigzag-conversion/)

```markdown
* 将一个给定字符串 s 根据给定的行数 numRows ，以从上往下、从左到右进行 Z 字形排列。
* 比如输入字符串为 "PAYPALISHIRING" 行数为 3 时，排列如下：
```

```markdown
P   A   H   N
A P L S I I G
Y   I   R
```

```markdown
* 之后，你的输出需要从左往右逐行读取，产生出一个新的字符串，比如："PAHNAPLSIIGYIR"。
* 请你实现这个将字符串进行指定行数变换的函数：
```

---

> 需要先画出n为4的情形，找规律

```markdown
** 这种按某种形状打印字符的题目，一般通过手画小图找规律来做。
我们先画行数是4的情况：
0     6       12
1   5 7    11 ..
2 4   8 10
3     9
对于第一行和最后一行，是公差为2×（n-1）等差数列
对于第i行，是两个公差为2×（n-1）的等差数列，首项分别是i和2n-i-2
```

```c++
class Solution {
public:
    string convert(string s, int numRows) {
        string res;
        int n = numRows;
        if(n == 1) return s;
        for(int i = 0 ; i < n ; i++ ){

            int diff = 2 * (n - 1);
            if(i == 0 || i == n - 1){
                for(int j = i ; j < s.size() ; j += diff){
                    res += s[j];
                }
            }else{
                for(int j = i , k = diff - i ; j < s.size() || k < s.size() ;j+=diff,k+=diff){
                    if(j < s.size()) res += s[j];
                    if(k < s.size()) res += s[k];
                }
            }
        }
        return res;
    }
};
```



## 7、 整数反转

[整数反转](https://leetcode.cn/problems/reverse-integer/submissions/)

```markdown
* 给你一个 32 位的有符号整数 x ，返回将 x 中的数字部分反转后的结果。
* 如果反转后整数超过 32 位的有符号整数的范围 [−2^31,  2^31 − 1] ，就返回 0。
* 假设环境不允许存储 64 位整数（有符号或无符号）。
```

---

> INT_MAX和INT_MIN的边界

```markdown
** 要考虑INT_MAX和INT_MIN的边界情况
```

```c++
class Solution {
public:
    int reverse(int x) {
        int res = 0;
        while (x) {
            if (x > 0 && res > (INT_MAX - x % 10) / 10) return 0;
            if (x < 0 && res < (INT_MIN - x % 10) / 10) return 0;
            res = res * 10 + x % 10;
            x /= 10;
        }
        return res;
    }
};
```



## 8、 字符串转换整数

[字符串转换整数](https://leetcode.cn/problems/string-to-integer-atoi/)

```markdown
* 请你来实现一个 myAtoi(string s) 函数，使其能将字符串转换成一个 32 位有符号整数（类似 C/C++ 中的 atoi 函数）。
* 函数 myAtoi(string s) 的算法如下：
* 读入字符串并丢弃无用的前导空格
* 检查下一个字符（假设还未到字符末尾）为正还是负号，读取该字符（如果有）。 确定最终结果是负数还是正数。 如果两者都不存在，则假定结果为正。
* 读入下一个字符，直到到达下一个非数字字符或到达输入的结尾。字符串的其余部分将被忽略。
* 将前面步骤读入的这些数字转换为整数（即，"123" -> 123， "0032" -> 32）。如果没有读入数字，则整数为 0 。必要时更改符号（从步骤 2 开始）。
* 如果整数数超过 32 位有符号整数范围 [−2^31,  2^31 − 1] ，需要截断这个整数，使其保持在这个范围内。具体来说，小于 −2^31 的整数应该被固定为 −2^31 ，大于 2^31 − 1 的整数应该被固定为 2^31 − 1 。
* 返回整数作为最终结果。
* 注意：
* 本题中的空白字符只包括空格字符 ' ' 。
* 除前导空格或数字后的其余字符串外，请勿忽略 任何其他字符。
```

---

> 模拟

```markdown
** 这题是模拟，按着模拟的顺序来写就好了。
```

```c++
class Solution {
public:
    int myAtoi(string str) {
        int res = 0;
        int k = 0;
        while (k < str.size() && str[k] == ' ') k ++ ;
        if (k == str.size()) return 0;

        int minus = 1;
        if (str[k] == '-') minus = -1, k ++ ;
        if (str[k] == '+')
            if (minus == -1) return 0;
            else k ++ ;

        while (k < str.size() && str[k] >= '0' && str[k] <= '9') {
            int x = str[k] - '0';
            if (minus > 0 && res > (INT_MAX - x) / 10) return INT_MAX;
            if (minus < 0 && -res < (INT_MIN + x) / 10) return INT_MIN;
            if (-res * 10 - x == INT_MIN) return INT_MIN;
            res = res * 10 + x;
            k ++ ;
        }

        res *= minus;
        return res;
    }
};
```



## 9、 回文数

[回文数](https://leetcode.cn/problems/palindrome-number/)

```markdown
* 给你一个整数 x ，如果 x 是一个回文整数，返回 true ；否则，返回 false 。
* 回文数是指正序（从左向右）和倒序（从右向左）读都是一样的整数。
* 例如，121 是回文，而 123 不是。
```

---

> to_string()，回文数

```markdown
** 先将int类型的转换为string类型，然后直接考虑该string是不是回文串
```

```c++
class Solution {
public:
    bool isPalindrome(int x) {
        string s = to_string(x);
        int i = 0 , j = s.size() - 1;
        while(i < j){
            if(s[i] == s[j]){
                i ++ , j -- ;
            }
            else{
                return false;
            }
        }
        return true;
    }
};
```



## 10、 正则表达式匹配

[正则表达式匹配](https://leetcode.cn/problems/regular-expression-matching/)

```markdown
* 给你一个字符串 s 和一个字符规律 p，请你来实现一个支持 '.' 和 '\*' 的正则表达式匹配。
* '.' 匹配任意单个字符
* '\*' 匹配零个或多个前面的那一个元素
* 所谓匹配，是要涵盖 整个 字符串 s的，而不是部分字符串。
```

---

> DP

```markdown
** 动态规划
1、 状态表示
	1、集合
		所有s[1~i]和p[1~j]的方案
	2、属性
		bool，是否存在匹配的合法方案
2、 状态计算
	1、 p[j] != '*'
		f[i][j] = ( s[i] == p[j] || p[j] == '.' ) && f[i - 1][j - 1]
	2、 p[j] == '*'
		f[i][j] = f[i][j-2] (*代表空的时候)|f[i-1][j-2]&s[i]与p[j-1]匹配（*代表一个的时候）|f[i-2][j-2]&s[i]与p[j-2]匹配&s[i-1]与p[j-2]匹配（*代表两个字符的时候）
		f[i-1][j] = f[i-1][j-2]|f[i-2][j-2]&s[i-1]与p[j-1]匹配|f[i-2][j-2]&s[i-1]与p[j-2]匹配&s[i-2]与p[j-2]匹配
		如此以后，我们发现f[i][j]的后半段就是f[i-1][j] & s[i]与p[j-1]匹配
		f[i][j] = f[i][j-2]|f[i-1][j]&s[i]与p[j-1]匹配
```

```c++
class Solution {
public:
    bool isMatch(string s, string p) {
        int n = s.size();
        int m = p.size();
        s = ' ' + s;
        p = ' ' + p;
        vector<vector<bool>> f(n + 1,vector<bool>(m + 1));
        f[0][0] = true;

        for(int i = 0 ; i <= n ; i++)
            for(int j = 1 ; j <= m ;j++){
                if (j + 1 <= m && p[j + 1] == '*') continue;
                if(i && p[j] != '*'){
                    f[i][j] = ( s[i] == p[j] || p[j] == '.' ) && f[i - 1][j - 1];
                }else if(p[j] == '*'){
                    f[i][j] = f[i][j - 2] || i && f[i - 1][j] && ( s[i] == p[j - 1] || p[j - 1] == '.');
                }
            }
        return f[n][m];
    }
};
```



## 11、 盛水最多的容器

[盛水最多的容器](https://leetcode.cn/problems/container-with-most-water/)

```markdown
* 给定一个长度为 n 的整数数组 height 。有 n 条垂线，第 i 条线的两个端点是 (i, 0) 和 (i, height[i]) 。
* 找出其中的两条线，使得它们与 x 轴共同构成的容器可以容纳最多的水。
* 返回容器可以储存的最大水量。
* 说明：你不能倾斜容器。
```

---

> 指针扫描，难在思路

```markdown
** 这题思路比较难
```

![image-20230129184911080](../assets/image-20230129184911080.png)

```c++
class Solution {
public:
    int maxArea(vector<int>& height) {
        int res = 0;
        for(int i = 0 , j = height.size() - 1 ; i < j; ){
            res = max(res,min(height[i],height[j]) * (j - i));
            if(height[j] > height[i]) i++;
            else j--;
        }
        return res;
    }
};
```



## 12、 整数转罗马数字

[整数转罗马数字](https://leetcode.cn/problems/integer-to-roman/)

```markdown
* 罗马数字包含以下七种字符： I， V， X， L，C，D 和 M。
	字符          数值
	I             1
	V             5
	X             10
	L             50
	C             100
	D             500
	M             1000
* 例如， 罗马数字 2 写做 II ，即为两个并列的 1。12 写做 XII ，即为 X + II 。 27 写做  XXVII, 即为 XX + V + II 。
* 通常情况下，罗马数字中小的数字在大的数字的右边。但也存在特例，例如 4 不写做 IIII，而是 IV。数字 1 在数字 5 的左边，所表示的数等于大数 5 减小数 1 得到的数值 4 。同样地，数字 9 表示为 IX。这个特殊的规则只适用于以下六种情况：
* I 可以放在 V (5) 和 X (10) 的左边，来表示 4 和 9。
* X 可以放在 L (50) 和 C (100) 的左边，来表示 40 和 90。 
* C 可以放在 D (500) 和 M (1000) 的左边，来表示 400 和 900。
* 给你一个整数，将其转为罗马数字。
```

---

> 模拟

```markdown
**
```

![image-20230129190943143](../assets/image-20230129190943143.png)

```c++
class Solution {
public:
    string intToRoman(int num) {
        int values[] = {
            1000,
            900, 500, 400, 100,
            90, 50, 40, 10,
            9, 5, 4, 1
        };
        string reps[] = {
            "M",
            "CM", "D", "CD", "C",
            "XC", "L", "XL", "X",
            "IX", "V", "IV", "I",
        };

        string res;
        for (int i = 0; i < 13; i ++ ) {
            while (num >= values[i]) {
                num -= values[i];
                res += reps[i];
            }
        }

        return res;
    }
};
```



## 13、罗马数字转数字

```markdown
* 罗马数字包含以下七种字符: I， V， X， L，C，D 和 M。
	字符          数值
	I             1
	V             5
	X             10
	L             50
	C             100
	D             500
	M             1000
* 例如， 罗马数字 2 写做 II ，即为两个并列的 1 。12 写做 XII ，即为 X + II 。 27 写做  XXVII, 即为 XX + V + II 。
* 通常情况下，罗马数字中小的数字在大的数字的右边。但也存在特例，例如 4 不写做 IIII，而是 IV。数字 1 在数字 5 的左边，所表示的数等于大数 5 减小数 1 得到的数值 4 。同样地，数字 9 表示为 IX。这个特殊的规则只适用于以下六种情况：
* I 可以放在 V (5) 和 X (10) 的左边，来表示 4 和 9。
* X 可以放在 L (50) 和 C (100) 的左边，来表示 40 和 90。 
* C 可以放在 D (500) 和 M (1000) 的左边，来表示 400 和 900。
* 给定一个罗马数字，将其转换成整数。
```

---

> 哈希表 + 模拟

```markdown
** 如果左边的对应的数字比右边小，就减去，如果左边对应的数字比右边大或相等，就加上
```

```c++
class Solution {
public:
    int romanToInt(string s) {
        unordered_map<char, int> hash;
        hash['I'] = 1, hash['V'] = 5;
        hash['X'] = 10, hash['L'] = 50;
        hash['C'] = 100, hash['D'] = 500;
        hash['M'] = 1000;

        int res = 0;
        for (int i = 0; i < s.size(); i ++ ) {
            if (i + 1 < s.size() && hash[s[i]] < hash[s[i + 1]])
                res -= hash[s[i]];
            else
                res += hash[s[i]];
        }

        return res;
    }
};
```



## 14、 最长公共前缀

[最长公共前缀](https://leetcode.cn/problems/longest-common-prefix/)

```markdown
* 编写一个函数来查找字符串数组中的最长公共前缀。
* 如果不存在公共前缀，返回空字符串 ""。
```

---

> 模拟

```markdown
** 简单题，模拟一下就可以了
```

```c++
class Solution {
public:
    string longestCommonPrefix(vector<string>& strs) {
        string res = "";
        if(strs.empty()) return"";

        for(int i = 0 ;;i++){
            if(i == strs[0].size()) return res;
            char c = strs[0][i];
            for(auto s : strs){
                if(s[i] != c)
                    return res;
            }
            res += c;
        }
        return res;
    }
};
```



## 15、 三数之和

[三数之和](https://leetcode.cn/problems/3sum/)

```markdown
* 给你一个整数数组 nums ，判断是否存在三元组 [nums[i], nums[j], nums[k]] 满足 i != j、i != k 且 j != k ，同时还满足 nums[i] + nums[j] + nums[k] == 0 。请
* 你返回所有和为 0 且不重复的三元组。
* 注意：答案中不可以包含重复的三元组。
```

---

> 排序 + 双指针做法

```markdown
** 首先排序保证顺序，保证i < j < k
接着j和k跟着动，当j往右边移动，则k要往左边移动，双指针做法。
注意 k - 1 和 k 的区别
```

```c++
class Solution {
public:
    vector<vector<int>> threeSum(vector<int>& nums) {
        vector<vector<int>> res;
        sort(nums.begin(), nums.end());
        for (int i = 0; i < nums.size(); i ++ ) {
            if (i && nums[i] == nums[i - 1]) continue;
            for (int j = i + 1, k = nums.size() - 1; j < k; j ++ ) {
                if (j > i + 1 && nums[j] == nums[j - 1]) continue;
                while (j < k - 1 && nums[i] + nums[j] + nums[k - 1] >= 0) k -- ;
                if (nums[i] + nums[j] + nums[k] == 0) {
                    res.push_back({nums[i], nums[j], nums[k]});
                }
            }
        }

        return res;

    }
};
```



## 16、 最接近的三数之和

[最接近的三数之和](https://leetcode.cn/problems/3sum-closest/)

```markdown
* 给你一个长度为 n 的整数数组 nums 和 一个目标值 target。请你从 nums 中选出三个整数，使它们的和与 target 最接近。
* 返回这三个数的和。
* 假定每组输入只存在恰好一个解。
```

---

> 排序 + 双指针算法

```markdown
** 这题思路与上面相同，只需要将0改成target，使用pair来存储即可
```

```c++
class Solution {
public:
    int threeSumClosest(vector<int>& nums, int target) {
        sort(nums.begin(),nums.end());
        pair<int,int> res(INT_MAX,INT_MAX);

        for(int i = 0 ; i < nums.size() ; i++ ){
            for(int j = i + 1 , k = nums.size() - 1 ; j < k ; j++){
                while(j < k - 1 && nums[i] + nums[j] + nums[k - 1] >= target) k--;
                int s = nums[i] + nums[j] + nums[k];
                res = min(res, make_pair(abs(s - target), s));
                if (k - 1 > j) {
                    s = nums[i] + nums[j] + nums[k - 1];
                    res = min(res, make_pair(target - s, s));
                }
            }
        }

        return res.second;

    }
};
```



## 17、 电话号码的字母组合

[电话号码的字母组合](https://leetcode.cn/problems/letter-combinations-of-a-phone-number/)

```markdown
* 给定一个仅包含数字 2-9 的字符串，返回所有它能表示的字母组合。答案可以按 任意顺序 返回。
* 给出数字到字母的映射如下（与电话按键相同）。注意 1 不对应任何字母。
```

---

> 递归

```markdown
** 递归的方法实现。
dfs中有三个变量，digits（原来的串），u（遍历到第几个），path（当前组合的情况）
首先判断出口，即u遍历到digits.size()，遍历完整个串，那么直接将path加入并返回
获取当前这个串对应的数字，将这个数字加入str中，然后传递到下一层
```

```c++
class Solution {
public:
    vector<string> ans;
    string strs[10] = {
        "", "", "abc", "def",
        "ghi", "jkl", "mno",
        "pqrs", "tuv", "wxyz",
    };
    vector<string> letterCombinations(string digits) {
        if(digits.empty()) return ans;
        dfs(digits,0,"");
        return ans;
    }
    void dfs(string& digits, int u, string path) {
        
        if(u == digits.size()) {
            ans.push_back(path);
            return;
        }
        int idx = digits[u] - '0';
        for(int i = 0 ; i < strs[idx].size() ; i++){
            string str = path;
            str = str + strs[idx][i];
            dfs(digits,u + 1,str);
        }
    }
};
```



## 18、 四数之和

[四数之和](https://leetcode.cn/problems/4sum/)

```markdown
* 给你一个由 n 个整数组成的数组 nums ，和一个目标值 target 。请你找出并返回满足下述全部条件且不重复的四元组 [nums[a], nums[b], nums[c], nums[d]] （若两个四元组元素一一对应，则认为两个四元组重复）：
* 0 <= a, b, c, d < n
* a、b、c 和 d 互不相同
* nums[a] + nums[b] + nums[c] + nums[d] == target
* 你可以按 任意顺序 返回答案 。
```

---

> 排序 + 双指针做法

```markdown
** 与三数之和相同，只是多加了一重循环
```

```c++
class Solution {
public:
    vector<vector<int>> fourSum(vector<int>& nums, int target) {
        sort(nums.begin(), nums.end());
        vector<vector<int>> res;
        for (int i = 0; i < nums.size(); i ++ ) {
            if (i && nums[i] == nums[i - 1]) continue;
            for (int j = i + 1; j < nums.size(); j ++ ) {
                if (j > i + 1 && nums[j] == nums[j - 1]) continue;
                for (int k = j + 1, u = nums.size() - 1; k < u; k ++ ) {
                    if (k > j + 1 && nums[k] == nums[k - 1]) continue;
                    while (u - 1 > k && (long)nums[i] + nums[j] + nums[k] + nums[u - 1] >= target) u -- ;
                    if ((long)nums[i] + nums[j] + nums[k] + nums[u] == target) {
                        res.push_back({nums[i], nums[j], nums[k], nums[u]});
                    }
                }
            }
        }

        return res;
    }
};
```



