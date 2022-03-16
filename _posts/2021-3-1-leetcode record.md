# leetcode刷题记录
## 第一题
### 题目
1. 两数之和
难度
简单

给定一个整数数组 nums 和一个目标值 target，请你在该数组中找出和为目标值的那 两个 整数，并返回他们的数组下标。

你可以假设每种输入只会对应一个答案。但是，数组中同一个元素不能使用两遍。

 
示例:

给定 nums = [2, 7, 11, 15], target = 9

因为 nums[0] + nums[1] = 2 + 7 = 9
所以返回 [0, 1]
### 解答
```
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        vector<int> result;
        int a = 0, b = 0;
        for(int i = 0; i < nums.size(); i++)
        {
            a = nums[i];
            b = target - a;
            for(int j = i+1 ; j < nums.size() ; j++ )
            {
                if(nums[j] == b)
                {
                    result.push_back(i);
                    result.push_back(j);
                    return result;
                }
            }
            
        }
        return result;
    }
};
```
## 第二题
### 题目
709. 转换成小写字母
难度
简单

实现函数 ToLowerCase()，该函数接收一个字符串参数 str，并将该字符串中的大写字母转换成小写字母，之后返回新的字符串。

 

示例 1：

输入: "Hello"
输出: "hello"
示例 2：

输入: "here"
输出: "here"
示例 3：

输入: "LOVELY"
输出: "lovely"
### 解答 
```
class Solution {
public:
    string toLowerCase(string str) {
        for(int i = 0; i < str.size(); i++)
        {
            if(str[i]<=90 && str[i]>=65)
            {
                str[i] += 32;
            }
        }
        return str;
    }
};
```
## 第三题
### 题目
1631. 最小体力消耗路径
难度
中等

你准备参加一场远足活动。给你一个二维 rows x columns 的地图 heights ，其中 heights[row][col] 表示格子 (row, col) 的高度。一开始你在最左上角的格子 (0, 0) ，且你希望去最右下角的格子 (rows-1, columns-1) （注意下标从 0 开始编号）。你每次可以往 上，下，左，右 四个方向之一移动，你想要找到耗费 体力 最小的一条路径。

一条路径耗费的 体力值 是路径上相邻格子之间 高度差绝对值 的 最大值 决定的。

请你返回从左上角走到右下角的最小 体力消耗值 。

 

示例 1：



输入：heights = [[1,2,2],[3,8,2],[5,3,5]]
输出：2
解释：路径 [1,3,5,3,5] 连续格子的差值绝对值最大为 2 。
这条路径比路径 [1,2,2,2,5] 更优，因为另一条路径差值最大值为 3 。
示例 2：



输入：heights = [[1,2,3],[3,8,4],[5,3,5]]
输出：1
解释：路径 [1,2,3,4,5] 的相邻格子差值绝对值最大为 1 ，比路径 [1,3,5,3,5] 更优。
示例 3：


输入：heights = [[1,2,1,1,1],[1,2,1,2,1],[1,2,1,2,1],[1,2,1,2,1],[1,1,1,2,1]]
输出：0
解释：上图所示路径不需要消耗任何体力。
 

提示：

rows == heights.length
columns == heights[i].length
1 <= rows, columns <= 100
1 <= heights[i][j] <= 106
### 思路
广度优先搜索遍历，对于每一个节点，都记录一个到他的路线的绝对值的最小值

每次探索到节点 首先判断当前节点与上一节点的高度之差，与之到它前一个节点的绝对值比较，取最大者，此时就是当前路径到此节点的绝对值之差的信息

用这一信息与当前节点事先在矩阵中存储的绝对值比较，因为一个节点有不同路径，当前路径得出的权值与历史版本最优值相比较

若优于历史版本，则将当前得出的权值记录，若不优于，则不存储，不进行下一步探索

返回结果即为最后一个节点的权值

### 解答

```
class Solution {
public:
    int minimumEffortPath(vector<vector<int>>& heights) {
        int row_size = heights.size(), col_size = heights[0].size();
        
        //bitset<(row_size-1)*(col_size-1)> map;
        vector<vector<int>> count(row_size,vector<int>(col_size,INT_MAX-1));
        count[0][0] = 0;
        queue<vector<int>> q;
        vector<int> cur     = {0,0};
        vector<int> temp    = {0,0};
        q.push(cur);
        int dx[4] = {0, 1, 0, -1}, dy[4] = {1, 0, -1, 0};
        int step_gap = 0;
        while(!q.empty())
        {
            cur = q.front();
            q.pop();
            for(int i = 0; i<4; i++)
            {
                int x = cur[0]+dx[i],y = cur[1]+dy[i];
                if(x>=0 && x<row_size&&y>=0&&y<col_size)
                {
                    step_gap = abs(heights[cur[0]][cur[1]]-heights[x][y]);
                    step_gap = (step_gap>count[cur[0]][cur[1]])?step_gap:count[cur[0]][cur[1]];
                
                
                if(step_gap >= count[x][y]) continue;
                
                count[x][y] = step_gap;
                temp[0] =x;
                temp[1] = y;
                q.push(temp);
                }
            }
           
        }
        return count[row_size-1][col_size-1];
    }
};
```
## 第四题
### 题目
401. 二进制手表

难度
简单

二进制手表顶部有 4 个 LED 代表 小时（0-11），底部的 6 个 LED 代表 分钟（0-59）。

每个 LED 代表一个 0 或 1，最低位在右侧。



例如，上面的二进制手表读取 “3:25”。

给定一个非负整数 n 代表当前 LED 亮着的数量，返回所有可能的时间。

 

示例：

输入: n = 1
返回: ["1:00", "2:00", "4:00", "8:00", "0:01", "0:02", "0:04", "0:08", "0:16", "0:32"]
 

提示：

输出的顺序没有要求。
小时不会以零开头，比如 “01:00” 是不允许的，应为 “1:00”。
分钟必须由两位数组成，可能会以零开头，比如 “10:2” 是无效的，应为 “10:02”。
超过表示范围（小时 0-11，分钟 0-59）的数据将会被舍弃，也就是说不会出现 "13:00", "0:61" 等时间。
### 解答
正向思维 根据位数来生成时间太复杂了。
反向思考 生成所有的时间的可能，然后计算他的位数
```
class Solution {
public:
    vector<string> readBinaryWatch(int num) {
        vector<string> res;
        int hours = 0, minutes = 0;
        string tmp ;
        for(hours = 0; hours<12; hours++)
            for( minutes = 0; minutes <60; minutes++)
            {
                if(count(hours) + count(minutes) == num)
                {
                    tmp = to_string(hours);
                    tmp +=":";
                    if (minutes<10)
                        tmp+="0";
                    tmp += to_string(minutes);
                    res.push_back(tmp);
                }
            }
        return res;
    }
    int count(int num)
    {
        int res = 0;
        while(num!=0)
        {
            if(num%2==1)
                res ++;
            num = num>>1;
        }
        return res;
    }
};
```
## 第五题
### 题目
7. 整数反转

难度
简单

给出一个 32 位的有符号整数，你需要将这个整数中每位上的数字进行反转。

示例 1:

输入: 123
输出: 321
 示例 2:

输入: -123
输出: -321
示例 3:

输入: 120
输出: 21
注意:

假设我们的环境只能存储得下 32 位的有符号整数，则其数值范围为 [−231,  231 − 1]。请根据这个假设，如果反转后整数溢出那么就返回 0。

### 解答
```
class Solution {
public:
    int reverse(int x) {
        unsigned long long result = 0;
        bool flag = 0;
        if(-2147483648>x&&x>2147483647)
            return 0;
        if(x<0)
            flag = 1;
        x = abs(x);

        while(x>0)
        {
            result *=10;
            result += x%10;
            x/=10;
            
            if(result>2147483647)
            return 0;
        }
        //result /=10;
        
        if(flag)
            result *=-1;
        
        return result;
    }
    
};
```
## 第六题
### 题目
1020. 飞地的数量
难度
中等

给出一个二维数组 A，每个单元格为 0（代表海）或 1（代表陆地）。

移动是指在陆地上从一个地方走到另一个地方（朝四个方向之一）或离开网格的边界。

返回网格中无法在任意次数的移动中离开网格边界的陆地单元格的数量。

 

示例 1：

输入：[[0,0,0,0],[1,0,1,0],[0,1,1,0],[0,0,0,0]]
输出：3
解释： 
有三个 1 被 0 包围。一个 1 没有被包围，因为它在边界上。
示例 2：

输入：[[0,1,1,0],[0,0,1,0],[0,0,1,0],[0,0,0,0]]
输出：0
解释：
所有 1 都在边界上或可以到达边界。
 

提示：

1 <= A.length <= 500
1 <= A[i].length <= 500
0 <= A[i][j] <= 1
所有行的大小都相同
### 解答
思路就是对于边界每一个1的点 广度优先搜索 遇到一个点就给他变成0

这样就排除了所有可以跑掉的点，然后再对于其他的1点进行计数

自己写的版本可能队列之类的开销太大了，超时，看了个另一个版本的和我思路一模一样的，他的方法可行

```c
/*class Solution {
public:
    int numEnclaves(vector<vector<int>>& A) {
        int dx[4] = {0,1,-1,0}, dy[4] = {1,0,0,-1};
        int row_size = A.size(), col_size = A[0].size();
        queue<vector<int>> q;
        vector<int> tmp = {0,0};
        vector<int> cur = {0,0};
        int x,y,count = 0;
        for(int j = 0; j < col_size ; j++)
        {
            if(A[0][j]==1)
            {
                tmp[0]=0;
                tmp[1]=j;
                q.push(tmp);
            }
        }
        for(int j = 0; j < col_size ; j++)
        {
            if(A[row_size-1][j]==1)
            {
                tmp[0]=row_size-1;
                tmp[1]=j;
                q.push(tmp);
            }
        }
        for(int i =1; i<row_size-1;i++)
        {
            if(A[i][0]==1)
            {
                tmp[0] = i;
                tmp[1] = 0;
                q.push(tmp);
            }
        }
        for(int i =1; i<row_size-1;i++)
        {
            if(A[i][col_size-1]==1)
            {
                tmp[0] = i;
                tmp[1] = col_size-1;
                q.push(tmp);
            }
        }
        while(!q.empty())
        {
            cur=q.front();
            q.pop();
            A[cur[0]][cur[1]] = 0;
            for(int i = 0;i<4;i++)
            {
                x=cur[0]+dx[i];
                y=cur[1]+dy[i];
                if(x>0&&x<row_size&&y>0&&y<col_size)
                {
                    if(A[x][y]==1)
                    {
                        tmp[0] = x;
                        tmp[1] = y;
                        q.push(tmp);
                    }
                }
            }
        }
        for(int i = 0; i<row_size;i++)
            for(int j = 0; j< col_size; j++)
                if(A[i][j] == 1)
                    count ++;
        return count;
    }
};*/
class Solution {
public:
    void dfs(vector<vector<int>>& A,int i,int j)
    {
        if(i<0||i>=A.size()||j<0||j>=A[i].size()||A[i][j]==0)
        {
            return;
        }
        A[i][j]=0;
        dfs(A,i-1,j);
        dfs(A,i+1,j);
        dfs(A,i,j-1);
        dfs(A,i,j+1);
    }
    
    int numEnclaves(vector<vector<int>>& A) {
        int ans = 0;
        for(int i=0;i<A.size();i++)
        {
            for(int j=0;j<A[i].size();j++)
            {
                if((i==0||j==0||i==A.size()-1||j==A[0].size()-1)&&A[i][j]==1)
                {
                    dfs(A,i,j);
                }
            }
        }
        for(int i=0;i<A.size();i++)
        {
            for(int j=0;j<A[0].size();j++)
            {
                if(A[i][j]==1)
                {
                    ans++;
                }
            }
        }
        return ans;
    }
};
```
## 第七题
### 题目
1512. 好数对的数目
难度
简单

给你一个整数数组 nums 。

如果一组数字 (i,j) 满足 nums[i] == nums[j] 且 i < j ，就可以认为这是一组 好数对 。

返回好数对的数目。

 

示例 1：

输入：nums = [1,2,3,1,1,3]
输出：4
解释：有 4 组好数对，分别是 (0,3), (0,4), (3,4), (2,5) ，下标从 0 开始
示例 2：

输入：nums = [1,1,1,1]
输出：6
解释：数组中的每组数字都是好数对
示例 3：

输入：nums = [1,2,3]
输出：0
 

提示：

1 <= nums.length <= 100
1 <= nums[i] <= 100
### 解答
双重循环效率低，用的map
```
class Solution {
public:
    int numIdenticalPairs(vector<int>& nums) {
        map<int,int> m;
        auto it=m.begin();
        int count=0;
        for(int i = 0;i<nums.size();i++)
        {
            it = m.find(nums[i]);
            if(it == m.end())
            {
                //cout<<'+'<<endl;
                m.emplace(nums[i],1);
            }
            else
            {
                
                it->second++;
                //cout<<it->first<<'-'<<it->second<<endl;
            }
        }
        for (auto i = m.begin(); i != m.end(); ++i) {
            //cout<<it->first<<' '<<it->second<<endl;
            count += calc_c(i->second);
        }
        return count;
    }
    int calc_c(int num)
    {
        if(num<2)
            return 0;
        return num *(num-1)/2;
    }
};
```
## 第八题
### 题目
454. 四数相加 II
难度
中等

给定四个包含整数的数组列表 A , B , C , D ,计算有多少个元组 (i, j, k, l) ，使得 A[i] + B[j] + C[k] + D[l] = 0。

为了使问题简单化，所有的 A, B, C, D 具有相同的长度 N，且 0 ≤ N ≤ 500 。所有整数的范围在 -228 到 228 - 1 之间，最终结果不会超过 231 - 1 。

例如:

输入:
A = [ 1, 2]
B = [-2,-1]
C = [-1, 2]
D = [ 0, 2]

输出:
2

解释:
两个元组如下:
1. (0, 0, 0, 1) -> A[0] + B[0] + C[0] + D[1] = 1 + (-2) + (-1) + 2 = 0
2. (1, 1, 0, 0) -> A[1] + B[1] + C[0] + D[0] = 2 + (-1) + (-1) + 0 = 0

### 解答
第一次暴力循环，不出意外的超时

第二次寻思数组里面有相同的数，可以弄进一个map，但是总体复杂度仍旧是n^4

看了别人的题解，用ab和cd相加的和作为键值，这样复杂度就变成了2n^2并且在第二次遍历cd时就可以得出结果

学习到了这种方法，并且感觉map用法没有那么复杂，且auto指针好神奇
```c
class Solution {
  public:
    int fourSumCount(vector<int> &A, vector<int> &B, vector<int> &C, vector<int> &D) {
        int res = 0;
        map<int, int> map;
        for (const auto &a : A) for (const auto &b : B) ++map[a + b];
        for (const auto &c : C) for (const auto &d : D) res += map[-(c + d)];
        return res;
    }
};
```
## 第九题
### 题目
682. 棒球比赛
难度
简单


你现在是一场采特殊赛制棒球比赛的记录员。这场比赛由若干回合组成，过去几回合的得分可能会影响以后几回合的得分。

比赛开始时，记录是空白的。你会得到一个记录操作的字符串列表 ops，其中 ops[i] 是你需要记录的第 i 项操作，ops 遵循下述规则：

整数 x - 表示本回合新获得分数 x
"+" - 表示本回合新获得的得分是前两次得分的总和。题目数据保证记录此操作时前面总是存在两个有效的分数。
"D" - 表示本回合新获得的得分是前一次得分的两倍。题目数据保证记录此操作时前面总是存在一个有效的分数。
"C" - 表示前一次得分无效，将其从记录中移除。题目数据保证记录此操作时前面总是存在一个有效的分数。
请你返回记录中所有得分的总和。

 

示例 1：

输入：ops = ["5","2","C","D","+"]
输出：30
解释：
"5" - 记录加 5 ，记录现在是 [5]
"2" - 记录加 2 ，记录现在是 [5, 2]
"C" - 使前一次得分的记录无效并将其移除，记录现在是 [5].
"D" - 记录加 2 * 5 = 10 ，记录现在是 [5, 10].
"+" - 记录加 5 + 10 = 15 ，记录现在是 [5, 10, 15].
所有得分的总和 5 + 10 + 15 = 30
示例 2：

输入：ops = ["5","-2","4","C","D","9","+","+"]
输出：27
解释：
"5" - 记录加 5 ，记录现在是 [5]
"-2" - 记录加 -2 ，记录现在是 [5, -2]
"4" - 记录加 4 ，记录现在是 [5, -2, 4]
"C" - 使前一次得分的记录无效并将其移除，记录现在是 [5, -2]
"D" - 记录加 2 * -2 = -4 ，记录现在是 [5, -2, -4]
"9" - 记录加 9 ，记录现在是 [5, -2, -4, 9]
"+" - 记录加 -4 + 9 = 5 ，记录现在是 [5, -2, -4, 9, 5]
"+" - 记录加 9 + 5 = 14 ，记录现在是 [5, -2, -4, 9, 5, 14]
所有得分的总和 5 + -2 + -4 + 9 + 5 + 14 = 27
示例 3：

输入：ops = ["1"]
输出：1
 

提示：

1 <= ops.length <= 1000
ops[i] 为 "C"、"D"、"+"，或者一个表示整数的字符串。整数范围是 [-3 * 104, 3 * 104]
对于 "+" 操作，题目数据保证记录此操作时前面总是存在两个有效的分数
对于 "C" 和 "D" 操作，题目数据保证记录此操作时前面总是存在一个有效的分数
### 解答
```
class Solution {
public:
    int calPoints(vector<string>& ops) {
        stack<int> s;
        int tmp1 = 0,tmp2 = 0;
        for(int i = 0; i< ops.size(); i++)
        {
            //cout<<ops[i]<<'`'<<endl;
            if(ops[i]=="C")
            {
                s.pop();
            }
            else if(ops[i] == "D")
            {
                tmp1 =s.top();

                tmp1*=2;
                //cout<<tmp1<<endl;
                s.push(tmp1);
            }
            else if(ops[i] == "+")
            {
                tmp1 = s.top();
                s.pop();
                tmp2 = s.top();
                s.push(tmp1);
                s.push(tmp1+tmp2);

            }
            else
                s.push(atoi(ops[i].c_str()));
        }
        tmp1 = 0;
        while(!s.empty())
        {
            tmp1 +=s.top();
            s.pop();
        }
        return tmp1;
    }
};
```
## 第十题
### 题目描述
86. 分隔链表
难度
中等

给你一个链表和一个特定值 x ，请你对链表进行分隔，使得所有小于 x 的节点都出现在大于或等于 x 的节点之前。

你应当保留两个分区中每个节点的初始相对位置。

 

示例：

输入：head = 1->4->3->2->5->2, x = 3
输出：1->2->2->4->3->5
### 解答
不是很明白为什么传指针不好使，所以写的特别绕
讨论了一下明白了
&作为操作符 意思为取地址的意思 在函数定义时 是引用的意思

```c
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    ListNode* partition(ListNode* head, int x) {
        ListNode * head_min = NULL;
        ListNode * tail_min = NULL;
        ListNode * head_max = NULL;
        ListNode * tail_max = NULL;
        while(head!=NULL)
        {
            if(head->val<x)
            {
                if(!head_min)
                {
                    head_min = insert_node(head_min,tail_min,head->val);
                    tail_min = head_min;
                }
                else
                {
                    tail_min = insert_node(head_min,tail_min,head->val);
                }
            }
                
            else
            {
                  if(!head_max)
                {
                    head_max = insert_node(head_max,tail_max,head->val);
                    tail_max = head_max;
                }
                else
                {
                    tail_max = insert_node(head_max,tail_max,head->val);
                }             
            }
            head = head->next;
        }
        if(!head_min)
            return head_max;
        if(tail_min)
        {
            cout<<'-'<<endl;

            tail_min->next = head_max;
        }

        return head_min;

    }
    ListNode * insert_node(ListNode *head,ListNode * tail, int val)
    {
        cout<<val<<endl;
        ListNode * tmp = new ListNode(0);
        tmp ->val = val;
        if(head==NULL)
        {

            head = tmp;
            tail = head;
            return head;
        }
        else
        {
            tail->next = tmp;
            tail = tmp;
            return tail;
        }
    }
};
```
```c
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    ListNode* partition(ListNode* head, int x) {
        ListNode * head_min = NULL;
        ListNode * tail_min = NULL;
        ListNode * head_max = NULL;
        ListNode * tail_max = NULL;
        while(head!=NULL)
        {
            if(head->val<x)
                insert_node(head_min,tail_min,head->val);
            else
                insert_node(head_max,tail_max,head->val);
            head = head->next;
        }
        if(!head_min)
            return head_max;
        if(tail_min)
        {
            cout<<'-'<<endl;

            tail_min->next = head_max;
        }

        return head_min;

    }
    ListNode * insert_node(ListNode *&head,ListNode * &tail, int val)
    {
        cout<<val<<endl;
        ListNode * tmp = new ListNode(0);
        tmp ->val = val;
        if(head==NULL)
        {

            head = tmp;
            tail = head;
            return head;
        }
        else
        {
            tail->next = tmp;
            tail = tmp;
            return tail;
        }
    }
};
```
## 第十一题
### 题目描述
对于非负整数 X 而言，X 的数组形式是每位数字按从左到右的顺序形成的数组。例如，如果 X = 1231，那么其数组形式为 [1,2,3,1]。

给定非负整数 X 的数组形式 A，返回整数 X+K 的数组形式。

 

示例 1：

输入：A = [1,2,0,0], K = 34
输出：[1,2,3,4]
解释：1200 + 34 = 1234
示例 2：

输入：A = [2,7,4], K = 181
输出：[4,5,5]
解释：274 + 181 = 455
示例 3：

输入：A = [2,1,5], K = 806
输出：[1,0,2,1]
解释：215 + 806 = 1021


### 解答
思路就是弄个vector存储每一步的结果，因为k和a两个数的长度不知道多大，所以不能订下来数组长度，在最后倒置即可
```c

class Solution {
public:
    vector<int> addToArrayForm(vector<int>& A, int K) {
        int carry = 0,tmp = 0;
        vector<int>result = {};
        for(int i = A.size() - 1;i >= 0;i--)
        {
            tmp = K%10;
            tmp += carry;
            tmp += A[i];
            K/=10;
            if(tmp >= 10)
            {
                result.push_back(tmp%10);
                carry = 1;
            }
            else
            {
                result.push_back(tmp%10);
                carry = 0;
            }
        }
        while(K>0)
        {
            tmp = K%10;
            tmp += carry;
            K/=10;
            if(tmp >= 10)
            {
                result.push_back(tmp%10);
                carry = 1;
            }
            else
            {
                result.push_back(tmp%10);
                carry = 0;
            }
        }
        if(carry==1)
        {
            result.push_back(1);
        }
        for(int i = 0; i < result.size()/2; i++)
        {
            tmp = result[i];
            result[i] = result[result.size() - 1 - i];
            result[result.size() - 1 - i] = tmp;
        }
        return result;
    }
};
```
## 第十二题
### 题目描述
219. 存在重复元素 II
难度
简单

给定一个整数数组和一个整数 k，判断数组中是否存在两个不同的索引 i 和 j，使得 nums [i] = nums [j]，并且 i 和 j 的差的 绝对值 至多为 k。

 

示例 1:

输入: nums = [1,2,3,1], k = 3
输出: true
示例 2:

输入: nums = [1,0,1,1], k = 1
输出: true
示例 3:

输入: nums = [1,2,3,1,2,3], k = 2
输出: false
### 解答
这道题暴力法超时，于是乎用字典做，遍历一次数组，每次遇到一个数，如果字典里面没有就加入字典，有的话看原来的下标和现在这个下标差值符不符合要求，符合就返回，不符合就更新一下，因为正序遍历，递增的。
```
class Solution {
public:
    bool containsNearbyDuplicate(vector<int>& nums, int k) {
        map<int,int> m;
        for(int i = 0; i<nums.size();i++)
        {
            if(m.count(nums[i])==0)//insert
                m.insert({nums[i],i});
            else
            {
                if(abs(m[nums[i]] - i)<=k)
                    return true;
                else
                    m[nums[i]] = i;
            }
        }
        return false;
    }
};
```
## 第十三题
### 题目描述
852. 山脉数组的峰顶索引
难度
简单

符合下列属性的数组 arr 称为 山脉数组 ：
arr.length >= 3
存在 i（0 < i < arr.length - 1）使得：
arr[0] < arr[1] < ... arr[i-1] < arr[i]
arr[i] > arr[i+1] > ... > arr[arr.length - 1]
给你由整数组成的山脉数组 arr ，返回任何满足 arr[0] < arr[1] < ... arr[i - 1] < arr[i] > arr[i + 1] > ... > arr[arr.length - 1] 的下标 i 。

 

示例 1：

输入：arr = [0,1,0]
输出：1
示例 2：

输入：arr = [0,2,1,0]
输出：1
示例 3：

输入：arr = [0,10,5,2]
输出：1
示例 4：

输入：arr = [3,4,5,1]
输出：2
示例 5：

输入：arr = [24,69,100,99,79,78,67,36,26,19]
输出：2
 

提示：

3 <= arr.length <= 104
0 <= arr[i] <= 106
题目数据保证 arr 是一个山脉数组
### 解答
二分法
```

class Solution {
public:
    int peakIndexInMountainArray(vector<int>& arr) {
        int head= 0,tail = arr.size() - 1,mid = (head+tail)/2;
        while(!(arr[mid]>arr[mid+1]&&arr[mid]>arr[mid-1]))
        {
            if(arr[mid]<arr[mid+1])
            {
                head = mid;
            }
            else
                tail = mid;
            mid = (head+tail)/2;
        }
        return mid;
    }
};
```
## 第十四题
### 题目描述
1491. 去掉最低工资和最高工资后的工资平均值
难度
简单


给你一个整数数组 salary ，数组里每个数都是 唯一 的，其中 salary[i] 是第 i 个员工的工资。

请你返回去掉最低工资和最高工资以后，剩下员工工资的平均值。

 

示例 1：

输入：salary = [4000,3000,1000,2000]
输出：2500.00000
解释：最低工资和最高工资分别是 1000 和 4000 。
去掉最低工资和最高工资以后的平均工资是 (2000+3000)/2= 2500
示例 2：

输入：salary = [1000,2000,3000]
输出：2000.00000
解释：最低工资和最高工资分别是 1000 和 3000 。
去掉最低工资和最高工资以后的平均工资是 (2000)/1= 2000
示例 3：

输入：salary = [6000,5000,4000,3000,2000,1000]
输出：3500.00000
示例 4：

输入：salary = [8000,9000,2000,3000,6000,1000]
输出：4750.00000
 

提示：

3 <= salary.length <= 100
10^3 <= salary[i] <= 10^6
salary[i] 是唯一的。
与真实值误差在 10^-5 以内的结果都将视为正确答案。
### 解答
```
class Solution {
public:
    double average(vector<int>& salary) {
        double min = 1000000,max = 1000,sum = 0;
        for(int i = 0; i< salary.size(); i++)
        {
            sum += salary[i];
            if(salary[i]>max)
                max = salary[i];
            if(salary[i]<min)
                min = salary[i];
        }
        sum -= min;
        sum -= max;
        return sum/(salary.size() - 2);
    }
};
```
## 第十五题
### 题目描述
678. 有效的括号字符串
难度
中等

给定一个只包含三种字符的字符串：（ ，） 和 *，写一个函数来检验这个字符串是否为有效字符串。有效字符串具有如下规则：

任何左括号 ( 必须有相应的右括号 )。
任何右括号 ) 必须有相应的左括号 ( 。
左括号 ( 必须在对应的右括号之前 )。
* 可以被视为单个右括号 ) ，或单个左括号 ( ，或一个空字符串。
一个空字符串也被视为有效字符串。
示例 1:

输入: "()"
输出: True
示例 2:

输入: "(*)"
输出: True
示例 3:

输入: "(*))"
输出: True
注意:

字符串大小将在 [1，100] 范围内。
### 解答
这道题难点在于*的处理，因为其既可以作为空字符串，也可以充当任意一个括号

我的思路是用一个栈，通过栈匹配的方式搞定右括号的情况，换言之就是*充当左括号，这里的规则如下，遇到左括号或者*，入栈，并记录其总数，遇到右括号，优先在栈里面找左括号，没有的话在找*

搞完这一套之后栈里面，也就是原先字符串里面就只剩下左括号和*，这时候再把*作为右括号看待，这样就好弄多了。
```
class Solution {
public:
    bool checkValidString(string s) {
        int lcount = 0,scount = 0,count = 0;
        stack<char> s1;
        for(int i = 0 ; i< s.size(); i++)
        {
            if(s[i] == '(')
            {
                lcount ++;
                s1.push('(');
            }
            else if(s[i] == '*')
            {
                scount ++;
                s1.push('*');
            }
            else// encounter )
            {
                if(s1.empty())
                    return false;
                if(lcount == 0)// only * left
                {
                    s1.pop();
                    scount --;
                }
                else if(scount == 0)//only ( left
                {
                    s1.pop();
                    lcount -- ;
                }
                else//now there are * and ( in the stack
                {
                    //first use (
                    while(s1.top()!='(')
                    {
                        s1.pop();
                        count ++;
                    }
                    s1.pop();
                    lcount--;
                    //push back *
                    while(count!=0)
                    {
                        s1.push('*');
                        count -- ;
                    }
                }
            }
        }
        if(lcount > scount)
            return false;
        count = 0;
        while(!s1.empty())
        {
            if(s1.top()=='*')
                count ++;
            else
            {
                count--;
                if(count<0)
                    return false;
            }
            s1.pop();
        }
        return true;
    }
};
```
## 第十六题
### 题目描述
206. 反转链表
难度
简单


反转一个单链表。

示例:

输入: 1->2->3->4->5->NULL
输出: 5->4->3->2->1->NULL
进阶:
你可以迭代或递归地反转链表。你能否用两种方法解决这道题？
### 解答
```
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
    ListNode* reverseList(ListNode* head) {
        stack<int> s;
        ListNode * cur = head;
        while(cur)
        {
            s.push(cur->val);
            cur = cur->next;
        }
        cur = head;
        while(cur)
        {
            cur->val = s.top();
            s.pop();
            cur = cur->next;
        }
        return head;
    }
};
```
## 第十七题
### 题目描述
1328. 破坏回文串
难度
中等

给你一个回文字符串 palindrome ，请你将其中 一个 字符用任意小写英文字母替换，使得结果字符串的字典序最小，且 不是 回文串。

请你返回结果字符串。如果无法做到，则返回一个空串。

 

示例 1：

输入：palindrome = "abccba"
输出："aaccba"
示例 2：

输入：palindrome = "a"
输出：""
 

提示：

1 <= palindrome.length <= 1000
palindrome 只包含小写英文字母。
### 解答
只破坏一个，还要求字典序最小，那就碰到第一个不是a的就改成a呗，除了奇数情况下中间节点

要全是a的话，那就把最后一个字符变成b
```
class Solution {
public:
    string breakPalindrome(string palindrome) {
        int n = palindrome.size();
        if(n == 1)
            return "";
        for(int i = 0; i<(n/2);i++)
        {
            if(palindrome[i] != 'a')
            {
                palindrome[i] = 'a';
                return palindrome;
            }   
        }
        palindrome[n-1] = 'b';


        return palindrome;
    }
};
```
## 第十八题
### 题目描述
面试题 16.13. 平分正方形
难度
中等

给定两个正方形及一个二维平面。请找出将这两个正方形分割成两半的一条直线。假设正方形顶边和底边与 x 轴平行。

每个正方形的数据square包含3个数值，正方形的左下顶点坐标[X,Y] = [square[0],square[1]]，以及正方形的边长square[2]。所求直线穿过两个正方形会形成4个交点，请返回4个交点形成线段的两端点坐标（两个端点即为4个交点中距离最远的2个点，这2个点所连成的线段一定会穿过另外2个交点）。2个端点坐标[X1,Y1]和[X2,Y2]的返回格式为{X1,Y1,X2,Y2}，要求若X1 != X2，需保证X1 < X2，否则需保证Y1 <= Y2。

若同时有多条直线满足要求，则选择斜率最大的一条计算并返回（与Y轴平行的直线视为斜率无穷大）。

示例：

输入：
square1 = {-1, -1, 2}
square2 = {0, -1, 2}
输出： {-1,0,2,0}
解释： 直线 y = 0 能将两个正方形同时分为等面积的两部分，返回的两线段端点为[-1,0]和[2,0]
提示：

square.length == 3
square[2] > 0
### 解答
这个线肯定经过正方形中心，所以线好确定，那么点呢？

点要求最远的两点，那么就要讨论两种情况，斜率绝对值小于一，左右边，大于一上下边，有了这个结论，就好确定了
```
class Solution {
public:
    vector<double> cutSquares(vector<int>& square1, vector<int>& square2) {
        double  x1 = square1[0]+square1[2]/2.0,
                y1 = square1[1]+square1[2]/2.0,
                x2 = square2[0]+square2[2]/2.0,
                y2 = square2[1]+square2[2]/2.0;
        vector<double>result(4);
        if(x1==x2)//k infinite
        {
            //rerturn low coord + high coord
            result[0] = x1;
            result[1] = min(square1[1],square2[1]);
            result[2] = x2;
            result[3] = max(square1[1]+square1[2],square2[1]+square2[2]);
        }
        else// there is k, and must calculate two coord
        {
            ///calc k and b
            double k = (y1-y2)/(x1-x2);
            double b = y1 - k*x1;
            // two coord

            if(abs(k) < 1)
            {
                result[0] = min(square1[0],square2[0]);
                result[1] = k*result[0] + b;
                result[2] = max(square1[0]+square1[2],square2[0]+square2[2]);
                result[3] = k*result[2] + b;
            }
            else
            {
                result[1] = min(square1[1],square2[1]);
                result[0] = (result[1] - b)/k;
                result[3] = max(square1[1]+square1[2],square2[1]+square2[2]);
                result[2] = (result[3] - b)/k;
                if(result[0]>=result[2])
                {
                    double tmp = result[1];
                    result[1] = result[3];
                    result[3] = tmp;
                    tmp = result[0];
                    result[0] = result[2];
                    result[2] = tmp; 
                }
            }
        }
        return result;
    }
};
```
## 第十九题
### 题目描述
面试题 04.02. 最小高度树
难度
简单


给定一个有序整数数组，元素各不相同且按升序排列，编写一个算法，创建一棵高度最小的二叉搜索树。

示例:
给定有序数组: [-10,-3,0,5,9],

一个可能的答案是：[0,-3,9,-10,null,5]，它可以表示下面这个高度平衡二叉搜索树：

          0 
         / \ 
       -3   9 
       /   / 
     -10  5 
### 解答
这道题主要需要注意边界情况，0，1，2长度的情况
```
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    TreeNode* sortedArrayToBST(vector<int>& nums) {
        if(nums.size()==0)
            return NULL;

        return generate(nums,0,nums.size()-1);
    }
    TreeNode* generate(vector<int>& nums,int start,int end)
    {

        int mid = (start+end)/2;
        TreeNode * tmp = new TreeNode;
        
        if(start==end)//one left
        {
            tmp->val = nums[start];
            return tmp;
        }
            
        if(end - start == 1)// two left
        {
            tmp->val = nums[start];
            TreeNode * tmp2 = new TreeNode;
            tmp2->val = nums[end];
            tmp->right = tmp2;
            return tmp;
        }
        tmp->val = nums[mid];
        tmp->left = generate(nums,start,mid-1);
        tmp->right = generate(nums,mid+1,end);
        return tmp;
    }
};
```
## 第二十题
### 题目描述
1186. 删除一次得到子数组最大和
难度
中等


给你一个整数数组，返回它的某个 非空 子数组（连续元素）在执行一次可选的删除操作后，所能得到的最大元素总和。

换句话说，你可以从原数组中选出一个子数组，并可以决定要不要从中删除一个元素（只能删一次哦），（删除后）子数组中至少应当有一个元素，然后该子数组（剩下）的元素总和是所有子数组之中最大的。

注意，删除一个元素后，子数组 不能为空。

请看示例：

示例 1：

输入：arr = [1,-2,0,3]
输出：4
解释：我们可以选出 [1, -2, 0, 3]，然后删掉 -2，这样得到 [1, 0, 3]，和最大。
示例 2：

输入：arr = [1,-2,-2,3]
输出：3
解释：我们直接选出 [3]，这就是最大和。
示例 3：

输入：arr = [-1,-1,-1,-1]
输出：-1
解释：最后得到的子数组不能为空，所以我们不能选择 [-1] 并从中删去 -1 来得到 0。
     我们应该直接选择 [-1]，或者选择 [-1, -1] 再从中删去一个 -1。
 

提示：

1 <= arr.length <= 10^5
-10^4 <= arr[i] <= 10^4
### 解答
这应该是我这次刷题遇到的最难的题了，神奇的动态规划，最近一定要搞懂！！！！

我一开始的思路是像求最大子序列的和一样，然后遇到负数的话考虑删不删掉，这里我是每次都记录删的元素，然后遇到新的负数就比较他俩大小，要是比他更小，那就删，否则就不管。

看起来没啥问题，但是遇到这样的序列：11，-10，-11，7，8，9，10，-6，8. 按理说从头开始加到最后，删掉-11，整个过程也没出现负的情况，但是其实从7开始，删掉-6，这样才是最优解。搞不定

看提示，说每次假装删除这个点，计算两边序列的最大值。n方的复杂度，是没有错的，但是超时。

看分类，动态规划，我服了。。。 搞不明白什么是动态规划，也不知道从何下手。

我已经在这道题用了3个小时以上了，按理说要是复试这道题，我估计又和去年一样被干掉。脑子昏，夜也深了，实在是没办法了看了眼评论区的答案。

好家伙，我写了将近一百行，超时报错，人家十几行完美解决，这就是动态规划的魅力吗。。。

思路是这样的，dp矩阵nx2，dp[i][0]代表末尾为i，一个没删的最大和，dp[i][1]代表末尾为i，删了一个的最大值。

那么初始条件就是dp[0][0] = arr[0],dp[0][1] = 0.

这样遍历一遍数组，dp[i][0] = arr[i] + max(0,dp[i-1][0])，很好理解，如果dp[i-1][0]小于0，那就不要了，自立门户。dp[i][1] = max(dp[i-1][0],dp[i-1][1] + arr[i]),同样很好理解，既然是删掉一个的情况，那么要么删当前值，要么删之前的，之前的删哪个？上一步已经给你算出来了（动态规划的精妙之处）。然后取最大值，就解决了。
代码
```
class Solution {
public:
    int maximumSum(vector<int>& arr) {
       const int n = arr.size();
       int dp[n][2];
       dp[0][0] = arr[0];
       dp[0][1] = 0;
       int result = arr[0];
       for(int i = 1; i < n ; i ++)
       {
           dp[i][0] = max(0,dp[i-1][0]) + arr[i];
           dp[i][1] = max(dp[i-1][0],arr[i] + dp[i-1][1]);
           result = max(max(result,dp[i][0]),dp[i][1]);
       }
       return result;
    }
};
```
## 第21题
### 题目描述
5. 最长回文子串
难度
中等


给你一个字符串 s，找到 s 中最长的回文子串。

 

示例 1：

输入：s = "babad"
输出："bab"
解释："aba" 同样是符合题意的答案。
示例 2：

输入：s = "cbbd"
输出："bb"
示例 3：

输入：s = "a"
输出："a"
示例 4：

输入：s = "ac"
输出："a"
 

提示：

1 <= s.length <= 1000
s 仅由数字和英文字母（大写和/或小写）组成
### 解答
依然是一个动态规划题目，自己想了好久之后还是没有整明白，看了题解的思路，和我的思路差不多，不同的地方是他的循环是以回文串长度作为循环条件的，这也是他的方法的独到之处。

dp[i][j]意思就是以i为开头，j为结尾的子串是否是回文串，这个我想到了，但是怎么进行下一步计算呢？

我刚开始的思路是存储的值为回文串的长度，但是没有想到递推的关系，也就是状态转移方程

首先长度为1时，就是单个字符，这个是肯定成立的

长度为2时，考虑他和前一个字符是否相同，相同的话就是成立

其他情况，dp[i][j] = (dp[i+1][j-1] && s[i] == s[j]),也就是说先看前面的是不是已经是一个回文串，是的话就看两端，相同的话那肯定就成立了。

那么怎么才能做到每一步的结果可以用到上一步的结果呢？其实我就这一点没搞懂，但是有了上述的思路之后，就很明了了：

既然每一步都需要比较里面那一截，而且这一截和下标没啥关系，那么有关系成规律的是什么呢？长度！

于是乎每次就以长度作为循环节，就可以解决了。

```
class Solution {
public:
    string longestPalindrome(string s) {
        int n = s.size(),max = 1;
        vector<vector<int>> dp (n,vector<int>(n,0));
        // row start, col end, val means is a palindrome or not
        dp[0][0] = 1;
        string result = "";
        int j;
        for(int l = 0; l<n ; l++)// len of palindrome
        {
            for(int i = 0; i+l<n; i++)
            {
                j = i+l;
                if(l==0)
                {
                    dp[i][j] = 1;
                }
                else if(l==1)
                {
                    if(s[i] == s[j])
                        dp[i][j] = 1;
                }
                else
                {
                    if(dp[i+1][j-1]&&s[i]==s[j])
                        dp[i][j] = 1;
                }
                if(dp[i][j]&& l+1 > result.size())
                    result = s.substr(i,l+1);
            }
        }
        return result;
    }
};
```
## 第22题
### 题目描述
10. 正则表达式匹配
难度
困难

给你一个字符串 s 和一个字符规律 p，请你来实现一个支持 '.' 和 '*' 的正则表达式匹配。

'.' 匹配任意单个字符
'*' 匹配零个或多个前面的那一个元素
所谓匹配，是要涵盖 整个 字符串 s的，而不是部分字符串。

 
示例 1：

输入：s = "aa" p = "a"
输出：false
解释："a" 无法匹配 "aa" 整个字符串。
示例 2:

输入：s = "aa" p = "a*"
输出：true
解释：因为 '*' 代表可以匹配零个或多个前面的那一个元素, 在这里前面的元素就是 'a'。因此，字符串 "aa" 可被视为 'a' 重复了一次。
示例 3：

输入：s = "ab" p = ".*"
输出：true
解释：".*" 表示可匹配零个或多个（'*'）任意字符（'.'）。
示例 4：

输入：s = "aab" p = "c*a*b"
输出：true
解释：因为 '*' 表示零个或多个，这里 'c' 为 0 个, 'a' 被重复一次。因此可以匹配字符串 "aab"。
示例 5：

输入：s = "mississippi" p = "mis*is*p*."
输出：false
 

提示：

0 <= s.length <= 20
0 <= p.length <= 30
s 可能为空，且只包含从 a-z 的小写字母。
p 可能为空，且只包含从 a-z 的小写字母，以及字符 . 和 *。
保证每次出现字符 * 时，前面都匹配到有效的字符
### 解答
第一次挑战苦难题目，虽然自己的版本有点问题，但是我的思路是正确的，而且与最终答案大体相似

功亏一篑，比较可惜

我的版本，在处理*时有些问题
```
class Solution {
public:
    bool isMatch(string s, string p) {
        int m = s.size(), n = p.size();
        if(m==0&&n==0)
            return true;
        if(m == 0)
        {
            int i = n-1;
            while(i>=0)
            {
                if(p[i] == '*')
                    i-=2;
                else    
                    return false;
            }
            return true;
        }
        if(n==0)
            return false;
        vector< vector<int> > dp(m + 1,vector<int> (n + 1,0));
        
        dp[0][0] = 1;
        //s[i-1] p[j-1]
        for(int i = 1; i<m+1; i++)
        {
            for(int j = 1; j< n+1; j++)
            {
                if(p[j-1] == '.')
                {
                    if(j>1&& p[j-2] == '*')
                    {
                        dp[i][j] = dp[i-1][j-1] || dp[i][j-2] || dp[i-1][j-3];
                    }
                    else
                        dp[i][j] = dp[i-1][j-1];
                }
                else if(p[j-1] == '*')
                {
                    if(p[j-2] == '.' || p[j-2] == s[i-1])
                        dp[i][j] = (dp[i-1][j-1] || dp[i][j-1] || dp[i-1][j] || dp[i][j-2]);
                    else 
                        if(j>1)
                            dp[i][j] = dp[i][j-2];
                }
                else // alphabet
                {
                    if(s[i-1] == p[j-1])//same as '.'
                        if(dp[i-1][j-1])
                            dp[i][j] = dp[i-1][j-1];
                        else
                        {
                            if(j>2&&p[j-2] == '*' )
                                dp[i][j] = dp[i-1][j-3];
                        }

                }
            }
        }
        for(int i = 1; i<m+1; i++)
        {
            for(int j = 1; j< n+1; j++)
                cout<<dp[i][j]<<' ';
            cout<<endl;
        }

        return dp[m][n];
        
    }
};
```
正确版本
```
class Solution {
public:
    bool isMatch(string s, string p) {
        int m = s.size(), n = p.size();
        if(m==0&&n==0)
            return true;
        if(m == 0)
        {
            int i = n-1;
            while(i>=0)
            {
                if(p[i] == '*')
                    i-=2;
                else    
                    return false;
            }
            return true;
        }
        if(n==0)
            return false;
        vector< vector<int> > dp(m + 1,vector<int> (n + 1,0));
        
        dp[0][0] = 1;
        //s[i-1] p[j-1]
        for (int j = 1; j < n+1; j++) 
        {
            if (p[j - 1] == '*') dp[0][j] = dp[0][j - 2];
        }


        for (int i = 1; i < m + 1; i++) 
        {
            for (int j = 1; j < n + 1; j++) 
            {
                if (s[i - 1] == p[j - 1] || p[j - 1] == '.') 
                {
                    dp[i][j] = dp[i - 1][j - 1];
                } 
                else if (p[j - 1] == '*') 
                {
                    if (s[i - 1] == p[j - 2] || p[j - 2] == '.') 
                    {
                        dp[i][j] = dp[i][j - 2] || dp[i - 1][j - 2] || dp[i - 1][j];
                    } 
                    else 
                    {
                       dp[i][j] = dp[i][j - 2];
                    }
                }
            }
        }

        for(int i = 1; i<m+1; i++)
        {
            for(int j = 1; j< n+1; j++)
                cout<<dp[i][j]<<' ';
            cout<<endl;
        }

        return dp[m][n];
        
    }
};
```
## 第23题
### 题目描述
1716. 计算力扣银行的钱
难度
简单

Hercy 想要为购买第一辆车存钱。他 每天 都往力扣银行里存钱。

最开始，他在周一的时候存入 1 块钱。从周二到周日，他每天都比前一天多存入 1 块钱。在接下来每一个周一，他都会比 前一个周一 多存入 1 块钱。

给你 n ，请你返回在第 n 天结束的时候他在力扣银行总共存了多少块钱。

 

示例 1：

输入：n = 4
输出：10
解释：第 4 天后，总额为 1 + 2 + 3 + 4 = 10 。
示例 2：

输入：n = 10
输出：37
解释：第 10 天后，总额为 (1 + 2 + 3 + 4 + 5 + 6 + 7) + (2 + 3 + 4) = 37 。注意到第二个星期一，Hercy 存入 2 块钱。
示例 3：

输入：n = 20
输出：96
解释：第 20 天后，总额为 (1 + 2 + 3 + 4 + 5 + 6 + 7) + (2 + 3 + 4 + 5 + 6 + 7 + 8) + (3 + 4 + 5 + 6 + 7 + 8) = 96 。
 

提示：

1 <= n <= 1000
### 解答
```
class Solution {
public:
    int totalMoney(int n) {
        int week = n/7;
        int day = n%7;
        int result = 0;
        int carry = 3;
        for(int i = 0 ; i<week; i++)
        {
            carry++;
            result += carry*7;
        }
        if(day)
        {
            carry -=2;
            for(int i = 0;i<day;i++)
            {
                result+=carry;
                carry++;
            }

        }
        return result;
    }
};
```
## 第24题
### 题目描述
164. 最大间距
难度
困难

给定一个无序的数组，找出数组在排序之后，相邻元素之间最大的差值。

如果数组元素个数小于 2，则返回 0。

示例 1:

输入: [3,6,9,1]
输出: 3
解释: 排序后的数组是 [1,3,6,9], 其中相邻元素 (3,6) 和 (6,9) 之间都存在最大差值 3。
示例 2:

输入: [10]
输出: 0
解释: 数组元素个数小于 2，因此返回 0。
说明:

你可以假设数组中所有元素都是非负整数，且数值在 32 位有符号整数范围内。
请尝试在线性时间复杂度和空间复杂度的条件下解决此问题。
### 解答
题目说线性，我首先想的就是不排序
我的思路是这样的，弄一个5元组，记录当前数组的头尾，最大间隔的头尾，以及最大间隔长度
首先边界情况，即长度为2一下的时候先搞定
长度大于2，用nums1和nums0初始化五元组，从2开始遍历，遇到的元素有四种情况，比头小，比尾部大，在maxgap中间，以及其他的情况，分别处理
这样一趟下来得到的不是最优的，但是我们可以好几趟遍历这样。我想的是这个过程弄好几次，直到取到极大值。但是还是有问题，因为并不是每次取到的值都是递增的，所以得不到正确答案。
我的代码
```
class Solution {
public:
    int maximumGap(vector<int>& nums) {
        int n = nums.size();
        int gap = 0;//record tmp gap
        int global_max = -1,local_max = 0;
        int cur_end = 0;
        vector<int> record (5,0);
        if(n<2)
            return 0;
        if(n==2)
            return abs(nums[1]-nums[0]);
        // sort means nlogn
        // so no need to sort completely
        // how about we just focus on those max gap pairs?
        // record cur max gap pair's set, add next element to this set
        // then consider change the max or not
        
        //record[0] start of sorted array
        //record[1] start of max gap
        //record[2] end of max gap
        //record[3] end of sorted array
        //record[4] max len
        while(global_max < local_max)
        {
            global_max = local_max;
            if(nums[0]>=cur_end)
                nums[0] -= (local_max);
            if(nums[1]>=cur_end)
                nums[1] -= (local_max);
            if(nums[0]<nums[1])
            {
                record[0] = nums[0];
                record[1] = nums[0];
                record[2] = nums[1];
                record[3] = nums[1];
                record[4] = nums[1]-nums[0];
            }
            else
            {
                record[0] = nums[1];
                record[1] = nums[1];
                record[2] = nums[0];
                record[3] = nums[0];
                record[4] = nums[0]-nums[1];
            }
            for(int i=2;i<n;i++)
            {
                if(nums[i]>=cur_end)
                    nums[i]-=(local_max);
                if(nums[i]<record[0])
                {
                //in the front of the array
                    gap = record[0] - nums[i];
                    record[0]=nums[i];
                    if(gap > record[4])
                    {
                        record[4] = gap;
                        record[2] = nums[i] + gap;
                        record[1] = nums[i];
                    }
                }
                else if(nums[i]>record[3])//end of the array
                {
                    gap = nums[i] - record[3];
                    record[3] = nums[i];
                    if(gap > record[4])
                    {
                        record[4] = gap;
                        record[1] = nums[i] - gap;
                        record[2] = nums[i];
                    }
                }
                else if(record[1]<nums[i]&&nums[i]<record[2])
                {
                    //in the middle of the max gap
                    // now max gap shrinked
                    // how we get current max gap?
                // we need to compare current gap with sencond\third\4th largest gap
                //in that case, why dont we just sort the array?
                // so we need to record every gap that larger than max/2
                //map maybe worked
                // no it wont work
                    gap = nums[i] - record[1];
                    if(gap >= record[4] - gap)
                    {
                        record[4] = gap;
                        record[2] = nums[i];
                    }
                    else
                    {
                        record[4] = record[4] - gap;
                        record[1] = nums[i];
                    }
                }
            }
            local_max = record[4];
            cur_end = record[2];
            cout<<"local: "<<local_max<<'-';
            cout<<"global: "<<global_max<<endl;
            cout<<"end: "<<cur_end<<endl;
        }
            return global_max;

    }
};
```
我的思路如果要得到正确答案，那么复杂度和排序没什么不同，甚至还要比排序高。
于是乎看了眼评论区，我以为大家都是和我一样的思路，看了才知道全都是排序做的。。。
vector排序谁都会，而且这道题vector排序也能过，但是我准备学习一下桶排序，也就是时间复杂度和空间复杂度都是n的桶排序。

桶里面元素的数量为m=(max-min)/n-1,桶的数量为（min-max）/m + 1

这样把所有元素放进桶里面，然后再遍历一遍桶，就能得到最大间距了
```
class Solution {
public:
    int maximumGap(vector<int>& nums) {
        int n = nums.size();
        if(n<2)
            return 0;
        if(n==2)
            return abs(nums[1]-nums[0]);
        int max_ = 0,min_ = INT_MAX;
        
        int index = 0;
        for(int i = 0; i< n ; i++)
        {
            if(nums[i]>max_)
                max_ = nums[i];
            if(nums[i]<min_)
                min_ = nums[i];
        }
        int gap = max((max_-min_)/(n-1),1);
        vector< vector<int>> bucket((max_-min_)/gap+1);
        for(int i = 0;i<n;i++)
        {
            index = (nums[i] - min_)/gap;
            bucket[index].push_back(nums[i]);
        }
        int max_gap = 0,prev_max = INT_MAX;
        for(int i =0;i<bucket.size();i++)
        {
            if(bucket[i].size()&& prev_max!=INT_MAX)
            {
                max_gap = max(max_gap,*min_element(bucket[i].begin(),bucket[i].end())-prev_max);
            }
            if(bucket[i].size())
            {
                prev_max = *max_element(bucket[i].begin(),bucket[i].end());
            }
        }
        return max_gap;

    }
};
```
## 第25题
### 题目描述
849. 到最近的人的最大距离
难度
中等

给你一个数组 seats 表示一排座位，其中 seats[i] = 1 代表有人坐在第 i 个座位上，seats[i] = 0 代表座位 i 上是空的（下标从 0 开始）。

至少有一个空座位，且至少有一人已经坐在座位上。

亚历克斯希望坐在一个能够使他与离他最近的人之间的距离达到最大化的座位上。

返回他到离他最近的人的最大距离。

 

示例 1：


输入：seats = [1,0,0,0,1,0,1]
输出：2
解释：
如果亚历克斯坐在第二个空位（seats[2]）上，他到离他最近的人的距离为 2 。
如果亚历克斯坐在其它任何一个空位上，他到离他最近的人的距离为 1 。
因此，他到离他最近的人的最大距离是 2 。 
示例 2：

输入：seats = [1,0,0,0]
输出：3
解释：
如果亚历克斯坐在最后一个座位上，他离最近的人有 3 个座位远。
这是可能的最大距离，所以答案是 3 。
示例 3：

输入：seats = [0,1]
输出：1
 

提示：

2 <= seats.length <= 2 * 104
seats[i] 为 0 或 1
至少有一个 空座位
至少有一个 座位上有人
### 解答
中等就这？
```
class Solution {
public:
    int maxDistToClosest(vector<int>& seats) {
        //find first occupied seat
        int cur_max = 0, n = seats.size();
        int cur_gap = 0,prev = 0;
        for(cur_max = 0; cur_max<n && seats[cur_max] == 0; cur_max++);
        prev = cur_max;
        for(int i = cur_max + 1;i<n;i++)
        {
            if(seats[i] == 1)
            {
                cur_gap = (i-prev)/2;
                if(cur_gap>cur_max)
                    cur_max = cur_gap;
                cur_gap = 0;
                prev = i;
            }
            else
            {    
                cur_gap ++;
            }
        }
        if(cur_gap>cur_max)
            cur_max = cur_gap;
        return cur_max;
    }
};
```
## 第26题
### 题目描述
343. 整数拆分
难度
中等

给定一个正整数 n，将其拆分为至少两个正整数的和，并使这些整数的乘积最大化。 返回你可以获得的最大乘积。

示例 1:

输入: 2
输出: 1
解释: 2 = 1 + 1, 1 × 1 = 1。
示例 2:

输入: 10
输出: 36
解释: 10 = 3 + 3 + 4, 3 × 3 × 4 = 36。
说明: 你可以假设 n 不小于 2 且不大于 58。

### 解答
又是一道奇奇怪怪的题目，看起来挺难其实emmmmm
题目标签是动态规划，我想了半天不知道怎么规划，于是我用一个看起来很low的方法，复杂度也是n，0ms，击败100%对手。。。
评论区的数学方法：
数学方法，求函数y=(n/x)^x(x>0)的最大值，可得x=e时y最大，但只能分解成整数，故x取2或3，由于6=2+2+2=3+3，显然2^3=8<9=3^2,故应分解为多个3
```
class Solution {
public:
    int integerBreak(int n) {
        int max_ = n-1;
        int times = 0, carry = 0, tmp = 1;
        for(int i = 2; i<=n/2;i++)
        {
            times = n/i;
            carry = n%i;
            if(carry)   
                for(int j = 0;j<i-carry;j++)
                {
                    tmp*=times;
                }
            else
                for(int j = 0;j<i;j++)
                {
                    tmp*=times;
                }

            for(int j = 0;j<carry;j++)
                tmp*=(times+1);

            if(tmp > max_)
                max_ = tmp;
            tmp = 1;
        }
        return max_;
    }
};
```
## 第二十七题
### 题目描述
1452. 收藏清单
难度
中等

给你一个数组 favoriteCompanies ，其中 favoriteCompanies[i] 是第 i 名用户收藏的公司清单（下标从 0 开始）。

请找出不是其他任何人收藏的公司清单的子集的收藏清单，并返回该清单下标。下标需要按升序排列。

 

示例 1：

输入：favoriteCompanies = [["leetcode","google","facebook"],["google","microsoft"],["google","facebook"],["google"],["amazon"]]
输出：[0,1,4] 
解释：
favoriteCompanies[2]=["google","facebook"] 是 favoriteCompanies[0]=["leetcode","google","facebook"] 的子集。
favoriteCompanies[3]=["google"] 是 favoriteCompanies[0]=["leetcode","google","facebook"] 和 favoriteCompanies[1]=["google","microsoft"] 的子集。
其余的收藏清单均不是其他任何人收藏的公司清单的子集，因此，答案为 [0,1,4] 。
示例 2：

输入：favoriteCompanies = [["leetcode","google","facebook"],["leetcode","amazon"],["facebook","google"]]
输出：[0,1] 
解释：favoriteCompanies[2]=["facebook","google"] 是 favoriteCompanies[0]=["leetcode","google","facebook"] 的子集，因此，答案为 [0,1] 。
示例 3：

输入：favoriteCompanies = [["leetcode"],["google"],["facebook"],["amazon"]]
输出：[0,1,2,3]
 

提示：

1 <= favoriteCompanies.length <= 100
1 <= favoriteCompanies[i].length <= 500
1 <= favoriteCompanies[i][j].length <= 20
favoriteCompanies[i] 中的所有字符串 各不相同 。
用户收藏的公司清单也 各不相同 ，也就是说，即便我们按字母顺序排序每个清单， favoriteCompanies[i] != favoriteCompanies[j] 仍然成立。
所有字符串仅包含小写英文字母。
### 解答
直接暴力的拿set做了，效果不是很好只是过了而已，看了眼评论，方法都差不多，大同小异，区别就是他们用了更好的判断是否为子集的函数，以及更好的哈希表。今天太累了，先不学新的了，脑子疼。。
```
class Solution {
public:
    vector<int> peopleIndexes(vector<vector<string>>& favoriteCompanies) {
        vector< set<string> > record(favoriteCompanies.size());
        for(int i =0;i<favoriteCompanies.size();i++)
        {
            for(int j = 0; j< favoriteCompanies[i].size();j++)
            {
                record[i].insert(favoriteCompanies[i][j]);
            }
        }
        bool global_flag = false,local_flag = false;
        int count = 0;
        vector<int> result;
        for(int i =0;i<favoriteCompanies.size();i++)
        {
            count=0;
            for(int k = 0; k<record.size(); k++)//check
            {
                local_flag = false;
                if(k!=i)
                    for(int j = 0; j< favoriteCompanies[i].size();j++)
                    {
                        if(record[k].count(favoriteCompanies[i][j]) == 0 )//dont have
                        {
                            local_flag = true;
                            break;
                        }
                    }
                if(local_flag)
                {
                    count++;
                }
            }
            if(count == favoriteCompanies.size() - 1)
            {
                result.push_back(i);
                count = 0;
            }
        }
        return result;
    }
};
```
## 第28题
### 题目描述
1128. 等价多米诺骨牌对的数量
难度
简单


给你一个由一些多米诺骨牌组成的列表 dominoes。

如果其中某一张多米诺骨牌可以通过旋转 0 度或 180 度得到另一张多米诺骨牌，我们就认为这两张牌是等价的。

形式上，dominoes[i] = [a, b] 和 dominoes[j] = [c, d] 等价的前提是 a==c 且 b==d，或是 a==d 且 b==c。

在 0 <= i < j < dominoes.length 的前提下，找出满足 dominoes[i] 和 dominoes[j] 等价的骨牌对 (i, j) 的数量。

 

示例：

输入：dominoes = [[1,2],[2,1],[3,4],[5,6]]
输出：1
 

提示：

1 <= dominoes.length <= 40000
1 <= dominoes[i][j] <= 9
### 解答
map直接做
```
class Solution {
public:
    int numEquivDominoPairs(vector<vector<int>>& dominoes) {
        map<vector<int>,int> m;
        int m1 = 0, n = 0;
        int sum = 0;
        for(int i = 0; i< dominoes.size() ; i++)
        {
            if(m.count(dominoes[i]))//has
                m[dominoes[i]] ++;
            else
                m[dominoes[i]] = 1;
        }
        for (auto i = m.begin(); i != m.end(); ++i)
        {
            m1 = i->first[0];
            n = i->first[1];
            
            if(m.count({n,m1})&&n!=m1)//has
            {
                sum+=(m[{n,m1}] * m[{m1,n}]);
                cout<<n<<' '<<m1<<endl;
            }
            if(i->second > 1)
                sum +=(i->second * (i->second - 1))/2;
            i->second = 0;
        }
        cout<<m[{2,1}]<<endl;
        return sum;
    }
};
```
## 第29题
### 题目描述
14. 最长公共前缀
难度
简单

编写一个函数来查找字符串数组中的最长公共前缀。

如果不存在公共前缀，返回空字符串 ""。

 

示例 1：

输入：strs = ["flower","flow","flight"]
输出："fl"
示例 2：

输入：strs = ["dog","racecar","car"]
输出：""
解释：输入不存在公共前缀。
 

提示：

0 <= strs.length <= 200
0 <= strs[i].length <= 200
strs[i] 仅由小写英文字母组成
### 解答
```
class Solution {
public:
    string longestCommonPrefix(vector<string>& strs) {
        string result = "";
        if(strs.size() == 0)
            return "";
        int min_len = 201;
        for(int i = 0; i<strs.size(); i++)
        {
            if(strs[i].size() < min_len)
                min_len = strs[i].size();
        }
        if(min_len == 0)
            return "";
        char tmp; 
        for(int j = 0; j<min_len;j++)
        {
            tmp = strs[0][j];
            for(int i = 0; i <strs.size();i++)
            {
                if(strs[i][j]!=tmp)
                    return result;
            }
            result.push_back(tmp);
        }
        return result;
        
    }
};
```
## 第30题
### 题目描述
32. 最长有效括号
难度
困难

给你一个只包含 '(' 和 ')' 的字符串，找出最长有效（格式正确且连续）括号子串的长度。

 

示例 1：

输入：s = "(()"
输出：2
解释：最长有效括号子串是 "()"
示例 2：

输入：s = ")()())"
输出：4
解释：最长有效括号子串是 "()()"
示例 3：

输入：s = ""
输出：0
 

提示：

0 <= s.length <= 3 * 104
s[i] 为 '(' 或 ')'
### 解答
我觉得这道题挺简单的，没有困难的难度
```
class Solution {
public:
    int longestValidParentheses(string s) {
        int n = s.size();
        //vector< vector<int> > dp(n,vector<int>(n,0));
        vector<int> dp(n,0);
        int cur_index;
        int result = 0;
        for(int i = 0 ; i < n ; i++ )
        {
            if(s[i]=='('&& i!=n-1)//forward
            {
                if(i + 1 < n&&s[i  + 1] == ')')
                {
                    //need to calc consecutive pair
                    //backward find
                    dp[i] = 2;
                    dp[i +1] =2;
                    if(i>0)
                    {
                        dp[i] += dp[i-1];
                        dp[i+1] = dp[i];
                    }
                    
                }
            }
            if(s[i] == ')'&& i>=1)//backward
            {
                if(s[i-1] == ')')//find consecutive pair
                {
                    cur_index = i-1;
                    while(cur_index>0 && s[cur_index] == ')')
                    {
                        if(dp[cur_index] == 0)
                            break;
                        cur_index -= dp[cur_index];
                    }
                        
                    if(cur_index>=0 && s[cur_index] == '(')//match
                    {
                        dp[i] = (i - cur_index + 1);
                        dp[cur_index] = dp[i];
                        if(cur_index >=1)
                        {
                            dp[i] += dp[cur_index - 1];
                            dp[cur_index] = dp[i];
                        }
                    }
                }
            }
            if(dp[i] > result)
                result = dp[i];
        }
 
            
        return result;
    }
};
```
## 第31题
### 题目描述
42. 接雨水
难度
困难

给定 n 个非负整数表示每个宽度为 1 的柱子的高度图，计算按此排列的柱子，下雨之后能接多少雨水。

 

示例 1：



输入：height = [0,1,0,2,1,0,1,3,2,1,2,1]
输出：6
解释：上面是由数组 [0,1,0,2,1,0,1,3,2,1,2,1] 表示的高度图，在这种情况下，可以接 6 个单位的雨水（蓝色部分表示雨水）。 
示例 2：

输入：height = [4,2,0,3,2,5]
输出：9
 

提示：

n == height.length
0 <= n <= 3 * 104
0 <= height[i] <= 105
### 解答
没用动态规划做，有点慢，不过复杂度都是n
我的版本
```
class Solution {
public:
    int trap(vector<int>& height) {
        if(height.size() <= 2)
            return 0;

        int cur_min = height[0];
        int start = 0,end = 0;
        int sum = 0,cur_sum = 0;
        for(int i = 1; i < height.size() ; i++)//forward
        {
            if(height[i]>=cur_min)
            {
                sum += (i-start-1)*cur_min;
                sum -= cur_sum;
                cur_sum = 0;
                start = i;
                cur_min = height[i];
            }
            else if(height[i] < cur_min)
            {
                cur_sum +=height[i];
            }
        }
        end = start;
        cur_min = height[height.size() - 1];
        start = height.size() -1;
        cur_sum = 0;
        for(int i = height.size() - 2; i >= end; i--)//backward
        {
            if(height[i] >=cur_min)
            {
                sum += abs(start - i -1) * cur_min;
                sum -= cur_sum;
                start = i;
                cur_min = height[i];
                cur_sum = 0;
                //cout<<cur_min<<' '<<start<<' '<<cur_sum<<' '<<sum<<endl;
            }
            else
            {
                cur_sum += height[i];
            }
        }
        return sum;
    }
};

```
评论区版本
```
class Solution {
public:
	int trap(vector<int>& height) {
		int n = height.size();
		// left[i]表示i左边的最大值，right[i]表示i右边的最大值
		vector<int> left(n), right(n);
		for (int i = 1; i < n; i++) {
			left[i] = max(left[i - 1], height[i - 1]);
		}
		for (int i = n - 2; i >= 0; i--) {
			right[i] = max(right[i + 1], height[i + 1]);
		}
		int water = 0;
		for (int i = 0; i < n; i++) {
			int level = min(left[i], right[i]);
			water += max(0, level - height[i]);
		}
		return water;
	}
};
```
## 第32题
### 题目描述
剑指 Offer 13. 机器人的运动范围
难度
中等

地上有一个m行n列的方格，从坐标 [0,0] 到坐标 [m-1,n-1] 。一个机器人从坐标 [0, 0] 的格子开始移动，它每次可以向左、右、上、下移动一格（不能移动到方格外），也不能进入行坐标和列坐标的数位之和大于k的格子。例如，当k为18时，机器人能够进入方格 [35, 37] ，因为3+5+3+7=18。但它不能进入方格 [35, 38]，因为3+5+3+8=19。请问该机器人能够到达多少个格子？

 

示例 1：

输入：m = 2, n = 3, k = 1
输出：3
示例 2：

输入：m = 3, n = 1, k = 0
输出：1
提示：

1 <= n,m <= 100
0 <= k <= 20
### 解答
bfs pair是这么用的
```
class Solution {
    int get(int x) {
        int res=0;
        for (; x; x /= 10) {
            res += x % 10;
        }
        return res;
    }
public:
    int movingCount(int m, int n, int k) {
        if (!k) return 1;
        queue<pair<int,int> > Q;
        int dx[2] = {0, 1};
        int dy[2] = {1, 0};
        vector<vector<int> > vis(m, vector<int>(n, 0));
        Q.push(make_pair(0, 0));
        vis[0][0] = 1;
        int ans = 1;
        while (!Q.empty()) {
            auto [x, y] = Q.front();
            Q.pop();
            for (int i = 0; i < 2; ++i) {
                int tx = dx[i] + x;
                int ty = dy[i] + y;
                if (tx < 0 || tx >= m || ty < 0 || ty >= n || vis[tx][ty] || get(tx) + get(ty) > k) continue;
                Q.push(make_pair(tx, ty));
                vis[tx][ty] = 1;
                ans++;
            }
        }
        return ans;
    }
};
```
## 第32题
### 题目描述
860. 柠檬水找零
难度
简单

在柠檬水摊上，每一杯柠檬水的售价为 5 美元。

顾客排队购买你的产品，（按账单 bills 支付的顺序）一次购买一杯。

每位顾客只买一杯柠檬水，然后向你付 5 美元、10 美元或 20 美元。你必须给每个顾客正确找零，也就是说净交易是每位顾客向你支付 5 美元。

注意，一开始你手头没有任何零钱。

如果你能给每位顾客正确找零，返回 true ，否则返回 false 。

示例 1：

输入：[5,5,5,10,20]
输出：true
解释：
前 3 位顾客那里，我们按顺序收取 3 张 5 美元的钞票。
第 4 位顾客那里，我们收取一张 10 美元的钞票，并返还 5 美元。
第 5 位顾客那里，我们找还一张 10 美元的钞票和一张 5 美元的钞票。
由于所有客户都得到了正确的找零，所以我们输出 true。
示例 2：

输入：[5,5,10]
输出：true
示例 3：

输入：[10,10]
输出：false
示例 4：

输入：[5,5,10,10,20]
输出：false
解释：
前 2 位顾客那里，我们按顺序收取 2 张 5 美元的钞票。
对于接下来的 2 位顾客，我们收取一张 10 美元的钞票，然后返还 5 美元。
对于最后一位顾客，我们无法退回 15 美元，因为我们现在只有两张 10 美元的钞票。
由于不是每位顾客都得到了正确的找零，所以答案是 false。
 

提示：

0 <= bills.length <= 10000
bills[i] 不是 5 就是 10 或是 20 
### 解答
注意不需要排序
```
class Solution {
public:
    bool lemonadeChange(vector<int>& bills) {
        int count5 = 0;
        int count10 = 0;
        //sort(bills.begin(),bills.end());
        for(int i = 0; i<bills.size(); i++)
        {
            if(bills[i] == 5)
                count5++;
            else if(bills[i] == 10)
            {
                if(count5 == 0)
                    return false;
                else
                {
                    count5--;
                    count10++;
                }
            }
            else
            {
                if(count10 == 0)
                {
                    if(count5<3)
                        return false;
                    else
                    {
                        count5-=3;
                    }
                }
                else
                {
                    count10--;
                    if(count5)
                        count5--;
                    else
                        return false;
                }
            }
        }
        return true;
    }
};
```
## 第33题
### 题目描述
207. 课程表
难度
中等

701

收藏

分享
切换为英文
接收动态
反馈
你这个学期必须选修 numCourse 门课程，记为 0 到 numCourse-1 。

在选修某些课程之前需要一些先修课程。 例如，想要学习课程 0 ，你需要先完成课程 1 ，我们用一个匹配来表示他们：[0,1]

给定课程总量以及它们的先决条件，请你判断是否可能完成所有课程的学习？

 

示例 1:

输入: 2, [[1,0]] 
输出: true
解释: 总共有 2 门课程。学习课程 1 之前，你需要完成课程 0。所以这是可能的。
示例 2:

输入: 2, [[1,0],[0,1]]
输出: false
解释: 总共有 2 门课程。学习课程 1 之前，你需要先完成​课程 0；并且学习课程 0 之前，你还应先完成课程 1。这是不可能的。
 

提示：

输入的先决条件是由 边缘列表 表示的图形，而不是 邻接矩阵 。详情请参见图的表示法。
你可以假定输入的先决条件中没有重复的边。
1 <= numCourses <= 10^5
### 解答
拓补排序，实现算法思想为记录入度，然后维护一个栈或者队列，列入入度为0的点，然后遍历表，遇到一个后记节点就将他的入度-1，如果减到零，入栈，直到空战
```
class Solution {
public:
    bool canFinish(int numCourses, vector<vector<int>>& prerequisites) {
        vector<int> count_in_dimension(numCourses,0);
        for(int i = 0; i<prerequisites.size() ; i ++)
            count_in_dimension[prerequisites[i][1]] ++;
        stack<int> s;
        int result = 0, cur = 0;
        for(int i = 0 ; i<numCourses ;i++)
            if(count_in_dimension[i] == 0)
                s.push(i);
        while(!s.empty())
        {
            cur = s.top();
            s.pop();
            result++;
            for(int i = 0; i < prerequisites.size(); i++)
            {
                if(prerequisites[i][0] == cur)
                {
                    if(--count_in_dimension[prerequisites[i][1]] == 0)
                        s.push(prerequisites[i][1]);
                }
            }
        }
        return result==numCourses;
    }
};
```
## 第34题
### 题目描述
524. 通过删除字母匹配到字典里最长单词
难度
中等

给定一个字符串和一个字符串字典，找到字典里面最长的字符串，该字符串可以通过删除给定字符串的某些字符来得到。如果答案不止一个，返回长度最长且字典顺序最小的字符串。如果答案不存在，则返回空字符串。

示例 1:

输入:
s = "abpcplea", d = ["ale","apple","monkey","plea"]

输出: 
"apple"
示例 2:

输入:
s = "abpcplea", d = ["a","b","c"]

输出: 
"a"
说明:

所有输入的字符串只包含小写字母。
字典的大小不会超过 1000。
所有输入的字符串长度不会超过 1000。
### 解答
双指针，按照正常想法做就好，学习到的一点就是string内置了比较运算符，直接比较就是字典序
```
class Solution {
public:
    string findLongestWord(string s, vector<string>& d) {
        int max_len = 0;
        //int d_index = 0;
        int s_index = 0,di_index=0;
        string result = "";
        for(int i = 0 ; i<d.size(); i++)
        {
            if(d[i].size() < max_len)
                continue;
            else
            {
                //double pointer
                s_index = 0;
                di_index = 0;
                while(s_index < s.size()&& di_index<d[i].size())
                {
                    if(s[s_index] == d[i][di_index])
                    {
                        s_index++;
                        di_index++;
                    }
                    else
                    {
                        s_index++;
                    }
                }
                if(di_index == d[i].size())//match
                {
                    if(di_index>max_len)
                    {
                        max_len = di_index;
                        result = d[i];
                    }
                    else if(di_index == max_len)
                    {
                        //compare
                        result = (result > d[i])?d[i]:result;
                    }
                }
            }
        }
        return result;
    }
};
```
## 第35题
### 题目描述
1573. 分割字符串的方案数
难度
中等

给你一个二进制串 s  （一个只包含 0 和 1 的字符串），我们可以将 s 分割成 3 个 非空 字符串 s1, s2, s3 （s1 + s2 + s3 = s）。

请你返回分割 s 的方案数，满足 s1，s2 和 s3 中字符 '1' 的数目相同。

由于答案可能很大，请将它对 10^9 + 7 取余后返回。

 

示例 1：

输入：s = "10101"
输出：4
解释：总共有 4 种方法将 s 分割成含有 '1' 数目相同的三个子字符串。
"1|010|1"
"1|01|01"
"10|10|1"
"10|1|01"
示例 2：

输入：s = "1001"
输出：0
示例 3：

输入：s = "0000"
输出：3
解释：总共有 3 种分割 s 的方法。
"0|0|00"
"0|00|0"
"00|0|0"
示例 4：

输入：s = "100100010100110"
输出：12
 

提示：

s[i] == '0' 或者 s[i] == '1'
3 <= s.length <= 10^5
### 解答
问题倒是不大，主要是结果需要取余数
```
class Solution {
public:
    int numWays(string s) {
        int sum = 0;
        int cur0 = 0;
        int cur1 = 0;
        long long unsigned int result = 1;
        for(int i = 0; i< s.size(); i++)
            if(s[i]=='1')
                sum++;
        if(sum%3)
            return 0;
        if(sum == 0)
            return ((s.size()-1)*(s.size()-2)/2)%1000000007;;
        sum /= 3;
        //cout<<sum<<endl;
        int i;
        for( i =0 ;cur1<sum; i++)
        {
            if(s[i] == '1')
            {
                cur1++;
            }
        }
        
        for(;s[i]=='0';i++)
            cur0++;
        result *=(cur0+1);
        result %=1000000007;
        //cout<<cur0<<' '<<cur1<<' '<<i<<endl;
        cur1 = 0;
        cur0 = 0;
        for(;cur1<sum;i++)
            if(s[i]=='1')
                cur1++;
        for(;s[i]!='1';i++)
            cur0++;
        result *=(cur0+1);
        result %=1000000007;
        return result;

    }
};

```
##第36题
### 题目描述
1048. 最长字符串链
难度
中等

给出一个单词列表，其中每个单词都由小写英文字母组成。

如果我们可以在 word1 的任何地方添加一个字母使其变成 word2，那么我们认为 word1 是 word2 的前身。例如，"abc" 是 "abac" 的前身。

词链是单词 [word_1, word_2, ..., word_k] 组成的序列，k >= 1，其中 word_1 是 word_2 的前身，word_2 是 word_3 的前身，依此类推。

从给定单词列表 words 中选择单词组成词链，返回词链的最长可能长度。
 

示例：

输入：["a","b","ba","bca","bda","bdca"]
输出：4
解释：最长单词链之一为 "a","ba","bda","bdca"。

### 解答
这里要注意一下，自定义的cmp函数需要static，动态规划的一道比较简单的题目
```
class Solution {
public:
    static bool cmp(string s1,string s2)
    {
        return s1.size()<s2.size();
    }
    int longestStrChain(vector<string>& words) {
        sort(words.begin(),words.end(),cmp);
        
        int length = words.size();
        vector<int> dp(length,1);
        //for(int i = 0;i<length;i++)
          //  cout<<words[i].size()<<' ';
        //cout<<endl;
        int result = 1;
        for(int i = 0; i<length;i++)
        {
            for(int j = i+1;j<length&& words[j].size() <= words[i].size()+1 ;j++)
            {
                if(words[i].size() == words[j].size())
                    continue;
                else if(match(words[i],words[j]))
                {
                    dp[j] = max(dp[j],dp[i]+1);
                    if(dp[j]>result)
                        result = dp[j];
                }
            }
        }
       // for(int i = 0;i<length;i++)
         //   cout<<dp[i]<<' ';
        return result;
    }
    bool match(string s1, string s2)
    {
        int i = 0,j = 0;
        int count = 1;
        while(i<s1.size()&& j<s2.size())
        {
            if(s1[i]==s2[j])
            {
                i++;
                j++;
            }
            else
            {
                if(count)
                {
                    count--;
                    j++;
                }
                else
                    return false;
            }
        }
        return true;
    }
};
```
## 第38题
### 题目描述
882. 细分图中的可到达结点
难度
困难

从具有 0 到 N-1 的结点的无向图（“原始图”）开始，对一些边进行细分。

该图给出如下：edges[k] 是整数对 (i, j, n) 组成的列表，使 (i, j) 是原始图的边。

n 是该边上新结点的总数

然后，将边 (i, j) 从原始图中删除，将 n 个新结点 (x_1, x_2, ..., x_n) 添加到原始图中，

将 n+1 条新边 (i, x_1), (x_1, x_2), (x_2, x_3), ..., (x_{n-1}, x_n), (x_n, j) 添加到原始图中。

现在，你将从原始图中的结点 0 处出发，并且每次移动，你都将沿着一条边行进。

返回最多 M 次移动可以达到的结点数。

 

示例 1：

输入：edges = [[0,1,10],[0,2,1],[1,2,2]], M = 6, N = 3
输出：13
解释：
在 M = 6 次移动之后在最终图中可到达的结点如下所示。

示例 2：

输入：edges = [[0,1,4],[1,2,6],[0,2,8],[1,3,1]], M = 10, N = 4
输出：23
 

提示：

0 <= edges.length <= 10000
0 <= edges[i][0] < edges[i][1] < N
不存在任何 i != j 情况下 edges[i][0] == edges[j][0] 且 edges[i][1] == edges[j][1].
原始图没有平行的边。
0 <= edges[i][2] <= 10000
0 <= M <= 10^9
1 <= N <= 3000
可到达结点是可以从结点 0 开始使用最多 M 次移动到达的结点。

### 解答
我卡了好久，两三天这样，然后终于把bug弄没了，但是超时。
我的版本，正常思维，广度优先搜索，超时
```
class Solution {
public:
    int reachableNodes(vector<vector<int>>& edges, int M, int N) {
        vector<int> visit(N,0);
        vector<int> record(edges.size());
        for(int i = 0; i<edges.size();i++)
            record[i] = edges[i][2];
        vector<vector<int>> consumed(N,vector<int>(edges.size(),0));//record every nodes consume
        //i node, j edge
        queue<pair<int,int>> q;
        q.push(make_pair(0,M));
        int result = 0,tmp = 0;
        int cur_node = 0, cur_step = 0,next_node = 0;
        while(!q.empty())
        {
            cur_node = q.front().first;
            cur_step = q.front().second;
            if(!visit[cur_node])
            {
                result++;
                visit[cur_node] = 1;
            }
            
            q.pop();
            if(cur_step == 0)
                continue;
            for(int i = 0;i<edges.size() ; i++)
            {
                //if(visit[edges[i][0]]&&visit[edges[i][1]]&& edges[i][2] == 0)
                    //continue;
                if(edges[i][0] == cur_node)
                    next_node = edges[i][1];
                else if(edges[i][1] == cur_node)
                    next_node = edges[i][0];
                else
                    continue;
                if(cur_step == record[i])
                {
                    result+=edges[i][2];
                    edges[i][2] =0;
                    consumed[cur_node][i] = record[i];
                }
                else if(cur_step < record[i])//bug
                {
                    if(cur_step > consumed[cur_node][i]&& edges[i][2] > 0)
                    {
                        tmp = (cur_step - consumed[cur_node][i]);
                        if(tmp>=edges[i][2])
                        {
                            result += edges[i][2];
                            edges[i][2] = 0;
                        }
                        else
                        {
                            result += tmp;
                            edges[i][2] -= tmp;
                        }
                        consumed[cur_node][i] = cur_step;
                    }
                }
                else if(cur_step > record[i])
                {
                    result += edges[i][2];
                    q.push(make_pair(next_node,cur_step-record[i]-1));
                    edges[i][2] = 0;
                    consumed[cur_node][i] = record[i];
                }
            }
        }
        return result;
    }
};
```
无奈之下看了下解题，用的是优先级队列的dijistra
```
class Solution {
public:
    int reachableNodes(vector<vector<int>>& edges, int M, int N) {
        vector<vector<pii>> graph(N);
        for (vector<int> edge: edges) {
            int u = edge[0], v = edge[1], w = edge[2];
            graph[u].push_back({v, w});
            graph[v].push_back({u, w});
        }
        //store graph
        map<int, int> dist;
        dist[0] = 0;
        for (int i = 1; i < N; ++i)
            dist[i] = M+1;

        map<pii, int> used;
        int ans = 0;

        priority_queue<pii, vector<pii>, greater<pii>> pq;
        pq.push({0, 0});

        while (!pq.empty()) {
            pii top = pq.top();
            pq.pop();
            int d = top.first, node = top.second;
            if (d > dist[node]) continue;
            dist[node] = d;

            // Each node is only visited once.  We've reached
            // a node in our original graph.
            ans++;

            for (auto pair: graph[node]) {
                // M - d is how much further we can walk from this node;
                // weight is how many new nodes there are on this edge.
                // v is the maximum utilization of this edge.
                int nei = pair.first;
                int weight = pair.second;
                used[{node, nei}] = min(weight, M - d);

                // d2 is the total distance to reach 'nei' (nei***or) node
                // in the original graph.
                int d2 = d + weight + 1;
                if (d2 < min(dist[nei], M+1)) {
                    pq.push({d2, nei});
                    dist[nei] = d2;
                }
            }
        }

        // At the end, each edge (u, v, w) can be used with a maximum
        // of w new nodes: a max of used[u, v] nodes from one side,
        // and used[v, u] nodes from the other.
        for (vector<int> edge: edges) {
            int u = edge[0], v = edge[1], w = edge[2];
            ans += min(w, used[{u, v}] + used[{v, u}]);
        }
        return ans;
    }
};
```
## 第39题
### 题目描述
简单
给定一个非负整数 num，反复将各个位上的数字相加，直到结果为一位数。

示例:

输入: 38
输出: 2 
解释: 各位相加的过程为：3 + 8 = 11, 1 + 1 = 2。 由于 2 是一位数，所以返回 2。
### 解答
```
class Solution {
public:
    int addDigits(int num) {
        int tmp = 0;
        while(num>=10)
        {
            tmp = 0;
            while(num>0)
            {
                tmp += num%10;
                num/=10;
            }
            num = tmp;
        }
        return num;
    }
};
```
## 第40题
### 题目描述
1625. 执行操作后字典序最小的字符串
难度
中等

给你一个字符串 s 以及两个整数 a 和 b 。其中，字符串 s 的长度为偶数，且仅由数字 0 到 9 组成。

你可以在 s 上按任意顺序多次执行下面两个操作之一：

累加：将  a 加到 s 中所有下标为奇数的元素上（下标从 0 开始）。数字一旦超过 9 就会变成 0，如此循环往复。例如，s = "3456" 且 a = 5，则执行此操作后 s 变成 "3951"。
轮转：将 s 向右轮转 b 位。例如，s = "3456" 且 b = 1，则执行此操作后 s 变成 "6345"。
请你返回在 s 上执行上述操作任意次后可以得到的 字典序最小 的字符串。

如果两个字符串长度相同，那么字符串 a 字典序比字符串 b 小可以这样定义：在 a 和 b 出现不同的第一个位置上，字符串 a 中的字符出现在字母表中的时间早于 b 中的对应字符。例如，"0158” 字典序比 "0190" 小，因为不同的第一个位置是在第三个字符，显然 '5' 出现在 '9' 之前。

 

示例 1：

输入：s = "5525", a = 9, b = 2
输出："2050"
解释：执行操作如下：
初态："5525"
轮转："2555"
累加："2454"
累加："2353"
轮转："5323"
累加："5222"
累加："5121"
轮转："2151"
累加："2050"​​​​​​​​​​​​
无法获得字典序小于 "2050" 的字符串。
示例 2：

输入：s = "74", a = 5, b = 1
输出："24"
解释：执行操作如下：
初态："74"
轮转："47"
累加："42"
轮转："24"​​​​​​​​​​​​
无法获得字典序小于 "24" 的字符串。
示例 3：

输入：s = "0011", a = 4, b = 2
输出："0011"
解释：无法获得字典序小于 "0011" 的字符串。
示例 4：

输入：s = "43987654", a = 7, b = 3
输出："00553311"
### 解答
bfs遍历所有情况，这里我思路错了，我老想着不用遍历所有情况，只用递减就行，其实还是需要遍历所有情况找最小的情况
```
class Solution {
public:
    string findLexSmallestString(string s, int a, int b) {
        queue<string> q;
        set<string> visit;
        string tmp = "";
        string off_tmp = "";
        string add_tmp = "";
        string result = s;
        q.push(s);
        while(!q.empty())
        {   tmp = q.front();
            q.pop();
            off_tmp = offset(tmp,b);
            add_tmp = add(tmp,a);
            if(!visit.count(add_tmp))
            {
                q.push(add_tmp);
                visit.emplace(add_tmp);
            }
            if(!visit.count(off_tmp))
            {
                q.push(off_tmp);
                visit.emplace(off_tmp);
            }
            result = min(result,min(off_tmp,add_tmp));
        }
        return result;
    
    }
    string offset(string s, int num)
    {
        string result = s.substr(s.size() - num,num);
        return result + s.substr(0,s.size()-num);
    }
    string add(string s, int num)
    {
        int tmp = 0;
        for(int i = 0; i<s.size(); i++)
        {
            char tmp_char = s[i];
            if(i%2)
            {
                tmp = tmp_char - '0' + num;
                tmp%=10;
                s[i] = '0' + tmp;
                
            }
        }
        return s;
    }
};
```
## 第41题
### 题目描述
1143. 最长公共子序列
难度
中等

给定两个字符串 text1 和 text2，返回这两个字符串的最长公共子序列的长度。

一个字符串的 子序列 是指这样一个新的字符串：它是由原字符串在不改变字符的相对顺序的情况下删除某些字符（也可以不删除任何字符）后组成的新字符串。
例如，"ace" 是 "abcde" 的子序列，但 "aec" 不是 "abcde" 的子序列。两个字符串的「公共子序列」是这两个字符串所共同拥有的子序列。

若这两个字符串没有公共子序列，则返回 0。

 

示例 1:

输入：text1 = "abcde", text2 = "ace" 
输出：3  
解释：最长公共子序列是 "ace"，它的长度为 3。
示例 2:

输入：text1 = "abc", text2 = "abc"
输出：3
解释：最长公共子序列是 "abc"，它的长度为 3。
示例 3:

输入：text1 = "abc", text2 = "def"
输出：0
解释：两个字符串没有公共子序列，返回 0。
### 解答
dp
```
class Solution {
public:
    int longestCommonSubsequence(string text1, string text2) {
        // seems like dp
        vector<vector<int>> dp(text1.size(),vector<int>(text2.size(),0));
        //vector<int> record1(text1.size(),0);
        //vector<int> record2(text2.size(),0);
        for(int i = 0 ; i<text1.size(); i++)
        {
            for(int j = 0; j<text2.size(); j++)
            {
                if(text1[i] == text2[j] )
                {
                    if(i == 0 ||j == 0)
                    {
                        dp[i][j] = 1;
                    }
                    //
                    else
                    {
                        dp[i][j] = dp[i-1][j-1] + 1;
                    }
                }
                else
                {
                    if(i == 0 && j == 0)
                    {
                        dp[i][j] = 0;
                    }
                    else if(i == 0 && j!=0)
                    {
                        dp[i][j] = dp[i][j-1];
                    }
                    else if(i!=0 && j == 0)
                    {
                        dp[i][j] = dp[i-1][j];
                    }
                    else
                    {
                        dp[i][j] = max(dp[i-1][j],dp[i][j-1]);
                    }
                }
            }
        }
            
        return dp[text1.size() - 1][text2.size() - 1];
    }
};
```
## 第42题
### 题目描述
1217. 玩筹码
难度
简单
数轴上放置了一些筹码，每个筹码的位置存在数组 chips 当中。

你可以对 任何筹码 执行下面两种操作之一（不限操作次数，0 次也可以）：

将第 i 个筹码向左或者右移动 2 个单位，代价为 0。
将第 i 个筹码向左或者右移动 1 个单位，代价为 1。
最开始的时候，同一位置上也可能放着两个或者更多的筹码。

返回将所有筹码移动到同一位置（任意位置）上所需要的最小代价。

 

示例 1：

输入：chips = [1,2,3]
输出：1
解释：第二个筹码移动到位置三的代价是 1，第一个筹码移动到位置三的代价是 0，总代价为 1。
示例 2：

输入：chips = [2,2,2,3,3]
输出：2
解释：第四和第五个筹码移动到位置二的代价都是 1，所以最小总代价为 2。
### 解答
只需记录偶数和奇数数目，返回最小者
```
class Solution {
public:
    int minCostToMoveChips(vector<int>& position) {
        int odd_count = 0, even_count = 0;
        for(int i = 0; i< position.size(); i++)
        {
            if(position[i]%2)//odd
                odd_count++;
            else
                even_count++;
        } 
        return min(even_count,odd_count);
    }
};
```
## 第43题
### 题目描述
1109. 航班预订统计
难度
中等

这里有 n 个航班，它们分别从 1 到 n 进行编号。

我们这儿有一份航班预订表，表中第 i 条预订记录 bookings[i] = [j, k, l] 意味着我们在从 j 到 k 的每个航班上预订了 l 个座位。

请你返回一个长度为 n 的数组 answer，按航班编号顺序返回每个航班上预订的座位数。

 

示例：

输入：bookings = [[1,2,10],[2,3,20],[2,5,25]], n = 5
输出：[10,55,45,25,25]
 

提示：

1 <= bookings.length <= 20000
1 <= bookings[i][0] <= bookings[i][1] <= n <= 20000
1 <= bookings[i][2] <= 10000

### 解答
暴力版本会超时，但是我没想到除了暴力有别的方法，看了评论区才知道可以做到n的复杂度，数学思想是查分数组。
暴力版
```
class Solution {
public:
    vector<int> corpFlightBookings(vector<vector<int>>& bookings, int n) {
        vector<int> result(n,0);
        int start,end,add;
        for(int i = 0; i < bookings.size(); i++)
        {
            start   = bookings[i][0] - 1;
            end     = bookings[i][1] - 1;
            add = bookings[i][2];
            for(int j = start; j<=end;j++)
                result[j] += add;
        }
        return result;
    }
};
```
优化版
```
class Solution {
public:
    vector<int> corpFlightBookings(vector<vector<int>>& bookings, int n) {
        vector<int> result(n,0);
        for(int i = 0; i < bookings.size(); i++)
        {
            result[bookings[i][0] - 1] += bookings[i][2];
            if(bookings[i][1] < n)
                result[bookings[i][1] ] -= bookings[i][2];
        }
        for(int i = 1; i<n; i++)
            result[i] += result[i-1];
        return result;
    }
};
```
## 第44题
### 题目描述
204. 计数质数
难度
简单

统计所有小于非负整数 n 的质数的数量。

 

示例 1：

输入：n = 10
输出：4
解释：小于 10 的质数一共有 4 个, 它们是 2, 3, 5, 7 。
示例 2：

输入：n = 0
输出：0
示例 3：

输入：n = 1
输出：0
 

提示：

0 <= n <= 5 * 106

### 解答
虽然是简单题目，但是这个判断统计质数的方法还是很好的，暴力法过不了
这个筛选法就是从2开始，如果遇到质数，就把所有小于n的这个质数的倍数全部筛出，以此类推
比正常一个一个判断质数然后计数的方法快很多很多倍
```
class Solution {
public:
    int countPrimes(int n) {
    if(n<=2)
        return 0;
    vector<int> isPrime(n,1);
    isPrime[0] = 0;
    isPrime[1] = 0;

    for(int i = 2;i*i < n;i++)
    {
        if(isPrime[i])// sieve out every i*m
        {
            for(int j = i;j*i<n;j++)
            {
                isPrime[i*j] = 0;
            }
        }
    }
    int result = 0;
    for(int i =2;i<n;i++)
        if(isPrime[i])
            result++;
    return result;
    }
};
```
## 第45题
### 题目描述
965. 单值二叉树
难度
简单

如果二叉树每个节点都具有相同的值，那么该二叉树就是单值二叉树。

只有给定的树是单值二叉树时，才返回 true；否则返回 false。

### 解答
```
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    bool isUnivalTree(TreeNode* root) {
        stack<TreeNode *> s;
        if(root == NULL)
            return true;
        s.push(root);
        int tmp = root->val;
        TreeNode * cur = NULL;
        while(!s.empty())
        {
            cur = s.top();
            s.pop();
            if(cur->val != tmp)
                return false;
            if(cur->right)
                s.push(cur->right);
            if(cur->left)
                s.push(cur->left);
        }
        return true;
    }
};
```
## 第46题
### 题目描述
面试题 17.05.  字母与数字
难度
中等

给定一个放有字符和数字的数组，找到最长的子数组，且包含的字符和数字的个数相同。

返回该子数组，若存在多个最长子数组，返回左端点最小的。若不存在这样的数组，返回一个空数组。

示例 1:

输入: ["A","1","B","C","D","2","3","4","E","5","F","G","6","7","H","I","J","K","L","M"]

输出: ["A","1","B","C","D","2","3","4","E","5","F","G","6","7"]
示例 2:

输入: ["A","A"]

输出: []
提示：

array.length <= 100000
### 解答
一开始我是懵的，后来看题目里的提示才知道可以统计字母和数字的差值来进行计算
假设在这个表中，索引 i 满足 count(A, 0->i) = 3 和 count(B, 0->i) = 7。这意味着 B 比 A 多 4 个。如果你发现后面的某点 j 具有相同的差值(count(B, 0->j) - count(a, 0->j))，那么这表示子数组中有相同数量的 A 和 B。
```
class Solution {
public:
    vector<string> findLongestSubarray(vector<string>& array) {
        int a_count = 0,n_count = 0;
        map<int,int> record;
        char tmp;
        int max_len = 0;
        int val = 0,start = 0, end = 0;
        record[0] = -1;
        for(int i = 0; i <array.size(); i++)
        {
            tmp = array[i][0];
            if(tmp>='0' && tmp<='9')
                val++;
            else
                val--;

            if(record.count(val) == 0)//no same before
            {
                record[val] = i;
            }
            else// there is a same record before
            {
                // no need to add this to map, because we are backward finding
                if(i-record[val]> max_len)
                {
                    max_len = i-record[val];
                    start = record[val];
                    end = i;
                }
            }

        }
        //cout<<start<<' '<<end<<' '<<max_len <<' '<<endl;
        if(val == 0)
            return array;
        vector<string> result;
        if(max_len == 0)
            return result;;
        for(int i = start+1; i<=end; i++)
            result.push_back(array[i]);
        return result;
        
    }
};
```
## 第47题
### 题目描述
1493. 删掉一个元素以后全为 1 的最长子数组
难度
中等

给你一个二进制数组 nums ，你需要从中删掉一个元素。

请你在删掉元素的结果数组中，返回最长的且只包含 1 的非空子数组的长度。

如果不存在这样的子数组，请返回 0 。

 

提示 1：

输入：nums = [1,1,0,1]
输出：3
解释：删掉位置 2 的数后，[1,1,1] 包含 3 个 1 。
示例 2：

输入：nums = [0,1,1,1,0,1,1,0,1]
输出：5
解释：删掉位置 4 的数字后，[0,1,1,1,1,1,0,1] 的最长全 1 子数组为 [1,1,1,1,1] 。
示例 3：

输入：nums = [1,1,1]
输出：2
解释：你必须要删除一个元素。
示例 4：

输入：nums = [1,1,0,0,1,1,1,0,1]
输出：4
示例 5：

输入：nums = [0,0,0]
输出：0
 

提示：

1 <= nums.length <= 10^5
nums[i] 要么是 0 要么是 1 。
### 解答
```
class Solution {
public:
    int longestSubarray(vector<int>& nums) {
        vector<int> record;
        int cur_1 = 0;
        for(int i = 0;i<nums.size();i++)
        {
            if(nums[i] == 1)
                cur_1++;
            else
            {
                record.push_back(cur_1);
                cur_1 = 0;
                //cout<<cur_1<<' ';
            }
        }
        if(cur_1)
        {
            record.push_back(cur_1);
            //cout<<cur_1<<endl;
        }
        if(record.size() == 1)
            return nums.size()-1;
        int result = record[0];
        for(int i = 0; i< record.size()-1;i++) 
        {
            result = max(result,record[i] + record[i+1]);
        } 
        return result;
        
    }
};
```
## 第48题
### 题目描述
剑指 Offer 46. 把数字翻译成字符串
难度
中等

给定一个数字，我们按照如下规则把它翻译为字符串：0 翻译成 “a” ，1 翻译成 “b”，……，11 翻译成 “l”，……，25 翻译成 “z”。一个数字可能有多个翻译。请编程实现一个函数，用来计算一个数字有多少种不同的翻译方法。

 

示例 1:

输入: 12258
输出: 5
解释: 12258有5种不同的翻译，分别是"bccfi", "bwfi", "bczi", "mcfi"和"mzi"
 

提示：

0 <= num < 231

### 解答
```
class Solution {
public:
    int translateNum(int num) {
        if(num <10)
            return 1;
        if(num < 26)
            return 2;
        if(num%100>25||num%100<10)
        {
            return translateNum(num/10);
        }
        else
        {
            return translateNum(num/10) + translateNum(num/100);
        }
    }
};
```
## 第49题
### 题目描述
1745. 回文串分割 IV

给你一个字符串 s ，如果可以将它分割成三个 非空 回文子字符串，那么返回 true ，否则返回 false 。

当一个字符串正着读和反着读是一模一样的，就称其为 回文字符串 。

 

示例 1：

输入：s = "abcbdd"
输出：true
解释："abcbdd" = "a" + "bcb" + "dd"，三个子字符串都是回文的。
示例 2：

输入：s = "bcbddxy"
输出：false
解释：s 没办法被分割成 3 个回文子字符串。
 

提示：

3 <= s.length <= 2000
s​​​​​​ 只包含小写英文字母。
### 解答
这道题很难，但是我学习到了很多！
首先，统计一个字符串的最长的回文串长度的马拉车算法被我搞懂懂了！！主要参考下面这个视频
https://leetcode-cn.com/problems/longest-palindromic-substring/solution/zui-chang-hui-wen-zi-chuan-by-leetcode-solution/
然后最后的统计长度，我脑子有点抽，先是用了dfs遍历，然后发现超时，刚开始是以为栈开销太大，后来用的三重循环，还是超时，看了评论才知道二重循环就可以搞定。希望我复试的时候脑子不要抽！
```
class Solution {
public:
    bool checkPartitioning(string s) {
        string table = "#";
        for(int i = 0;i<s.size(); i++)
        {
            table.push_back(s[i]);
            table.push_back('#');
        }
        vector<int> dp(table.size(),0);
        int max_right = 0;
        int center = 0;
        int mirror = 0;
        int left = 0,right = 0;
        for(int i = 0;i<table.size();i++)
        {
            mirror = 2*center - i;
            if(i<max_right)// find dp[mirror]
            {
                dp[i] = min(max_right - i,dp[mirror]);
            }
            //try expand
            left = i-(1+dp[i]);
            right = i+1+dp[i];
            while(left>=0 && right<table.size() && table[left] == table[right])
            {
                dp[i]++;
                left--;
                right++;
            }
            if(i+dp[i] > max_right)
            {
                center = i;
                max_right = i+dp[i];
            }

        }
        //cout<<"sssss"<<endl;
        /*
        for(int i =0;i<table.size();i++)
            cout<<i<<' ';
        cout<< endl;
        for(int i =0;i<table.size();i++)
            cout<<table[i]<<' ';
        cout<<endl;
        for(int i =0;i<table.size();i++)
            cout<<dp[i]<<' ';
        cout<<endl;
        //*/
        int n = table.size();
        int first_end = 0, second_end = 0;
        for(int i = 1;i<n-2;i++)
        {
            if(dp[i] == i)
            {
                first_end = i+dp[i];
                for(int j = first_end + 1;j<n-1;j++)
                {
                    if(j-dp[j] <= first_end)
                    {
                        second_end = j+(j-first_end);
                        int mid = (second_end+n)/2;
                        
                        //cout<<mid<<endl;
                        if(mid<n-1&&mid + dp[mid] >=n-1)
                            return true;
                    }
                }
            }
        }
        
        // now dfs
        /*int cur_step = 0;
        int cur_end = 0;
        int len = table.size();
        stack<vector<int>> dfs;
        vector<int> record(2);
        for(int i = 1; i<len;i++)
        {
            if(dp[i] == i)
            {
                record[0] = i + dp[i];//cur_end
                record[1] = 1;//cur step
                dfs.push(record);
            }
        }
        while(!dfs.empty())
        {
            cur_end    = dfs.top()[0];
            cur_step   = dfs.top()[1];
            dfs.pop();
            if(cur_end == len-1)
            {
                if(cur_step == 3)
                    return true;
                else
                    continue;
            }
            if(cur_step == 3)
                continue;
            for(int i = cur_end + 1; i<len;i++)
            {
                if(dp[i] && i-dp[i] <= cur_end)
                {
                    record[0] = 2*i - cur_end;
                    record[1] = cur_step+1;
                    dfs.push(record);
                }
            }
        }*/
        return false;
    }
};
```
## 第50题
### 题目描述
1052. 爱生气的书店老板
难度
中等

今天，书店老板有一家店打算试营业 customers.length 分钟。每分钟都有一些顾客（customers[i]）会进入书店，所有这些顾客都会在那一分钟结束后离开。

在某些时候，书店老板会生气。 如果书店老板在第 i 分钟生气，那么 grumpy[i] = 1，否则 grumpy[i] = 0。 当书店老板生气时，那一分钟的顾客就会不满意，不生气则他们是满意的。

书店老板知道一个秘密技巧，能抑制自己的情绪，可以让自己连续 X 分钟不生气，但却只能使用一次。

请你返回这一天营业下来，最多有多少客户能够感到满意的数量。
 

示例：

输入：customers = [1,0,1,2,1,1,7,5], grumpy = [0,1,0,1,0,1,0,1], X = 3
输出：16
解释：
书店老板在最后 3 分钟保持冷静。
感到满意的最大客户数量 = 1 + 1 + 1 + 1 + 7 + 5 = 16.
 

提示：

1 <= X <= customers.length == grumpy.length <= 20000
0 <= customers[i] <= 1000
0 <= grumpy[i] <= 1
### 解答
滑动窗口
```
class Solution {
public:
    int maxSatisfied(vector<int>& customers, vector<int>& grumpy, int X) {
        int result = 0;
        int total = 0;
        for(int i = 0; i<customers.size();i++)
        {
            if(!grumpy[i])
                total += customers[i];
        }     
        int cur_gain = 0;
        for(int i = 0;i<X;i++)
            if(grumpy[i])
                cur_gain+=customers[i];
        //cout<<total<<' '<<cur_gain<<endl;
        result = total+cur_gain;
        for(int window_left = 1;window_left+X-1<customers.size();window_left++)
        {
            //cout<<total<<' '<<cur_gain<<' '<<window_left<<' '<<window_left+X<< endl;
            if(grumpy[window_left-1])
                cur_gain-=customers[window_left-1];
            if(grumpy[window_left+X-1])
                cur_gain+=customers[window_left+X-1];
            result = max(result,cur_gain+total);
        }
        return result;
    }
};
```
## 第51题
### 题目描述
30. 串联所有单词的子串
难度
困难

给定一个字符串 s 和一些长度相同的单词 words。找出 s 中恰好可以由 words 中所有单词串联形成的子串的起始位置。

注意子串要与 words 中的单词完全匹配，中间不能有其他字符，但不需要考虑 words 中单词串联的顺序。

 

示例 1：

输入：
  s = "barfoothefoobarman",
  words = ["foo","bar"]
输出：[0,9]
解释：
从索引 0 和 9 开始的子串分别是 "barfoo" 和 "foobar" 。
输出的顺序不重要, [9,0] 也是有效答案。
示例 2：

输入：
  s = "wordgoodgoodgoodbestword",
  words = ["word","good","best","word"]
输出：[]
### 解答
用的map做的，但是速度很慢，只击败了百分之几
```
class Solution {
public:
    vector<int> findSubstring(string s, vector<string>& words) {
        map<string,int> record;
        vector<int> result;
        string tmp;
        int l = words[0].size();
        int total_len = l*words.size();
        //int count = 0;
        bool flag = true;

        for(int i = 0;i<words.size();i++)
        {
            if(!record.count(words[i]))
            {
                record[words[i]] = 1;
            }
            else
                record[words[i]] += 1;
        }
        map<string,int> record_copy;
        for(int i =0;i+total_len-1<s.size();i++)
        {
            // init everytime
            record_copy.clear();
            record_copy.insert(record.begin(),record.end());
            flag = true;
            //count = words.size();
            tmp = s.substr(i,total_len);
            for(int j = 0; j < total_len; j+=l)
            {
                if(record_copy.count(tmp.substr(j,l)))//has this str
                {
                    if(record_copy[tmp.substr(j,l)] == 0)//used up
                    {
                        flag = false;
                        break;
                    }
                    else
                    {
                        record_copy[tmp.substr(j,l)]--;
                        //count--;
                    }
                }
                else
                {
                    flag = false;
                    break;
                }
            }
            if(flag)
                result.push_back(i);
        }
        return result;
    }
};
```

## 第51题
### 题目描述
1594. 矩阵的最大非负积
难度
中等

给你一个大小为 rows x cols 的矩阵 grid 。最初，你位于左上角 (0, 0) ，每一步，你可以在矩阵中 向右 或 向下 移动。

在从左上角 (0, 0) 开始到右下角 (rows - 1, cols - 1) 结束的所有路径中，找出具有 最大非负积 的路径。路径的积是沿路径访问的单元格中所有整数的乘积。

返回 最大非负积 对 109 + 7 取余 的结果。如果最大积为负数，则返回 -1 。

注意，取余是在得到最大积之后执行的。

 

示例 1：

输入：grid = [[-1,-2,-3],
             [-2,-3,-3],
             [-3,-3,-2]]
输出：-1
解释：从 (0, 0) 到 (2, 2) 的路径中无法得到非负积，所以返回 -1
示例 2：

输入：grid = [[1,-2,1],
             [1,-2,1],
             [3,-4,1]]
输出：8
解释：最大非负积对应的路径已经用粗体标出 (1 * 1 * -2 * -4 * 1 = 8)
示例 3：

输入：grid = [[1, 3],
             [0,-4]]
输出：0
解释：最大非负积对应的路径已经用粗体标出 (1 * 0 * -4 = 0)
示例 4：

输入：grid = [[ 1, 4,4,0],
             [-2, 0,0,1],
             [ 1,-1,1,1]]
输出：2
解释：最大非负积对应的路径已经用粗体标出 (1 * -2 * 1 * -1 * 1 * 1 = 2)
 

提示：

1 <= rows, cols <= 15
-4 <= grid[i][j] <= 4

### 解答
dp,坑在于要存储两个，一个正的一个负的，然后取余操作不能每次都取余，只需要最后的结果取余
```
class Solution {
public:
    int maxProductPath(vector<vector<int>>& grid) {
        int m = grid.size();
        int n = grid[0].size();
        bool flag = false;
        //int result = -1;
        vector< vector<vector<long long>> > dp(m,vector<vector<long long>>(n,vector<long long>(2)));
        if(grid[0][0]>0)
            dp[0][0][0] = grid[0][0];
        else if(grid[0][0]<0)
            dp[0][0][1] = grid[0][0];
        else    
        {
            flag = true;
            dp[0][0][0] = 0;
            dp[0][0][1] = 0;
        }
        
        for(int i = 1;i<m;i++)
        {
            if(grid[i][0] == 0)
                flag = true;
            if(grid[i][0] > 0)
            {
                dp[i][0][0] = (dp[i-1][0][0] * grid[i][0]);
                dp[i][0][1] = (dp[i-1][0][1] * grid[i][0]);
            }
            else
            {
                dp[i][0][1] = (dp[i-1][0][0] * grid[i][0]);
                dp[i][0][0] = (dp[i-1][0][1] * grid[i][0]);
            }
                
        }
        for(int j = 1;j<n;j++)
        {
            if(grid[0][j] == 0)
                flag = true;
            if(grid[0][j] > 0)
            {
                dp[0][j][0] = (dp[0][j-1][0] * grid[0][j]);
                dp[0][j][1] = (dp[0][j-1][1] * grid[0][j]);
            }
            else
            {
                dp[0][j][1] = (dp[0][j-1][0] * grid[0][j]);
                dp[0][j][0] = (dp[0][j-1][1] * grid[0][j]);
            }
        }
        for(int i = 1; i < m; i++)
        {
            for(int j = 1; j < n; j++)
            {
                if(grid[i][j] == 0)
                    flag = true;
                if(grid[i][j] > 0)
                {
                    dp[i][j][0] = max((dp[i-1][j][0]*grid[i][j]),(dp[i][j-1][0]*grid[i][j]));
                    dp[i][j][1] = min((dp[i-1][j][1]*grid[i][j]),(dp[i][j-1][1]*grid[i][j]));
                }
                else
                {
                    dp[i][j][0] = max((dp[i-1][j][1]*grid[i][j]),(dp[i][j-1][1]*grid[i][j]));
                    dp[i][j][1] = min((dp[i-1][j][0]*grid[i][j]),(dp[i][j-1][0]*grid[i][j]));
                }
            }
        }
            
        if(dp[m-1][n-1][0])
            return dp[m-1][n-1][0]%1000000007;
        else if(flag)
            return 0;
        else
            return -1;
    }
};
```
## 第52题
### 题目描述
1014. 最佳观光组合
难度
中等

给定正整数数组 A，A[i] 表示第 i 个观光景点的评分，并且两个景点 i 和 j 之间的距离为 j - i。

一对景点（i < j）组成的观光组合的得分为（A[i] + A[j] + i - j）：景点的评分之和减去它们两者之间的距离。

返回一对观光景点能取得的最高分。

 

示例：

输入：[8,1,5,2,6]
输出：11
解释：i = 0, j = 2, A[i] + A[j] + i - j = 8 + 5 + 0 - 2 = 11
 

提示：

2 <= A.length <= 50000
1 <= A[i] <= 1000
### 解答
我只想到了暴力，评论区老哥的想法非常好：
dp来做，稍微给这个公式变形成A[i]+i+A[j]-j，这样就可以看成是左A[i]+i和右A[j]-j两部分和的最大值。随着遍历数组，我们对两部分和取最大值，并且若当前的值—下标对之和比之前更大，我们就更新left部分的值即可。
```
class Solution {
public:
    int maxScoreSightseeingPair(vector<int>& values) {
        int left = values[0], res = INT_MIN;
        for(int i = 1 ; i<values.size(); i++)
        {
            res = max(left + values[i] - i,res);
            left = max(left,values[i] + i);
        }
        return res;
    }
};
```
## 第53题
### 题目描述
1363. 形成三的最大倍数
难度
困难

给你一个整数数组 digits，你可以通过按任意顺序连接其中某些数字来形成 3 的倍数，请你返回所能得到的最大的 3 的倍数。

由于答案可能不在整数数据类型范围内，请以字符串形式返回答案。

如果无法得到答案，请返回一个空字符串。

 

示例 1：

输入：digits = [8,1,9]
输出："981"
示例 2：

输入：digits = [8,6,7,1,0]
输出："8760"
示例 3：

输入：digits = [1]
输出：""
示例 4：

输入：digits = [0,0,0,0,0,0]
输出："0"
 

提示：

1 <= digits.length <= 10^4
0 <= digits[i] <= 9
返回的结果不应包含不必要的前导零。

### 解答
优先级队列记录余3等于0，1，2的数，等于0的直接加入答案中，等于1或者2的要讨论
如果1和2的数目相等，或者1的数量和2的数量余3都等于0，或者二者差值余3等于0，这三种情况全部加入
然后剩下的唯一情况就是1和2的数目不能全部用完，这时候总数是固定的，只需讨论剩几个就好
两种方案，一种方案是先配对，再三个一三个2这样，称之为plan1
第二种方案就是先三个1三个2，然后配对，plan2
plan1 = 二者差值余3
plan2 = 二者余三的差值
找一个剩的最少的方案即可
```
class Solution {
public:
    string largestMultipleOfThree(vector<int>& digits) {
        priority_queue <int,vector<int>,less<int> > q1;
        priority_queue <int,vector<int>,less<int> > q2;
        priority_queue <int,vector<int>,less<int> > record;
        for(int i = 0; i < digits.size();i++)
        {
            if(digits[i]%3 == 0)
            {
                record.push(digits[i]);
            }
            else if(digits[i]%3 == 1)
                q1.push(digits[i]);
            else
                q2.push(digits[i]);
        }
        int s1 = q1.size();
        int s2 = q2.size();
        if(q1.size() == q2.size() || (q1.size()%3 == 0 && q2.size()%3 == 0) || (abs(s1 - s2)%3  == 0) )// can full use
        {
            while(!q1.empty())
            {
                record.push(q1.top());
                q1.pop();
            }
            while(!q2.empty())
            {
                record.push(q2.top());
                q2.pop();
            }
        }
        else// some will left, now is to determine which plan can use most nums
        {
            //total is fixed, so just need to calc left
            int plan1 = abs(s1 - s2) %3;//321 left
            int plan2 = abs(s1%3 - s2%3);//123 left
            if(plan1 > plan2)//use 123
            {
                if(!q1.empty() && q1.size() >= 3)
                {
                    while(q1.size() >= 3)
                    {
                        record.push(q1.top());
                        q1.pop();
                        record.push(q1.top());
                        q1.pop();
                        record.push(q1.top());
                        q1.pop();
                    }
                }
                if(!q2.empty() && q2.size() >= 3)
                {
                    while(q2.size() >= 3)
                    {
                        record.push(q2.top());
                        q2.pop();
                        record.push(q2.top());
                        q2.pop();
                        record.push(q2.top());
                        q2.pop();
                    }
                }
                while(!q1.empty() && !q2.empty())
                {
                    record.push(q1.top());
                    record.push(q2.top());
                    q1.pop();
                    q2.pop();
                }
            }
            else//use 321
            {
                while(!q1.empty() && !q2.empty())
                {
                    record.push(q1.top());
                    record.push(q2.top());
                    q1.pop();
                    q2.pop();
                }
                if(!q1.empty() && q1.size() >= 3)
                {
                    while(q1.size() >= 3)
                    {
                        record.push(q1.top());
                        q1.pop();
                        record.push(q1.top());
                        q1.pop();
                        record.push(q1.top());
                        q1.pop();
                    }
                }
                if(!q2.empty() && q2.size() >= 3)
                {
                    while(q2.size() >= 3)
                    {
                        record.push(q2.top());
                        q2.pop();
                        record.push(q2.top());
                        q2.pop();
                        record.push(q2.top());
                        q2.pop();
                    }
                }
            }
        }
        if(!record.empty() && record.top() == 0)
            return "0";
        string result = "";
        while(!record.empty())
        {
            result.push_back('0'+ record.top());
            record.pop();
        }
        return result;
    }
};
```
## 第54题
### 题目描述
清华大学复试真题第一题
题目描述
设a、b、c均是0到9之间的数字，abc、bcc是两个三位数，且有：abc+bcc=532。求满足条件的所有a、b、c的值。
输入描述:
题目没有任何输入。
输出描述:
请输出所有满足题目条件的a、b、c的值。
a、b、c之间用空格隔开。
每个输出占一行。
### 解答
```
#include<bits/stdc++.h>
using namespace std;
int main()
{
    cout<<3<<' '<<2<<' '<<1<<endl;
    return 0;
}
```
## 第55题
### 题目描述
剑指 Offer 16. 数值的整数次方
难度
中等

实现函数double Power(double base, int exponent)，求base的exponent次方。不得使用库函数，同时不需要考虑大数问题。

 

示例 1:

输入: 2.00000, 10
输出: 1024.00000
示例 2:

输入: 2.10000, 3
输出: 9.26100
示例 3:

输入: 2.00000, -2
输出: 0.25000
解释: 2-2 = 1/22 = 1/4 = 0.25
 

说明:

-100.0 < x < 100.0
n 是 32 位有符号整数，其数值范围是 [−231, 231 − 1] 。
### 解答
学习到了，快速幂算法
https://leetcode-cn.com/problems/shu-zhi-de-zheng-shu-ci-fang-lcof/solution/mian-shi-ti-16-shu-zhi-de-zheng-shu-ci-fang-kuai-s/
```
class Solution {
public:
    double myPow(double x, int n) {
        if(n==0) return 1;
        //考虑到负数右移永远是负数，
        if(n==-1) return 1/x;
        if(n&1) return myPow(x*x, n>>1)*x;
        else return myPow(x*x, n>>1);
    }
};
```
## 第56题
### 题目描述
清华复试真题第二题
题目描述
    N<k时，root(N,k) = N，否则，root(N,k) = root(N',k)。N'为N的k进制表示的各位数字之和。输入x,y,k，输出root(x^y,k)的值 (这里^为乘方，不是异或)，2=<k<=16，0<x,y<2000000000，有一半的测试点里 x^y 会溢出int的范围(>=2000000000) 
输入描述:
    每组测试数据包括一行，x(0<x<2000000000), y(0<y<2000000000), k(2<=k<=16)
输出描述:
    输入可能有多组数据，对于每一组数据，root(x^y, k)的值
### 解答
不会。。。 妈的
①首先是由于x^y可能很大，会超时，所以用快速幂算法。
如果y是偶数，那么指数减半底数平方；如果y是奇数，那么给最终的结果乘上x的一次方，这样能够求出x^y的结果res。
②再来看root（N，k），根据题意有N = a0 + a1*k + a2*k^2 +...
N' = a0 + a1 + a2+ ...
两式相减能够得到N-N'被k-1整除。
所以N' = N%(k-1)，但由于N'是在[0,k-1]之中，而N' = k-1这种情况会在N%(k-1)中算为0，所以加一个条件判断。
③quickpow 中还利用了数学上的定理(x * y) % k = (x % k) * (y % k) % k
```
#include<iostream>

using namespace std;

long long quickpower(long long x, long long y, int k){
    long long ans = 1, temp = x;
    while(y){
        if(y & 1) ans = ans * temp % (k - 1);
        temp = temp * temp % (k - 1);
        y >>= 1;
    }
    return ans ? ans : k - 1;
}

int main(){
    int x, y, k;
    while(cin >> x >> y >> k){
        cout << quickpower(x, y, k) << endl;
    }
    return 0;
}
```
## 第57题
### 题目描述
清华大学复试第三题
题目描述
输入一个整数n，输出n的阶乘（每组测试用例可能包含多组数据，请注意处理）
输入描述:
一个整数n(1<=n<=20)
输出描述:
n的阶乘
### 解答
```
#include<bits/stdc++.h>
using namespace std;
int main()
{
    vector<long long> record(21,1);
    for(int i = 2;i<21;i++)
        record[i] = record[i-1] * i;
    int n;
    while(cin>>n)
        cout<<record[n]<<endl;
}
```
## 第58题
### 题目描述
清华大学复试第四题
题目描述
写个算法，对2个小于1000000000的输入，求结果。 特殊乘法举例：123 * 45 = 1*4 +1*5 +2*4 +2*5 +3*4+3*5
输入描述:
两个小于1000000000的数
输出描述:
输入可能有多组数据，对于每一组数据，输出Input中的两个数按照题目要求的方法进行运算后得到的结果。
### 解答
```
#include<bits/stdc++.h>
using namespace std;
int main()
{
    int x,y;
    int tmpy,tmpx;
    int result = 0;
    while(cin>>x>>y)
    {
        result = 0;
        while(x>0)
        {
            tmpx = x%10;
            tmpy = y;
            while(tmpy>0)
            {
                result += (tmpx*(tmpy%10));
                tmpy/=10;
            }
            x/=10;
        }
        cout<<result<<endl;
    }
}
```
## 第59题
### 题目描述
清华大学复试第5题
题目描述
输入年、月、日，计算该天是本年的第几天。
输入描述:
包括三个整数年(1<=Y<=3000)、月(1<=M<=12)、日(1<=D<=31)。
输出描述:
输入可能有多组测试数据，对于每一组测试数据，
输出一个整数，代表Input中的年、月、日对应本年的第几天。
### 解答
```
#include<bits/stdc++.h>
using namespace std;
bool is_leapyear(int year)
{
    return (year%4==0&&year%100!=0) || year%400 == 0;
}
int main()
{
    vector<int> leap{31,29,31,30,31,30,31,31,30,31,30,31};
    vector<int> norm{31,28,31,30,31,30,31,31,30,31,30,31};
    int year,month,day;
    int result = 0;
    while(cin>>year>>month>>day)
    {
        result = 0;
        if(is_leapyear(year))
        {
            for(int i = 0;i < month-1;i++)
            {
                result +=leap[i];
            }
            result += day;
        }
        else
        {
            for(int i = 0;i < month-1;i++)
            {
                result +=norm[i];
            }
            result += day;
        }
        cout<<result<<endl;
    }
}
```
## 第60题
### 题目描述
清华大学复试第六题
题目描述
一个数如果恰好等于它的各因子(该数本身除外)子和，如：6=3+2+1。则称其为“完数”；若因子之和大于该数，则称其为“盈数”。 求出2到60之间所有“完数”和“盈数”。
输入描述:
题目没有任何输入。
输出描述:
输出2到60之间所有“完数”和“盈数”，并以如下形式输出：
E: e1 e2 e3 ......(ei为完数)
G: g1 g2 g3 ......(gi为盈数)
其中两个数之间要有空格，行尾不加空格。
### 解答
```
#include<bits/stdc++.h>
using namespace std;
int judge(int n)
{
    int tmp = 0;
    for(int i = 1;i<=n/2; i++)
    {
        if(n%i == 0)
            tmp += i;
    }
    if(tmp>n)
        return 1;
    else if(tmp == n)
        return 0;
    else
        return -1;
}
int main()
{
    vector<int> com;
    vector<int> big;
    for(int i = 2;i<=60;i++)
    {
        if(judge(i) == 1)
            big.push_back(i);
        else if(judge(i) == 0)
            com.push_back(i);
            
    }
    cout<<"E:";
    for(int i = 0;i<com.size();i++)
        cout<<' '<<com[i];
    cout<<endl;
    cout<<"G:";
    for(int i = 0;i<big.size();i++)
        cout<<' '<<big[i];
    cout<<endl;
}
```

## 第61题
### 题目描述
清华大学复试第七题
题目描述
给定a0,a1,以及an=p*a(n-1) + q*a(n-2)中的p,q。这里n >= 2。 求第k个数对10000的模。
输入描述:
输入包括5个整数：a0、a1、p、q、k。
输出描述:
第k个数a(k)对10000的模。
### 解答
```
#include<bits/stdc++.h>
using namespace std;
int main()
{
    long long a0,a1,p,q,k;
    cin>>a0>>a1>>p>>q>>k;
    vector<long long> result(k+1);
    result[0] = a0;
    result[1] = a1;
    for(int i =2;i<k+1;i++)
    {
        result[i] = (p*result[i-1] + q*result[i-2])%10000;
    }
    cout<<result[k]%10000<<endl;
}
```
## 第62题
### 题目描述
清华大学复试第八题
题目描述
给出一个整数序列S，其中有N个数，定义其中一个非空连续子序列T中所有数的和为T的“序列和”。 对于S的所有非空连续子序列T，求最大的序列和。 变量条件：N为正整数，N≤1000000，结果序列和在范围（-2^63,2^63-1）以内。
输入描述:
第一行为一个正整数N，第二行为N个整数，表示序列中的数。
输出描述:
输入可能包括多组数据，对于每一组输入数据，
仅输出一个数，表示最大序列和。
### 解答
最大数列和，老生常谈了
```
#include<bits/stdc++.h>
using namespace std;
int main()
{
    //vector<int> record(1000000);
    int n;
    int tmp = 0;
    int max = 0;
    int cur_sum = 0;
    while(cin>>n)
    {
        cin>>tmp;
        max = tmp;
        cur_sum = tmp;
        for(int i =1;i<n;i++)
        {
            cin>>tmp;
            cur_sum += tmp;
            if(cur_sum >max)
            {
                max = cur_sum;
            }
            if(cur_sum < 0)
            {
                cur_sum = 0;
            }
        }
        cout<<max<<endl;
    }
}
```
## 第63题
### 题目描述
清华大学复试第九题
题目描述
    有一个长度为整数L(1<=L<=10000)的马路，可以想象成数轴上长度为L的一个线段，起点是坐标原点，在每个整数坐标点有一棵树，即在0,1,2，...，L共L+1个位置上有L+1棵树。     现在要移走一些树，移走的树的区间用一对数字表示，如 100 200表示移走从100到200之间（包括端点）所有的树。     可能有M(1<=M<=100)个区间，区间之间可能有重叠。现在要求移走所有区间的树之后剩下的树的个数。
输入描述:
    两个整数L(1<=L<=10000)和M(1<=M<=100)。
    接下来有M组整数，每组有一对数字。
输出描述:
    可能有多组输入数据，对于每组输入数据，输出一个数，表示移走所有区间的树之后剩下的树的个数。
### 解答
集合运算，卡在了set用法上，没办法才用了vector，算是目前为止遇到清华的复试题比较复杂的一个了
```
#include<bits/stdc++.h>
using namespace std;
int main()
{
    vector<vector<int>> input_set(100,vector<int>(2));
    vector<vector<int>>   processed_set;
    int N,M;
    int front,tail;
    while(cin>>N>>M)
    {
        processed_set.clear();
        for(int i =0;i<M;i++)
            cin>>input_set[i][0]>>input_set[i][1];
        for(int i = 0;i<M;i++)
        {
            int pre = -1;
            bool flag = true;
            front = input_set[i][0];
            tail = input_set[i][1];
            for(int j = 0;j<processed_set.size();j++)
            {
                if( (front >= processed_set[j][0]&& front <=processed_set[j][1]) ||(tail >= processed_set[j][0] && tail <= processed_set[j][1]) || (front <=processed_set[j][0] && tail>= processed_set[j][1]))
                {
                    processed_set[j][0] = min(front,processed_set[j][0]);
                    processed_set[j][1] = max(tail,processed_set[j][1]);
                    front = processed_set[j][0];
                    tail = processed_set[j][1];
                    flag = false;
                    if(pre!= -1)
                    {
                        processed_set[pre][0] = -1;
                        processed_set[pre][1] = -1;
                    }
                    pre = j;
                    
                }
            }
            if(flag)
            {
                processed_set.push_back({front,tail});
            }
        }
        for(int j = 0;j<processed_set.size();j++)
            if(processed_set[j][0] != -1)
            {
                N-=(processed_set[j][1] - processed_set[j][0] + 1);
                //cout<<processed_set[j][0]<<' '<<processed_set[j][1]<<endl;
            }

        cout<<N+1<<endl;
    }
}
```
## 第64题
### 题目描述
清华大学复试第十题
题目描述
    对于一个十进制数A，将A转换为二进制数，然后按位逆序排列，再转换为十进制数B，我们称B为A的二进制逆序数。     例如对于十进制数173，它的二进制形式为10101101，逆序排列得到10110101，其十进制数为181，181即为173的二进制逆序数。
输入描述:
    一个1000位(即10^999)以内的十进制数。
输出描述:
    输入的十进制数的二进制逆序数。
### 解答
这道题主要麻烦在大数乘法除法上
```
#include<bits/stdc++.h>
using namespace std;
string divide_2(string s, bool &flag)
{
    string result = "";
    int tmp = 0;
    for(int i = 0;i<s.size();i++)
    {
        tmp = tmp*10 + s[i] - '0';
        result += tmp/2 + '0';
        tmp = tmp%2;
    }
    if(tmp)
        flag = true;//means odd num
    return result;
}
string multi_2(string s)
{
    string result(s.size()+1,'0');
    int carry = 0;
    int tmp = 0;
    for(int i = s.size()-1 ;i>=0;i--)
    {
        //cout<<result<<endl;
        tmp = (s[i] - '0') *2;
        if(tmp +carry >= 10)
        {
            result[i+1] = (tmp +carry)%10 +'0';
            carry = 1;
        }
        else
        {
            result[i+1] = tmp +carry +'0';
            carry = 0;
        }
    }
    if(carry)
        result[0] = '1';
    else
        if(result.size()>1)
            result = result.substr(1,s.size());
    return result;
}
string convert(string s)
{
    string result = "";
    bool flag = false;
    while(s.size() > 0)
    {
        flag = false;
        s = divide_2(s,flag);
        //cout<<s<<endl;
        if(flag)
            result += '1';
        else
            result += '0';
        int i = 0;
        while(s[i] == '0')
            i++;
        s = s.substr(i,s.size() - i);
    }
    //now result is a binary reverse string, all we need is to transfer this string to decimal
    //cout<<result<<endl;
    string ans = "0";
    for(int i =0;i<result.size();i++)
    {
        ans = multi_2(ans);
        ans[ans.size() -1] += result[i] - '0';
        
        //cout<<ans<<endl;
    }
    return ans;
    
}
int main()
{
    string s;
    cin>>s;
    //cout<<multi_2(s)<<endl;
    cout<<convert(s)<<endl;
}
```
## 第65题
### 题目描述
清华大学复试第11题
题目描述
输入N个学生的信息，然后进行查询。
输入描述:
输入的第一行为N，即学生的个数(N<=1000)
接下来的N行包括N个学生的信息，信息格式如下：
01 李江 男 21
02 刘唐 男 23
03 张军 男 19
04 王娜 女 19
然后输入一个M(M<=10000),接下来会有M行，代表M次查询，每行输入一个学号，格式如下：
02
03
01
04
输出描述:
输出M行，每行包括一个对应于查询的学生的信息。
如果没有对应的学生信息，则输出“No Answer!”
### 解答
简单的哈希表
```
#include<bits/stdc++.h>
using namespace std;
int main()
{
    int N;
    cin>>N;
    string tmp1;
    string tmp = "";
    string id;
    map<string,string> record;
    for(int i = 0;i<N ; i++)
    {
        cin>>id;
        tmp = id;
        for(int i = 0;i<3;i++)
        {
            cin>>tmp1;
            tmp+=' ';
            tmp += tmp1;
        }
        record[id] = tmp;
    }
    cin>>N;
    for(int i = 0; i<N ; i++)
    {
        cin>>id;
        if(record.count(id))//has
        {
            cout<<record[id]<<endl;
        }
        else
            cout<<"No Answer!"<<endl;
    }
}
```
## 第66题
### 题目描述
清华大学复试第11题
题目描述
玛雅人有一种密码，如果字符串中出现连续的2012四个数字就能解开密码。给一个长度为N的字符串（2=<N<=13），该字符串中只含有0,1,2三种数字，问这个字符串要移位几次才能解开密码，每次只能移动相邻的两个数字。例如02120经过一次移位，可以得到20120,01220,02210,02102，其中20120符合要求，因此输出为1.如果无论移位多少次都解不开密码，输出-1。

输入描述:
第一行输入N，第二行输入N个数字，只包含0，1，2
输出描述:
输出字符串要移几位才能解开密码，如果无论移位多少次都解不开密码，输出-1

### 解答
广度优先搜索，利用set标志访问
```
#include<bits/stdc++.h>
using namespace std;
int main()
{
    int n;
    string s;
    set<string> s_set;
    queue<pair<string,int>>q;
    string tmp;
    int step;
    while(cin>>n)
    {
        bool flag = false;
        cin>>s;
        while(!q.empty())
            q.pop();
        s_set.clear();
        q.push(make_pair(s,0));
        s_set.emplace(s);
        while(!q.empty())
        {
            s = q.front().first;
            step = q.front().second;
            q.pop();
            if(s.find("2012") != s.npos)
            {
                cout<<step<<endl;
                flag = true;
                break;
            }
            else
            {
                for(int i = 0; i<s.size() - 1; i++)
                {
                    tmp = s;
                    tmp[i] = s[i+1];
                    tmp[i+1] = s[i];
                    if(s_set.count(tmp) == 0)
                    {
                        s_set.emplace(tmp);
                        q.push(make_pair(tmp,step+1));
                    }
                }
            }
        }
        if(!flag)
            cout<<-1<<endl;
    }
}
```
## 第67题
### 题目描述
清华大学复试第十二题
题目描述
将M进制的数X转换为N进制的数输出。
输入描述:
输入的第一行包括两个整数：M和N(2<=M,N<=36)。
下面的一行输入一个数X，X是M进制的数，现在要求你将M进制的数X转换成N进制的数输出。
输出描述:
输出X的N进制表示的数。
### 解答
大数除法
```
#include<iostream>
#include<string>
using namespace std;
string divide(string origin,int origin_base,int target_base, int & num)
{
    string result = "";
    int n = origin.size();
    int tmp = 0;
    int remain = 0;
    for(int i = 0;i<n;i++)
    {
        //cout<<remain<<endl;
        remain *= origin_base;
        if(origin[i] >='0' && origin[i] <='9')
        {
            remain += origin[i] - '0';
        }
        else
        {
            remain += origin[i] - 'A' + 10;
        }
        tmp = remain/target_base;
        remain = remain%target_base;
        if(tmp>9)
            result +=('A' + tmp - 10);
        else
            result += ('0' + tmp);
    }
    num = remain;
    //cout<<result<<endl;
    return result;
}
string multi(string origin,int origin_base,int target_base)
{
    string result(origin.size()+1,'0');
    int carry = 0;
    int tmp = 0;
    for(int i = origin.size() - 1; i>=0 ; i-- )
    {
        if(origin[i] >='0' && origin[i] <='9')
        {
            tmp = origin[i] - '0';
        }
        else
        {
            tmp = origin[i] - 'A' + 10;
        }
        tmp *= target_base;
        tmp += carry;
        carry = tmp/origin_base;
        tmp %= origin_base;
        if(tmp > 9)
            result[i+1] = (tmp - 10 + 'A');
        else
            result[i+1] = (tmp + '0');
    }
    if(carry)
    {
        if(carry > 9)
            result[0] = (carry + 'A' - 10);
        else
            result[0] = '0' + carry;
        return result;
    }
    else
        return result.substr(1,result.size()-1);
}
int main()
{
    string origin;
    string result = "";
    long long  origin_base;
    long long  target_base;
    cin>>origin_base>>target_base;
    cin>>origin;
    //cout<<origin<<' '<<origin_base<<' '<<target_base<<endl;
    int remain;
    while(origin.size() > 0)
    {
        origin = divide(origin,origin_base,target_base,remain);
        //cout<<origin<<' '<<remain<<endl;
        
        if(remain > 9)
            result += 'a' + remain - 10;
        else
            result += '0' + remain;
        while(origin[0] == '0')
            origin = origin.substr(1,origin.size() - 1);
    }
    for(int i = result.size() - 1; i >= 0 ; i-- )
        cout<<result[i];
    cout<<endl;
    
}
```
## 第68题
### 题目描述
清华大学复试第十三题
时间限制：C/C++ 1秒，其他语言2秒 空间限制：C/C++ 32M，其他语言64M 热度指数：10731
本题知识点： 数学
 算法知识视频讲解
校招时部分企业笔试将禁止编程题跳出页面，为提前适应，练习时请使用在线自测，而非本地IDE。
题目描述
设N是一个四位数，它的9倍恰好是其反序数（例如：1234的反序数是4321）
求N的值
输入描述:
程序无任何输入数据。
输出描述:
输出题目要求的四位数，如果结果有多组，则每组结果之间以回车隔开。
### 解答
```
#include<bits/stdc++.h>
using namespace std;
int reverse(int n)
{
    int result = 0;
    result += n/1000;
    n%=1000;
    result += (n/100) *10;
    n%=100;
    result += (n/10) * 100;
    n%=10;
    result += n*1000;
    return result;
}
int main()
{
    for(int i = 1000; i <= 1111 ; i++)
    {
        if(9*i == reverse(i))
            cout<<i<<endl;
    }
}
```
## 第69题
### 题目描述
清华大学复试第14题
题目描述
打印所有不超过256，其平方具有对称性质的数。如2，11就是这样的数，因为2*2=4，11*11=121。
输入描述:
无任何输入数据
输出描述:
输出具有题目要求的性质的数。如果输出数据不止一组，各组数据之间以回车隔开。
### 解答
```
#include<bits/stdc++.h>
using namespace std;
int reverse(int n)
{
    int result = 0;
    int tmp = 0;
    while(n>0)
    {
        result*=10;
        tmp = n%10;
        result += tmp;
        n/=10;
    }
    return result;
}
bool judge(int n)
{
    int tmp = n*n;
    return tmp == reverse(tmp);
    
}
int main()
{
    for(int i = 0;i<=256;i++)
    {
        if(judge(i))
            cout<<i<<endl;
    }
}
```
## 第70题
### 题目描述
清华大学复试第15题
题目描述
    使用代理服务器能够在一定程度上隐藏客户端信息，从而保护用户在互联网上的隐私。我们知道n个代理服务器的IP地址，现在要用它们去访问m个服务器。这 m 个服务器的 IP 地址和访问顺序也已经给出。系统在同一时刻只能使用一个代理服务器，并要求不能用代理服务器去访问和它 IP地址相同的服务器（不然客户端信息很有可能就会被泄露）。在这样的条件下，找到一种使用代理服务器的方案，使得代理服务器切换的次数尽可能得少。
输入描述:
    每个测试数据包括 n + m + 2 行。
    第 1 行只包含一个整数 n，表示代理服务器的个数。
    第 2行至第n + 1行每行是一个字符串，表示代理服务器的 IP地址。这n个 IP地址两两不相同。
    第 n + 2 行只包含一个整数 m，表示要访问的服务器的个数。
    第 n + 3 行至第 n + m + 2 行每行是一个字符串，表示要访问的服务器的 IP 地址，按照访问的顺序给出。
    每个字符串都是合法的IP地址，形式为“xxx.yyy.zzz.www”，其中任何一部分均是0–255之间的整数。输入数据的任何一行都不包含空格字符。
     其中，1<=n<=1000，1<=m<=5000。
输出描述:
    可能有多组测试数据，对于每组输入数据， 输出数据只有一行，包含一个整数s，表示按照要求访问服务器的过程中切换代理服务器的最少次数。第一次使用的代理服务器不计入切换次数中。若没有符合要求的安排方式，则输出-1。
### 解答
每次访问最后一个遇到的proxy，然后清空记录缓存
```
#include<bits/stdc++.h>
using namespace std;
int main()
{
    int n,m;
    cin>>n;
    map<string,bool> proxy_record;
    string tmp;
    for(int i = 0;i<n;i++)
    {
        cin>>tmp;
        proxy_record[tmp] = false;
    }
    cin>>m;
    vector<string> visit_array;
    for(int i = 0; i< m ; i++)
    {
        cin>>tmp;
        visit_array.push_back(tmp);
    }
    int count = 0;
    int result = 0;
    for(int i = 0 ; i < m ; i++)
    {
        tmp = visit_array[i];
        if(proxy_record.count(tmp) && proxy_record[tmp] == false)
        {
            count ++;
            proxy_record[tmp] = true;
            if(count == n)
            {
                count = 1;
                result ++;
                for(auto it = proxy_record.begin() ; it != proxy_record.end() ; it++)
                {
                    if(it->first!=tmp)
                        it->second = false;
                }
            }
        }
        else if(n==1&& proxy_record[tmp] == true)
        {
            result = -1;
            break;
        }
    }

    cout<<result<<endl;
}
```
## 第71题
### 题目描述
清华大些复试第16题
题目描述
输入任意4个字符(如：abcd)， 并按反序输出(如：dcba)
输入描述:
题目可能包含多组用例，每组用例占一行，包含4个任意的字符。
输出描述:
对于每组输入,请输出一行反序后的字符串。
具体可见样例。
### 解答
```
#include<bits/stdc++.h>
using namespace std;
int main()
{
    string s;
    string result = "AAAA";
    while(cin>>s)
    {
        result[0] = s[3];
        result[1] = s[2];
        result[2] = s[1];
        result[3] = s[0];
        cout<<result<<endl;
    }
}
```
## 第72题
### 题目描述
清华大学复试第17题
题目描述
按照手机键盘输入字母的方式，计算所花费的时间 如：a,b,c都在“1”键上，输入a只需要按一次，输入c需要连续按三次。 如果连续两个字符不在同一个按键上，则可直接按，如：ad需要按两下，kz需要按6下 如果连续两字符在同一个按键上，则两个按键之间需要等一段时间，如ac，在按了a之后，需要等一会儿才能按c。 现在假设每按一次需要花费一个时间段，等待时间需要花费两个时间段。 现在给出一串字符，需要计算出它所需要花费的时间。
输入描述:
一个长度不大于100的字符串，其中只有手机按键上有的小写字母
输出描述:
输入可能包括多组数据，对于每组数据，输出按出Input所给字符串所需要的时间
### 解答
```
#include<bits/stdc++.h>
using namespace std;
int main()
{
    vector<int> layout = {1,1,1,2,2,2,3,3,3,4,4,4,5,5,5,6,6,6,6,7,7,7,8,8,8,8};
    vector<int> times =  {1,2,3,1,2,3,1,2,3,1,2,3,1,2,3,1,2,3,4,1,2,3,1,2,3,4};
    string s;
    int pre;
    int cur;
    int result;
    while(cin>>s)
    {
        pre = s[0] - 'a';
        result = times[pre];
        for(int i = 1 ; i<s.size() ; i++)
        {
            cur = s[i] - 'a';
            if(layout[cur] == layout[pre])
                result+=2;
            result += times[cur];
            pre = cur;
        }
        cout<<result<<endl;
    }
}
```
## 第73题
### 题目描述
清华大学复试第18题
题目描述
求正整数N(N>1)的质因数的个数。 相同的质因数需要重复计算。如120=2*2*2*3*5，共有5个质因数。
输入描述:
可能有多组测试数据，每组测试数据的输入是一个正整数N，(1<N<10^9)。
输出描述:
对于每组数据，输出N的质因数的个数。
### 解答
```
#include<bits/stdc++.h>
using namespace std;
bool is_prime(int n)
{
    for(int i = 2 ; i*i <= n ; i++)
    {
        if(n%i == 0)
            return false;
    }
    return true;
}
int main()
{
    int n;
    int result = 0;
    while(cin>>n)
    {
        result = 0;
        while(!is_prime(n))
        {
            for(int i = 2;i*i<=n;i++)
            {
                if(n%i==0 )
                {
                    n/=i;
                    break;
                }
            }
            result++;
        }
        result++;
        cout<<result<<endl;
    }
}
```
## 第74题
### 题目描述
清华大学复试第19题
题目描述
一个整数总可以拆分为2的幂的和，例如： 7=1+2+4 7=1+2+2+2 7=1+1+1+4 7=1+1+1+2+2 7=1+1+1+1+1+2 7=1+1+1+1+1+1+1 总共有六种不同的拆分方式。 再比如：4可以拆分成：4 = 4，4 = 1 + 1 + 1 + 1，4 = 2 + 2，4=1+1+2。 用f(n)表示n的不同拆分的种数，例如f(7)=6. 要求编写程序，读入n(不超过1000000)，输出f(n)%1000000000。
输入描述:
每组输入包括一个整数：N(1<=N<=1000000)。
输出描述:
对于每组数据，输出f(n)%1000000000。
### 解答
搬运一下思路：
记f(n)为n的划分数，我们有递推公式：
 
    f(2m + 1) = f(2m)，
f(2m) = f(2m - 1) + f(m)，
初始条件：f(1) = 1。
 
    证明:
 
    证明的要点是考虑划分中是否有1。
 
    记:
A(n) = n的所有划分组成的集合，
B(n) = n的所有含有1的划分组成的集合，
C(n) = n的所有不含1的划分组成的集合，
则有: A(n) = B(n)∪C(n)。
 
    又记:
f(n) = A(n)中元素的个数，
g(n) = B(n)中元素的个数，
h(n) = C(n)中元素的个数，
易知: f(n) = g(n) + h(n)。
 
    以上记号的具体例子见文末。
 
    我们先来证明: f(2m + 1) = f(2m)，
首先，2m + 1 的每个划分中至少有一个1，去掉这个1，就得到 2m 的一个划分，故 f(2m + 1)≤f(2m)。
其次，2m 的每个划分加上个1，就构成了 2m + 1 的一个划分，故 f(2m)≤f(2m + 1)。
综上，f(2m + 1) = f(2m)。
 
    接着我们要证明: f(2m) = f(2m - 1) + f(m)，
把 B(2m) 中的划分中的1去掉一个，就得到 A(2m - 1) 中的一个划分，故 g(2m)≤f(2m - 1)。
把 A(2m - 1) 中的划分加上一个1，就得到 B(2m) 中的一个划分，故 f(2m - 1)≤g(2m)。
综上，g(2m) = f(2m - 1)。
 
    把 C(2m) 中的划分的元素都除以2，就得到 A(m) 中的一个划分，故 h(2m)≤f(m)。
把 A(m) 中的划分的元素都乘2，就得到 C(2m) 中的一个划分，故 f(m)≤h(2m)。
综上，h(2m) = f(m)。
 
    所以: f(2m) = g(2m) + h(2m) = f(2m - 1) + f(m)。                                            
 
    这就证明了我们的递推公式。
    
```
#include<bits/stdc++.h>
using namespace std;
int main()
{
    int n;
    vector<long long> dp(1000001,-1);
    dp[0] = 0;
    dp[1] = 1;
    dp[2] = 2;
    dp[3] = 2;
    dp[4] = 3;
    int max_index = 4;
    while(cin>>n)
    {
        if(max_index >=n)
            cout<<dp[n]<<endl;
        else
        {
            for(int i = max_index; i<=n;i++)
            {
                if(i%2)
                    dp[i] = dp[i-1];
                else
                    dp[i] = (dp[i-1] + dp[i/2])%1000000000;
            }
            cout<<dp[n]%1000000000<<endl;
            max_index = n;
        }
    }
}
```
## 第75题
### 题目描述
清华大学复试第20题
题目描述
用一维数组存储学号和成绩，然后，按成绩排序输出。
输入描述:
输入第一行包括一个整数N(1<=N<=100)，代表学生的个数。
接下来的N行每行包括两个整数p和q，分别代表每个学生的学号和成绩。
输出描述:
按照学生的成绩从小到大进行排序，并将排序后的学生信息打印出来。
如果学生的成绩相同，则按照学号的大小进行从小到大排序。
### 解答
```
#include<bits/stdc++.h>
using namespace std;
bool cmp(const vector<int> a,const vector<int> b)
{
    if(a[1] == b[1])
        return a[0]<b[0];
    else
        return a[1]<b[1];
}
int main()
{
    int n;
    cin>>n;
    vector<vector<int>> record(n,vector<int>(2));
    for(int i = 0; i < n ; i++)
    {
        cin>>record[i][0]>>record[i][1];
    }
    sort(record.begin(),record.end(),cmp);
    for(int i = 0; i < n ; i++)
    {
        cout<<record[i][0]<<' '<<record[i][1]<<endl;
    }
}
```
## 第76题
### 题目描述
清华大学复试第21题
题目描述
输入球的中心点和球上某一点的坐标，计算球的半径和体积
输入描述:
球的中心点和球上某一点的坐标，以如下形式输入：x0 y0 z0 x1 y1 z1
输出描述:
输入可能有多组，对于每组输入，输出球的半径和体积，并且结果保留三位小数

为避免精度问题，PI值请使用arccos(-1)。
### 解答
学习了精度设定
```
#include<bits/stdc++.h>
using namespace std;
int main()
{
    double x0,y0,z0;
    double x1,y1,z1;
    cin>>x0>>y0>>z0>>x1>>y1>>z1;
    double radius = sqrt((x0-x1)*(x0-x1) + (y0-y1)*(y0-y1) + (z0-z1)*(z0-z1));
    double V = (4.0/3.0)*acos(-1) * radius*radius*radius;
    cout<<setiosflags(ios::fixed)<<setprecision(3);
    cout<<radius<<' '<<V<<endl;
}
```
## 第77题
### 题目描述
题目描述
    有若干张邮票，要求从中选取最少的邮票张数凑成一个给定的总值。     如，有1分，3分，3分，3分，4分五张邮票，要求凑成10分，则使用3张邮票：3分、3分、4分即可。
输入描述:
    有多组数据，对于每组数据，首先是要求凑成的邮票总值M，M<100。然后是一个数N，N〈20，表示有N张邮票。接下来是N个正整数，分别表示这N张邮票的面值，且以升序排列。
输出描述:
      对于每组数据，能够凑成总值M的最少邮票张数。若无解，输出0。
      
### 解答
01背包问题 看懂怎么回事了，需要练习
```
#include<iostream>
#include<cstdio>
 
using namespace std;
 
const int maxn=100;
 
int weight[maxn];
int dp[maxn];
 
//每张邮票价值为1，求背包容量M,N个物品，装满的前提下获得最小的价值；
 
//对于dp[i][j]来说，
//（1）如果放入第i个物品可以填满j容量的背包，那么使用前i-1个物品可以填满j-w[i]容量的背包，于是dp[i][j]=dp[i-1][j-w[i]]+1；
//（2）如果不放入第i个物品也可以填满j容量的背包,意味着只使用前i-1个物品仍然可以填满j容量的背包，dp[i][j]=dp[i-1][j]。
//总结以上两种情况：dp[i][j]取其中较小的。
 
//如果前i个物品不能存满背包，设置dp[i][j]为一个超过总价值的数，当更大的i,j需要使用较小的i,j的dp时，dp都不能存满背包，则较大的dp也不能存满背包。
//边界条件：dp[0][j]=MAX,不存放物品时，任何非零容量都不能存满;dp[i][0]=0，容量为0，总是可以填满且总价值为0；
//状态转移：dp[i][j]=min(dp[i-1][j-w[i]]+1,dp[i-1][j]);任何情况下，dp都不会超过设置的max值。
//优化：可以看到dp[i][j]只和上一行的数据有关，简化成dp[j]=min(dp[j-w[i]]+1,dp[j])，且遍历到dp[j]时，dp[j-w[i]]值应还未修改，所以j从后向前遍历。
int main(){
    int M,N;
    while(cin>>M>>N){
        for(int i=0;i<N;++i){
            scanf("%d",&weight[i]);
        }
        for(int i=0;i<=M;i++){
            dp[i]=maxn;
        }
        dp[0]=0;
        for(int i=0;i<N;i++){
            for(int j=M;j>=weight[i];j--){
                dp[j]=min(dp[j-weight[i]]+1,dp[j]);
            }
        }
        if(dp[M]==maxn){
            cout<<0<<endl;
        }else{
            cout<<dp[M]<<endl;
        }
    }
    return 0;
}
```
## 第78题
### 题目描述
某天，吴大佬准备和菜鸡Tirpitz一起组队刷题，聪明的吴大佬把题目分成了n个板块，每个板块有w[i]个题目，刷完这个板块需要消耗吴大佬m[i]的精力。吴大佬没有必要在一个板块死磕，/*毕竟有咸鱼队友Tirpitz*/。相反，如果他消耗m[i]%的精力时，他会解决这个专题w[i]%的题目，现在吴大佬想给聪明的你一个任务，让你计算吴大佬一共能做多少道题目？（没有a完就算成小数累加哦w/*这个也算是吴大佬的贡献嘛*/）

输入输出格式
输入描述:

输入由多个测试用例组成，每个测试用例是有两个非负整数m（总的精力），n的行作为第一行，然后后面有n行跟随，每行包括两个非负整数w[i],m[i]，最后一个测试用例后面有一组 -1 -1（所有的整数都不大于1000，毕竟人类是有极限的嘛hhh）
输出描述:

对于每一个测试用例，在一行中输出吴大佬可以做出的题目数目，精确到小数点后3位
### 解答
贪心
```
#include <stdio.h>
#include <string.h>
#include <math.h>
#include <stdlib.h>
#include <time.h>
#include <algorithm>
#include <iostream>
#include <queue>
#include <stack>
#include <vector>
#include <string>
#include<iomanip>
using namespace std;
bool cmp(vector<double> a, vector<double> b)
{
    return (a[0]/a[1]) >(b[0]/b[1]);
}
int main()
{
    int total = 0;
    int n = 0;
    double result = 0;
    vector<vector<double>> record(1000,vector<double>(2));
    while(cin>>total>>n)
    {
        if(total == -1 && n == -1)
            break;
        for(int i = 0;i<n;i++)
        {
            cin>>record[i][0]>>record[i][1];
        }
        sort(record.begin(),record.begin()+n,cmp);
        for(int i = 0; i<n && total >0;i++)
        {
            if(total >= record[i][1])
            {
                result += record[i][0];
                total -= record[i][1];
            }
            else
            {
                result += (total/record[i][1]) * record[i][0];
                break;
            }
        }
        cout<<setiosflags(ios::fixed)<<setprecision(3);
        cout<<result<<endl;

    }
}
```
## 第79题
### 题目描述
清华大学复试第23题
某个序列有n个正整数，每个正整数都是m位数。某科研人员想统计该序列各个位的“众数”。

第i（1<=i<=m）位的众数是指，n个正整数的第i位出现次数最多的最小数字。

最低位（个位）是第1位，最高位是第m位。

输入输出格式
输入描述:

从标准输入读入数据。
输入的第一行包含两个正整数n,m，保证n<=10^5, m <= 6。
输入的第二行包含n个正整数。
同行相邻两个整数用一个空格隔开。
输出描述:

输出到标准输出。
输出共m行，每行一个整数，第i行表示第i位的众数。
### 解答
```
#include<bits/stdc++.h>
using namespace std;
int main()
{
	int n,m;
	//int carry = 1;
	cin>>n>>m;
	vector<int> record(n);
	vector<int> result(10,0);
	
	for(int i = 0;i<n;i++)
		cin>>record[i];
	int max = 0;
	int index = 0;
	for(int i = 0;i<m;i++)
	{
		for(int j = 0; j < 10;j ++)
			result[j] = 0;
		for(int j = 0 ;j<n;j++)
		{
			result[record[j]%10]++;
			record[j]/=10;
		}
		max = 0;
		index = 0;
		for(int i = 0;i<10;i++)
		{
			if(result[i]>max)
			{
				max = result[i];
				index = i;
			}
		}
		cout<<index<<endl;
	}
}
```
## 第八十题
### 题目描述
某国为了防御敌国的导弹袭击，开发出一种导弹拦截系统。但是这种导弹拦截系统有一个缺陷：虽然它的第一发炮弹能够到达任意的高度，但是以后每一发炮弹都不能高于前一发的高度。某天，雷达捕捉到敌国的导弹来袭，并观测到导弹依次飞来的高度，请计算这套系统最多能拦截多少导弹。拦截来袭导弹时，必须按来袭导弹袭击的时间顺序，不允许先拦截后面的导弹，再拦截前面的导弹。 

输入输出格式
输入描述:

每组输入有两行，
第一行，输入雷达捕捉到的敌国导弹的数量k（k<=25），
第二行，输入k个正整数，表示k枚导弹的高度，按来袭导弹的袭击时间顺序给出，以空格分隔。
输出描述:

每组输出只有一行，包含一个整数，表示最多能拦截多少枚导弹。
### 解答
lis 套模版即可
```
#include<bits/stdc++.h>
using namespace std;
vector<int> a(25);
int dp[25];
int n;
int LIS_nlgn() {  
    int len = 1;  
    dp[1] = a[1];  
  
    for (int i = 2; i <= n; ++i) {  
        if (a[i] <= dp[len]) {  
            dp[++len] = a[i];  
        } else {  
            int pos = lower_bound(dp, dp + len, a[i]) - dp;  
            dp[pos] = a[i];  
        }  
    }  
    return len;  
}  
int main()
{
	cin>>n;
	for(int i = 0; i< n; i++)
	{
		cin>>a[i];
	}
	int result = LIS_nlgn();
	cout<<result-1<<endl;
}
```
## 第81题
### 题目描述
设有一个背包可以放入的物品重量为S，现有n件物品，重量分别是w1，w2，w3，...，wn。问能否从这n件物品中选择若干件放入背包中，使得放入的重量之和正好为S。如果有满足条件的选择，则此背包有解，否则此背包问题无解。

输入输出格式
输入描述:

第一行为物品重量S（整数）；  第二行为物品数量n，  第三行为n件物品的重量的序列。
输出描述:

有解就输出”yes!“，没有解就输出”no!“。
### 解答
```
#include<bits/stdc++.h>
using namespace std;
int main()
{
	int total;
	int n;
	while(cin>>total>>n)
	{
		vector<int> dp(total+1,0);
		dp[0] = 1;
		vector<int> record(n);
		for(int i = 0;i<n;i++)
		{
			cin>>record[i];
		}
		//int maxlen = 0;
		for(int i =0; i<n;i++)
		{
			for(int j = total;j>=0;j--)
			{
				if(dp[j])
				{
					if(j+record[i] <= total)
						dp[j+record[i]] = 1;
				}
			}
		}
		if(dp[total])
			cout<<"yes!"<<endl;
		else
			cout<<"no!"<<endl;
	}
}
```
## 第82题
### 题目描述
省政府“畅通工程”的目标是使全省任何两个村庄间都可以实现公路交通（但不一定有直接的公路相连，只要能间接通过公路可达即可）。现得到城镇道路统计表，表中列出了任意两城镇间修建道路的费用，以及该道路是否已经修通的状态。现请你编写程序，计算出全省畅通需要的最低成本。

输入输出格式
输入描述:

测试输入包含若干测试用例。每个测试用例的第1行给出村庄数目N ( 1< N < 100 )；随后的 N(N-1)/2 行对应村庄间道路的成本及修建状态，每行给4个正整数，分别是两个村庄的编号（从1编号到N），此两村庄间道路的成本，以及修建状态：1表示已建，0表示未建。

当N为0时输入结束。
输出描述:

每个测试用例的输出占一行，输出全省畅通需要的最低成本。
### 解答
最小生成树，kruskal
```
#include <bits/stdc++.h>  
using namespace std;  
  
const int maxn = 105;  
struct node {  
    int u, v, w;  
}edge[maxn * maxn];  
int cmp(node A, node B) {  
    return A.w < B.w;  
}  
int fa[maxn];  
int find(int x) {  
    if (x == fa[x]) return x;  
    fa[x] = find(fa[x]);  
    return fa[x];  
}  
int main(){  
    int N;
	int flag;
	
    while(cin>>N)
	{  
        if(N==0)
			break;
        for (int i = 0; i < (N*(N-1))/2; i++) 
		{  
            scanf("%d%d%d", &edge[i].u, &edge[i].v, &edge[i].w); 
			cin>>flag;
			if(flag)
				edge[i].w = 0;
        }  
        for (int i = 1; i <= N; i++) fa[i] = i;  
        sort(edge, edge + (N*(N-1))/2, cmp);  
        int sum = 0;  
        int total = 0;  
        for (int i = 0; i < (N*(N-1))/2; i++) {  
            int fx = find(edge[i].u);  
            int fy = find(edge[i].v);  
            if (fx != fy) {  
                fa[fx] = fy;  
                sum += edge[i].w;  
                total++;//统计加入边数量  
            }  
        }  
        if (total < N - 1)//不能生成树  
            printf("?\n");  
        else printf("%d\n", sum);  
    }  
    return 0;  
}  
```
## 第83题
### 题目描述
有一个特殊的 n 行 m 列的矩阵 Aij(1≤i≤n,1≤j≤m)，每个元素都是正整数，每一行和每一列都是独立的等差数列。在某一次故障中，这个矩阵的某些元素的真实值丢失了，被重置为 0。现在需要你想办法恢复这些元素，并且按照行号和列号从小到大的顺序（行号为第一关键字，列号为第二关键字，从小到大）输出能够恢复的元素。

输入输出格式
输入描述:

从标准输入读入数据。

输入的第一行包含两个正整数 n 和 m，保证 n≤10^3 和 m≤10^3。

接下来 n 行，每行 m 个整数，表示整个矩阵，保证 1≤Aij≤10^9。如果 Aij 等于 0，表示真实值丢失的元素。
输出描述:

输出若干行，表示所有能够恢复的元素。每行三个整数 i,j,x，表示 Aij 的真实值是 x。
### 解答
```
#include<bits/stdc++.h>
using namespace  std;
int matrix[1005][1005];
int patched_matrix[1005][1005];
int row_state[1005] = {0};// have how many none 0 nums
int col_state[1005] = {0};
void process_matrix(int row,int col)
{
    bool flag = true;
    while(flag)//untill cannot change
    {
        flag = false;
        for(int i =0;i<row;i++)
        {
            if(row_state[i] >1 && row_state[i] < row)//means this row can be solved
            {
                flag = true;//marked
                int first = -1;
                int second = -1;
                int first_val = -1;
                int second_val = -1;
                for(int j = 0 ;j < col; j++)
                {
                    if(matrix[i][j] || patched_matrix[i][j])
                    {
                        if(first!=-1)
                        {
                            second = j;
                            second_val = max(matrix[i][j],patched_matrix[i][j]);
                            break;
                        }
                        else
                        {
                            first = j;
                            first_val = max(matrix[i][j],patched_matrix[i][j]);
                        }
                    }
                }
                int carry = (second_val - first_val) / (second - first);
                int start = first_val - first * carry;
                //now fix it
                for(int j = 0 ;j < col; j++)
                {
                    if(matrix[i][j] == 0 && patched_matrix[i][j] == 0)
                    {
                        patched_matrix[i][j] = start;
                        col_state[j] ++;
                    }
                    start += carry;
                }
                row_state[i] = row;

            }
        }
        for(int j=0;j<col;j++)
        {
            if(col_state[j] >1 && col_state[j] < row)//means this col can be solved
            {
                flag = true;//marked
                int first = -1;
                int second = -1;
                int first_val = -1;
                int second_val = -1;
                for(int i= 0 ;i < row; i++)
                {
                    if(matrix[i][j] || patched_matrix[i][j])
                    {
                        if(first!=-1)
                        {
                            second = i;
                            second_val = max(matrix[i][j],patched_matrix[i][j]);
                            break;
                        }
                        else
                        {
                            first = i;
                            first_val = max(matrix[i][j],patched_matrix[i][j]);
                        }
                    }
                }
                int carry = (second_val - first_val) / (second - first);
                int start = first_val - first * carry;
                //now fix it
                for(int i = 0 ;i < row; i++)
                {
                    if(matrix[i][j] == 0 && patched_matrix[i][j] == 0)
                    {
                        patched_matrix[i][j] = start;
                        row_state[i] ++;
                    }
                    start += carry;
                }
                col_state[j] = col;
            }
        }
    }
}
int main()
{
    int row,col;
    scanf("%d%d",&row,&col);
    for(int i =0;i<row;i++)
    {
        for(int j = 0; j < col;j++)
        {
            scanf("%d",&matrix[i][j]);
            if(matrix[i][j])
            {
                row_state[i] ++;
                col_state[j] ++;
            }
        }
    }
    process_matrix(row,col);
    /*for(int i =0 ;i<row; i++)
    {
        for(int j = 0; j < col; j++)
        {
            cout<<patched_matrix[i][j]<<' ';
        }
        cout<<endl;
    }*/
    for(int i =0 ;i<row; i++)
    {
        for(int j = 0; j < col; j++)
        {
            if(patched_matrix[i][j])
            {
                printf("%d %d %d\n",i+1,j+1,patched_matrix[i][j]);

            }

        }
    }


}
```
## 第84题
清华大学复试不知道第几题
### 题目描述
给定一个有 n 个点，m 条边的有向图。图中第 i 个点的价值是 vi，每条边有一个代价 z，不同的边代价可能不一样。

一共有 q 个询问，每次询问包含两个数字 u,c，表示询问从 u 点出发，经过代价总和不超过 c 的边所能到达的点的价值总和的最大值。

如果一个点被多次经过，那么其价值要计算多次。初始节点的价值也要计算进去。

输入输出格式
输入描述:

从标准输入读入数据。

输入的第一行包含三个由空格隔开的正整数 n,m,q，保证 N≤2,000 和 M≤8,000,Q≤10^5。

接下来的一行包括 n 个由空格隔开的非负整数 vi 表示编号从小到大所有点的价值，保证 vi≤10^4。

接下来的 m 行每行包含三个由空格隔开的正整数 x,y,z，保证 1≤x,y≤n 和 1≤z≤30，表示存在一条从 x 到 y 代价为 z 的有向边。

接下来的 q 行每行包含两个由空格隔开的非负整数 u,c，保证 1≤u≤n 和 0≤c≤800。
输出描述:

输出到标准输出。

对于每次询问输出一个数，表示相应的答案。
### 解答
```
#include<bits/stdc++.h>
using namespace std;
#define INF 0x3f3f3f3f
int g[2005][2005];
int weight[2005];
int result;
int n, m, q;
void dfs(int start, int remain, int cur_sum)
{
    cur_sum += weight[start];
    //cout<<start<<' '<<remain<<' '<<cur_sum<<endl;
    for(int i = 0; i<=n;i++)
    {
        if(remain >= g[start][i])
            dfs(i,remain - g[start][i],cur_sum);
        else
            result = result > cur_sum ? result: cur_sum;
    }
}
int main()
{

    cin>>n>>m>>q;
    for(int i = 1;i<=n;i++)
        cin>>weight[i];
    memset(g,INF,sizeof(g));
    for(int i = 0;i<m;i++)
    {
        int x, y, w;
        cin>>x>>y>>w;
        g[x][y] = w;
    }
    int start, cost;
    for(int i = 0; i<q; i++)
    {
        //cout<<'-'<<endl;
        result = 0;
        cin>>start>>cost;
        dfs(start,cost,0);
        cout<<result<<endl;
    }

}
```
## 第85题
### 题目描述
题目描述
Time Limit: 1000 ms
Memory Limit: 256 mb
小诺被困在一个迷宫中了! 
给定一个m × n (m行, n列)的迷宫，迷宫中有两个位置，小诺想从迷宫的一个位置走到另外一个位置，当然迷宫中有些地方是空地，小诺可以穿越，有些地方是障碍，她必须绕行，从迷宫的一个位置，只能走到与它相邻的4个位置中,当然在行走过程中，小诺不能走到迷宫外面去。令人头痛的是，小诺是个没什么方向感的人，因此，她在行走过程中，不能转太多弯了，否则她会晕倒的。起点和终点也有可能为障碍，初始时，小诺所面向的方向未定，她可以选择4个方向的任何一个出发，而不算成一次转弯。小诺能从一个位置走到另外一个位置吗？ 

输入输出格式
输入描述:

第1行为一个整数t (1 ≤ t ≤ 100),表示测试数据的个数，接下来为t组测试数据，每组测试数据中，
第1行为两个整数m, n (1 ≤ m, n ≤ 100),分别表示迷宫的行数和列数，接下来m行，每行包括n个字符，其中字符'.'表示该位置为空地，字符'*'表示该位置为障碍，输入数据中只有这两种字符，每组测试数据的最后一行为5个整数k, x1, y1, x2, y2 (1 ≤ k ≤ 10, 1 ≤ x1, x2 ≤ n, 1 ≤ y1, y2 ≤ m),其中k表示PIPI最多能转的弯数，(x1, y1), (x2, y2)表示两个位置，其中x1，x2对应列，y1, y2对应行。
输出描述:

每组测试数据对应为一行，若PIPI能从一个位置走到另外一个位置，输出“yes”，否则输出“no”。
### 解答
题目数据有问题
```c
#include<bits/stdc++.h>
using namespace std;
char mapp[105][105];
bool record[105][105];
int t,m,n,k,y1_,y2,x1,x2;
bool flag;
int dir[4][2] = {0,1,0,-1,1,0,-1,0};
void dfs(int curx, int cury, int count,int curdir)
{
    if(count > k || curx < 1 || cury < 1 || curx > n || cury > m || flag || mapp[cury][curx] == '*' || record[cury][curx])
        return;
    record[cury][curx] = true;
    if(curx == x2 && cury == y2)
    {
        flag = true;
        return;
    }
    for(int i = 0; i<4; i++)
    {
        if(curdir == i)
            dfs(curx+dir[i][0],cury+dir[i][1],count,i);
        else
            dfs(curx+dir[i][0],cury+dir[i][1],count+1,i);
    }

}

int main()
{
    cin>>t;
    while(cin>>m>>n)
    {

        //cout<<t<<endl;
        flag = false;
        memset(record,false,sizeof(record));
        //cout<<m<<n<<endl;
        for(int i = 1;i<=m;i++)
            for(int j = 1; j <= n; j++)
            {
                cin>>mapp[i][j];
                //cout<<mapp[i][j]<<endl;
            }

        cin>>k>>x1>>y1_>>x2>>y2;
        //cout<<k<<x1<<y1_<<x2<<y2<<endl;
        for(int i = 0;i<4;i++)
        {
            dfs(x1+dir[i][0],y1_+dir[i][1],0,i);
        }
        //cout<<'+'<<endl;
        if(flag)
            cout<<"yes"<<'\n';
        else
            cout<<"no"<<'\n';

    }
}

```
## 第86题
### 题目描述
小诺有一个由0和1组成的字符串 
现在小诺有一次机会，可以选择一个任意的区间[L,R]，将该区间内的所有字符串进行翻转(即0->1,1->0)。 
请问小诺经过一次翻转之后字符串中最多会有多少个1？ 

输入输出格式
输入描述:

第一行输入一个正整数n，表示字符串长度，n<=10^7。 
接下来一行一个输入一个01字符串。
### 解答
如果反转，那么我们必须翻转一个0的数目多余1的数目的子串，而且是差值越大越好，那么我们将1看作-1，0看作1，求最大连续区间和，该值就是我们最多可以通过反转再得到的1的数目，加上串中本来的1的数目就是答案
```
#define ll long long
#define inf 0x3f3f3f3f
#define MAX 10000007
#define vec vector<int>
#define P pair<int,int>

int main() {
	string s; int n, dp[MAX], sum;
	while (cin >> n >> s) {
		if (n == 0) { cout << 0 << endl; continue; }
		int ma = -1, v; sum = 0;
		memset(dp, 0, sizeof(dp));
		if (s[0] == '0')dp[0] = 1, ma = 1;
		else dp[0] = -1, sum++;
		//翻转0的话：等价于将0看作1，1看作-1的最大连续区间和的问题
		for (int i = 1; i < n; i++) {
			if (s[i] == '1')sum++, v = -1;
			else v = 1;
			if (dp[i - 1] > 0)dp[i] = dp[i - 1] + v;
			else dp[i] = v;
			if (dp[i] > ma)ma = dp[i];
		}
		if (ma >= 0)cout << ma + sum << endl;
		else cout << sum << endl;
	}
}
```
## 第87题
### 题目描述
题目描述
Time Limit: 1000 ms
Memory Limit: 256 mb
多组测试用例，每组测试用例给出不超过1000课树，并且第二行给出每颗树的高度，初始时，相邻两个树的距离都相等，需要砍掉最少的树使得这些树高度呈现非递减的序列并且相邻之间距离要相等，输出最少砍的树的数组。

输入输出格式
输入描述:

输入树的棵树n，然后在接下来的一行输入n个正整数表示树的高度，整数之间以空格为间隔。
输出描述:

最少砍掉的树的颗数。
### 解答
```c
#include<bits/stdc++.h>
using namespace std;
int dp[1001][1001];
int record[1001];
int result,n;

int main()
{
	while(cin>>n)
	{
		result = 1;
		memset(dp,0,sizeof(dp));
		for(int i = 1; i <= n ; i ++)
		{
			cin>>record[i];
		}
		int tmp,sum;
		for(int i = 1; i <= n; i++)
		{
			dp[i][i] = 1;
			for(int gap = n-i;gap >0; gap --)
			{
				for(int j = i+gap; j <= n; j+= gap)
				{
					if(record[j] > record[j-gap])
					{
						dp[i][j] = max(dp[i][j],dp[i][j-gap]+1);
						result = max(result,dp[i][j]);
					}
				}
			}
		}
		cout<<n-result<<endl;
	}
}
```
## 第88题
### 题目描述
最长公共子串
### 解答
```
#include<bits/stdc++.h>
using namespace std;
int dp[101][101] = {0};

int main()
{
    string s1,s2;
    while(cin>>s1>>s2)
    {
        memset(dp,0,sizeof(dp));
        int result = 0, r = 0;
        for(int j = 0 ; j < s2.length(); j++)
        {
            if(s2[j] == s1[0])
            {
                dp[0][j] = 1;
                result = 1;
            }
        }
        for(int i = 0; i<s1.length(); i++)
        {
            if(s1[i] == s2[0])
            {
                dp[i][0] = 1;
                result = 1;
                r = i;
            }
        }

        for(int i = 1; i<s1.length() ; i++)
        {
            for(int j = 1 ; j < s2.length(); j++)
            {
                if(s1[i] == s2[j])
                {

                    dp[i][j] = dp[i-1][j-1] +1;
                    if(dp[i][j] >= result)
                    {
                        result = dp[i][j];
                        r = i;
                    }
                }
            }
        }
        cout<<result<<endl;
        cout<<s1.substr(r-result+1,result)<<endl;
    }
}
```
## 第89题
### 题目描述
题目描述
Time Limit: 1000 ms
Memory Limit: 256 mb
在一片n*m的土地上，每一块1*1的区域里都有一定数量的金子。这一天，你到这里来淘金，然而当地人告诉你，如果你挖了某一区域的金子，上一行，下一行，左边，右边的金子你都不能被允许挖了。那么问题来了：你最多能淘金多少？ 

输入输出格式
输入描述:

多组数据输入。
对于每组数据，第一行两个数n,m，表示土地的长和宽(1<=n,m<=200) 
接下来n行,每行m个数，表示每个区域的金子数量，每个区域的金子数量不超过1000
输出描述:

对于每组数据，输出最多得到的金子数量
### 解答
```
#include<bits/stdc++.h>
using namespace std;
int record[205][205];
int row_max_sum[205];
int tmp_sum[205];
int n,m,result;
int main()
{
    while(cin>>n>>m)
    {
        for(int i = 0; i< n; i++)
            for(int j = 0; j< m; j++)
                cin>>record[i][j];
        for(int i = 0; i<n ;i++)
        {
            if(m == 1)
                row_max_sum[i] = record[i][0];
            else if(m == 2)
                row_max_sum[i] = max(record[i][0],record[i][1]);
            else if(m == 3)
                row_max_sum[i] = max(record[i][0] + record[i][2], record[i][1]);
            else
            {

                tmp_sum[0] =record[i][0];
                tmp_sum[1] = record[i][1];
                tmp_sum[2] = record[i][2] + record[i][0];
                int tmp_max = max(tmp_sum[1],tmp_sum[2]);
                for(int j = 3;j<m;j++)
                {
                    tmp_sum[j] = max(tmp_sum[j-2],tmp_sum[j-3]) + record[i][j];
                    tmp_max = max(tmp_sum[j],tmp_max);
                }
                row_max_sum[i] = tmp_max;
            }
        }
        if(n == 1)
            result = row_max_sum[0];
        else if(n == 2)
            result = max(row_max_sum[0],row_max_sum[1]);
        else if(n==3)
            result = max(row_max_sum[0] + row_max_sum[2] , row_max_sum[1]);
        else
        {
            row_max_sum[2] += row_max_sum[0];
            result = max(row_max_sum[2],row_max_sum[1]);
            for(int i = 3; i< n; i++)
            {
                row_max_sum[i] = max(row_max_sum[i-2],row_max_sum[i-3]) + row_max_sum[i];
                result = max(result,row_max_sum[i]);
            }
        }
        cout<< result<<endl;
    }
}
```
## 第90题
### 题目描述
题目描述
Time Limit: 1000 ms
Memory Limit: 256 mb
给定n个零件，第i个零件所需的加工时间为ti，现分配到两台机床上加工，使得两台机器完成加工的时间尽可能同步。
例如，有5个零件，加工时间分别为3，6，2，7，8。如果把第一个和第二个零件分配给第一台机床，其余三个分配给第二台机床，则两台机床的完成时间相差|(3 + 6) - (2 + 7 +8)| = 8。而如果把第一个、第三个和第五个零件分配给第一台机床，其余零件分配给第二个机床，则两台机床的完成时间相差|(3+2+8) - (6+7)| = 0，差异最小。

输入输出格式
输入描述:

第一行输入一个数n，表示有n个零件。
第二行输入n个数，代表第i个零件所需的加工时间
1≤n≤20，0≤ti≤1e6
输出描述:

输出两台机床完工时间的最小差异。
### 解答
```
#include<bits/stdc++.h>
using namespace std;
int n;
vector<int> record(21,0);
bool dp[20000007] = {false};
bool cmp(int a, int b)
{
    return a>b;
}

int main()
{
    cin>>n;
    int sum = 0;
    for(int i = 0; i<n;i++)
    {
        cin>>record[i];
        sum += record[i];
    }
    int mid = sum >>1;
    if(sum & 1)
        mid +=1;
    dp[0] = true;
    int result = 0;
    for(int i = 0; i<n; i++)
    {
        for(int j = mid; j>=record[i]; j--)
        {
            if(dp[j - record[i]])
            {
                dp[j] = true;
                result = max(result,j);
            }
        }
    }

    cout<<abs(sum - result - result)<<endl;
}
```