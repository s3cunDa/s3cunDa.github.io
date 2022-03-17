#模版
##头文件<bits/stdc++.h>不好使怎么办
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
```
## 进制转换问题
### 反序数
输入一个整数如123，将其转换为反序之后的整数321
```
#include <stdio.h>  
  
int main() {  
    int n;  
    scanf("%d", &n);  
    int ans = 0;//将反序之后的答案存在这里  
    while (n > 0) {//将n逐位分解  
        ans *= 10;  
        ans += (n % 10);  
        n /= 10;  
    }  
    printf("%d\n", ans);  
    return 0;  
}  
```

### 十进制转x进制
```
#include <bits/stdc++.h>  
using namespace std;
int main() {  
    int origin, targetbase;  
    string s(1000);
    //输入10进制n 和 要转换的进制x  
    scanf("%d%d", &n, &x);  
    int cnt = 0;//数组下标  
    while (origin > 0) {//将n逐位分解  
        int w = (origin % targetbase);  
        if (w < 10) s[cnt++] = w + '0';//变成字符需要加'0'  
        else s[cnt++] = (w - 10) + 'A';//如果转换为小写则加'a'  
        //如果大于10则从A字符开始  
        origin /= targetbase;  
    }  
    //反序输出  
    for (int i = cnt - 1; i >= 0; i--) {  
        printf("%c", s[i]);  
    }  
    printf("\n");  
    return 0;  
}  
```
### x转y进制
```
int main() {  
    char s[105];  
    int x, y;  
    //输入二进制字符串 和 代表的进制x 以及要转换的进制y  
    scanf("%s%d%d", &s, &x, &y);  
    int ans = 0;  
    int len = strlen(s);  
    for (int i = 0; i < len; i++) {  
        ans = ans * x;  
        if (s[i] >= '0' && s[i] <= '9') ans += (s[i] - '0');  
        else ans += (s[i] - 'A') + 10;  
    }  
    char out[105];  
    int cnt = 0;  
    while (ans > 0) {  
        int w = (ans % y);  
        if (w < 10) out[cnt++] = w + '0';  
        else out[cnt++] = (w-10) + 'A';  
        ans /= y;  
    }  
    for (int i = cnt - 1; i >= 0; i--) {  
        printf("%c", out[i]);  
    }  
    printf("\n");  
    return 0;  
}  
```
### 同模余定理
(a+b)%c=(a%c+b%c)%c;
(a-b)%c=(a%c-b%c)%c; 
(a*b)%c=(a%c*b%c)%c;
### 最大公约数
```
int gcd(int a, int b) {  
    if (b == 0) return a;  
    else return gcd(b, a % b);  
}  
```
x/y = (x/gcd(x,y))/(y/gcd(x,y))
### 最小公倍数
LCM(x, y) = x * y / GCD(x, y)
x * y = LCM(x, y) * GCD(x, y)
### 线性素数筛选
```
// 线性素数筛选  prime[0]存的是素数的个数  
const int maxn = 1000000 + 5;  
int prime[maxn];  
void getPrime() {  
    memset(prime, 0, sizeof(prime));  
    for (int  i = 2; i <= maxn; i++) {  
        if (!prime[i]) prime[++prime[0]] = i;  
        for (int j = 1; j <= prime[0] && prime[j] * i <= maxn; j++) {  
            prime[prime[j] * i] = 1;  
            if (i % prime[j] == 0) break;  
        }  
    }  
}  
```
### 快速幂
解决求 (x^y) % p 问题
```
long long mod_pow(long long  x, long long y, long long mod) {  
    long long res = 1;  
    while (y > 0) {  
        //如果二进制最低位为1、则乘上x^(2^i)  
        if (y & 1) res = res * x % mod;  
        x = x * x % mod;  // 将x平方  
        y >>= 1;  
    }  
    return res;  
}  
```
### 数学公式总结
错排公式

问题： 十本不同的书放在书架上。现重新摆放，使每本书都不在原来放的位置。有几种摆法？
递推公式为：D(n) = (n - 1) * [D(n - 1) + D(n - 2)]
 

海伦公式

S = sqrt(p * (p - a) * (p - b) * (p - c))
公式描述：公式中a，b，c分别为三角形三边长，p为半周长，S为三角形的面积。
 

扇形面积

S=1/2×弧长×半径，S扇=（n/360）πR²
### 大数

```
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
```
### 哈夫曼树
可用优先级队列实现
### 深度优先搜索和剪枝
深度优先搜索模版
利用栈效率不高，递归可能好些
```
int dir[8][2] = {1,0,0,-1,-1,0,0,1,1,1,1,-1,-1,1,-1,-1};
int dfs(int x, int y) {  
    vis[x][y] = 1;  
    for (int i = 0; i < 8; i++) {//8个方向  
        int nx = x + dir[i][0];  
        int ny = y + dir[i][1];  
        if (mpt[nx][ny] == '@' && vis[nx][ny] == 0) {  
            dfs(nx, ny);  
        }  
    }  
} 
```
1.可行性剪枝

如果当前条件不合法就不再继续搜索，直接return。这是非常好理解的剪枝，搜索初学者都能轻松地掌握，而且也很好想。一般的搜索都会加上。

 

一般格式：

dfs(int x)  
{  
    if(x>n)return;  
    if(!check1(x))return;  
    ....  
    return;  
}  
 

2.最优性剪枝

如果当前条件所创造出的答案必定比之前的答案大，那么剩下的搜索就毫无必要，甚至可以剪掉。
我们利用某个函数估计出此时条件下答案的‘下界’，将它与已经推出的答案相比，如果不比当前答案小，就可以剪掉。
 

一般格式：

long long ans=987474477434487ll;  
... Dfs(int x,...)  
{  
    if(x... && ...){ans=....;return ...;}  
    if(check2(x)>=ans)return ...;    //最优性剪枝   
    for(int i=1;...;++i)  
    {  
        vis[...]=1;   
        dfs(...);  
        vis[...]=0;  
    }  
}  
//一般实现：在搜索取和最大值时，如果后面的全部取最大仍然不比当前答案大就可以返回。  
//在搜和最小时同理，可以预处理后缀最大/最小和进行快速查询。  
3.记忆化搜索

记忆化搜索其实很像动态规划（DP）。

它的关键是：如果对于相同情况下必定答案相同，就可以把这个情况的答案值存储下来，以后再次搜索到这种情况时就可以直接调用。

还有就是不能搜出环来，不能互相依赖。
 

一般格式：

long long ans=987474477434487ll;  
... Dfs(int x,...)  
{  
    if(x... && ...){ans=....;return ...;}  
    if(vis[x]!=0)return f[x];vis[x]=1;  
    for(int i=1;...;++i)  
    {  
        vis[...]=1;   
        dfs(...);  
        vis[...]=0;  
        f[x]=...;  
    }  
}  
### 最长上升子序列
序列可以不连续
```
int LIS_nlgn() {  
    int len = 1;  
    dp[1] = a[1];  
  
    for (int i = 2; i <= n; ++i) {  
        if (a[i] > dp[len]) {  
            dp[++len] = a[i];  
        } else {  
            int pos = lower_bound(dp, dp + len, a[i]) - dp;  //二分查找找到dp中最大值
            dp[pos] = a[i];  
        }  
    }  
    return len;  
}  
```
```
int LIS_nn() {  
    int ans = 0;  
    for (int i = 1; i <= n; ++i) {  
        dp[i] = 1;  
        for (int j = 1; j < i; ++j) {  
            if (a[j] < a[i]) {  //要满足上升的条件
                dp[i] = max(dp[i], dp[j] + 1);  
            }  
        }  
        ans = max(ans, dp[i]);  
    }  
    return ans;  
}  
```
### 最长公共子序列
dp[i, j]代表a字符串前i个字符组成的子串和b字符串前j个字符组成的子串的LCS。 
那么 

dp[i, j] = 0 	if i = 0 or j = 0
dp[i, j] = dp[i - 1, j - 1] + 1	if i, j > 0 and ai = bj
dp[i, j] = max{dp[i, j - 1], dp[i - 1, j]}	if i, j > 0 and ai != bj

根据上面的状态转移方程可以写出一下代码

for(int i = 1; i <= lena; ++i) {  
    for (int j = 1; j <= lenb; ++j) {  
        if(a[i - 1] == b[j - 1]) {  
            dp[i][j] = dp[i - 1][j - 1] + 1;  
        } else {  
            dp[i][j] = max(dp[i - 1][j], dp[i][j - 1]);  
        }  
    }  
}  
### 最长公共子串
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
### 01背包模版
```
// 01 背包模板  
#include <iostream>  
#include <string.h>  
using namespace std;  
  
int dp[21][1010];  
int w[21], c[21];  
  
int main() {  
    int N, V;  
    cin >> N >> V;//输入物品数量N  背包体积V  
    for (int i = 1; i <= N; ++i) {  
        cin >> w[i] >> c[i];//每个物品的重量wi 体积ci  
    }  
    //对于一个动态规划来说，最重要的是找到状态转移方程。   
    //在01背包问题中，一个物品要么装要么不装，那么我们可以得出下面的式子   
    //f[i,j]代表前i个物品背包容量最大为j最多能装的物品总重量   
    //f[i,j] = Max{ f[i-1,j-Ci]+Wi( j >= Ci ), f[i-1,j] }   
    //根据上面的状态转移方程可以写出下面的代码  
    for (int i = 1; i <= N; ++i) {  
        for (int j = 0; j <= V; ++j) {  
            if(j >= c[i]) {  
                dp[i][j] = max(dp[i - 1][j - c[i]] + w[i], dp[i - 1][j]);  
            }  
            else {  
                dp[i][j] = dp[i-1][j];  
            }  
        }  
    }  
    //dp[i][j]表示前i个物品装在j体积的背包中最大的重量  
    cout << dp[N][V] << endl;  
    return 0;  
}  
```
### 完全背包
有 N 种物品和一个容量为 V 的背包，每种物品都有无限件可用。第 i 种物品的费用是 c[i]，价 值是 w[i]。求解将哪些物品装入背包可使这些物品的费用总和不超过背包容量，且价值总和最 大。
```c
#include<bits/stdc++.h>
usingnamespacestd;
intn,W,v[10005],w[10005],dp[1005];
int main()
{
	while(cin>>n>>W)
	{
		for(int i=0;i<n;i++)
		cin>>w[i]>>v[i];
		memset(dp,0,sizeof(dp));
		for(int i=0;i<n;++i) //种类
 			for(int j=w[i];j<=W;++j) //重量从小到大枚举
	 			dp[j]=max(dp[j],dp[j-w[i]]+v[i]);
		cout<<dp[W]<<endl;
	}
	return 0;
}
```
### 拓扑
模版
```
#include <bits/stdc++.h>  
using namespace std;  
  
const int maxn = 505;  
bool mpt[maxn][maxn];  
int lev[maxn];  
vector<int> v[maxn];  
priority_queue<int, vector<int>, greater<int> > q;  
//拓扑排序  
void topo(int n) {  
    for (int i = 1; i <= n; i++) {  
        if (!lev[i]) q.push(i);  //队列里面存的是入度为0的点
    }  
    int flag = 0; //统计出队的元素个数 
    while(!q.empty()) {  
        int now = q.top();  
        q.pop();  
        if (flag) printf(" %d", now);  
        else printf("%d", now);  
        flag++;  
        for (int i = 0; i < v[now].size(); i++) {  
            int next = v[now][i];  
            lev[next]--;  
            if (!lev[next]) q.push(next);  
        }  
    }  
    if (flag != n) {  
        printf("这个图有环、并没有拓扑排序\n");  
    }  
}  
  
int main() {  
    int n, m;  
    while(scanf("%d%d", &n, &m) != EOF) {  
        memset(mpt, 0, sizeof(mpt));  
        for (int i = 1; i <= m; i++) {  
            int a, b;  
            scanf("%d%d", &a, &b);  
            mpt[a][b] = 1;  
        }  
        for (int i = 1; i <= n; i++) {  
            v[i].clear();  
            for (int j = 1; j <= n; j++) {  
                if (mpt[i][j]) {  
                    v[i].push_back(j);  
                    lev[j]++;  
                }  
            }  
        }  
        topo(n);  
        printf("\n");  
    }  
    return 0;  
}  
```
### ftt大数乘法
```
//求高精度乘法
#include<stdio.h>
#include<string.h>
#include<iostream>
#include<algorithm>
#include<math.h>
usingnamespacestd;
constdoublePI=acos(-1.0);
//复数结构体
struct complex
{
	double r,i;
	complex(double _r = 0.0,double _i = 0.0)
	{
		r = _r; i = _i;
	}
	complex operator +(const complex &b)
	{
		return complex(r+b.r,i+b.i);
	}
	complex operator -(const complex &b)
	{
		return complex(r-b.r,i-b.i);
	}
	complex operator *(const complex &b)
	{
		return complex(r*b.r-i*b.i,r*b.i+i*b.r);
	}
};
	/*
	*进行FFT和IFFT前的反转变换。
	*位置i和(i二进制反转后位置)互换
	* len必须去2的幂
	*/
void change(complex y[],int len)
{
	int i,j,k;
	for(i = 1, j = len/2;i < len-1; i++)
	{
		if(i < j)swap(y[i],y[j]);
		//交换互为小标反转的元素，i<j 保证交换一次
		//i 做正常的+1，j 左反转类型的+1,始终保持 i 和 j 是反转的
		k = len/2;
		while( j >= k)
		{
			j -= k;
			k /= 2;
		}
		if(j < k) j += k;
	}
}
	/*
	*做FFT
	*len必须为2^k形式，
	*on==1时是DFT，on==-1时是IDFT
	 */
void fft(complex y[],int len,int on)
{
	change(y,len);
	for(int h = 2; h <= len; h <<= 1)
	{
		complex wn(cos(-on*2*PI/h),sin(-on*2*PI/h));
		for(int j = 0;j < len;j+=h)
		{
			complex w(1,0);
			for(int k = j;k < j+h/2;k++)
			{
				complex u = y[k];
				complex t = w*y[k+h/2];
				y[k] = u+t;
				y[k+h/2] = u-t;
				w = w*wn;
			}
		}
	}
	if(on == -1)
	for(int i = 0;i < len;i++)
		y[i].r /= len;
}
const int MAXN = 200010;
complex x1[MAXN],x2[MAXN];
char str1[MAXN/2],str2[MAXN/2];
int sum[MAXN];
int main()
{
	while(scanf("%s%s",str1,str2)==2)
	{
		int len1 = strlen(str1);
		int len2 = strlen(str2);
		int len = 1;
		while(len < len1*2 || len < len2*2)len<<=1;
		for(int i = 0;i < len1;i++)
			x1[i] = complex(str1[len1-1-i]-'0',0);
		for(int i = len1;i < len;i++)
			x1[i] = complex(0,0);
		for(int i = 0;i < len2;i++)
			x2[i] = complex(str2[len2-1-i]-'0',0);
		for(int i = len2;i < len;i++)
			x2[i] = complex(0,0);
		//求 DFT
		fft(x1,len,1);
		fft(x2,len,1);
		for(int i = 0;i < len;i++)
			x1[i] = x1[i]*x2[i];
		fft(x1,len,-1);
		for(int i = 0;i < len;i++)
			sum[i] = (int)(x1[i].r+0.5);
		for(int i = 0;i < len;i++)
		{
			sum[i+1]+=sum[i]/10;
			sum[i]%=10;
		}
		len = len1+len2-1;
		while(sum[len] <= 0 && len > 0)len--;
		for(int i = len;i >= 0;i--)
			printf("%c",sum[i]+'0');
		printf("\n");
	}
	return 0;
}
```
### 最短路径
Floyd算法特点

1、适合求多源最短路径
2、可以求最小环
3、可以有负边权，不能有负环
4、可以求有向图的传递闭包
5、时间复杂度O(n^3)
 

单源最短路径算法的选择？

Dijkstra + 堆优化？无法处理负边权的问题
SPFA + 堆优化？期望复杂度很优秀，但是复杂度不稳定
```
/* 
spfa + vector 
SPFA可以有负边权、可以用一点个入队是否超过N次判断是否存在负环 
*/  
#include <bits/stdc++.h>  
using namespace std;  
  
#define INF 0x3f3f3f3f  
const int maxn = 105;  
int n, m;  
  
struct Edge{  
    int u, v, w;  
    Edge(int u, int v, int w):u(u),v(v),w(w) {}  
};  
  
vector<Edge> edges;  
vector<int> G[maxn];  
int dist[maxn]; // 存放起点到i点的最短距离  
int vis[maxn]; // 标记是否访问过  
int p[maxn]; // 存放路径  
  
void spfa(int s) {  
    queue<int> q;  // 如果这个spfa超时的时候可以把队列改为和dijkstra一样的优先队列
    for (int i = 0; i <= n; i++) dist[i] = INF;  
    dist[s] = 0;  
    memset(vis, 0, sizeof(vis));  
    q.push(s);  
    while (!q.empty()) {  
        int u = q.front(); q.pop();  
        vis[u] = 0;  
        for (int i = 0; i < G[u].size(); i++) {  
            Edge& e = edges[G[u][i]];  
            if (dist[e.v] > dist[u] + e.w) {  // 松弛过程
                dist[e.v] = dist[u] + e.w;  
                p[e.v] = u;  // 松弛过程 记录路径
                if (!vis[e.v]) {  
                    vis[e.v] = 1;  
                    q.push(e.v);  
                }  
            }  
        }  
    }  
}  
  
void addedge(int u, int v, int w) {  
    edges.push_back(Edge(u, v, w));  
    int sz = edges.size();  
    G[u].push_back(sz - 1);  
}  
  
void init() {  
    for(int i = 0; i <= n; i++) G[i].clear();  
    edges.clear();  
}  
  
int main() {  
    while (scanf("%d%d", &n, &m) != EOF) {  
        if (n + m == 0) break;  
        init();  
        for (int i = 0; i < m; i++) {  
            int a, b, c;  
            scanf("%d%d%d", &a, &b, &c);  
            addedge(a, b, c);  
            addedge(b, a, c);  
        }  
        spfa(1);  
        printf("%d\n",dist[n]);  
    }  
    return 0;  
}  
```
```
floyd
/* 
flody算法可以求多源最短路径 
*/  
#include <bits/stdc++.h>  
using namespace std;  
  
#define INF 0x3f3f3f3f  
const int maxn = 105;  
int mpt[maxn][maxn];  
int n, m;  
  
void floyd() {  
    for (int k = 1; k <= n; k++) {  
        for (int i = 1; i <= n; i++) {  
            for (int j = 1; j <= n; j++) {  
                mpt[i][j] = min(mpt[i][k] + mpt[k][j], mpt[i][j]);  
            }  
        }  
    }  
}  
  
int main() {  
    while (scanf("%d%d", &n, &m) != EOF) {  
        if (n + m == 0) break;  
        for (int i = 1; i <= n; i++) {  
            for (int j = 1; j <= n; j++) {  
                if (i == j) mpt[i][j] = 0;  
                else mpt[i][j] = INF;  
            }  
        }  
        for (int i = 1; i <= m; i++) {  
            int a, b, c;  
            scanf("%d%d%d", &a, &b, &c);  
            if (c < mpt[a][b]) {  //注意重边
                mpt[a][b] = c;  
                mpt[b][a] = c;  
            }  
        }  
        floyd();  
        printf("%d\n",mpt[1][n]);  
    }  
    return 0;  
}  
```
```c
// dijkstra + 堆优化  
#include <bits/stdc++.h>  
using namespace std;  
  
#define INF 0x3f3f3f3f  
const int maxn = 105;  
int n, m;  
  
struct Edge{  
    int u, v, w;  
    Edge(int u, int v, int w):u(u),v(v),w(w) {}  
};  
  
struct node {  
    int d, u;  
    node(int d, int u):d(d),u(u) {}  
    friend bool operator < (node a, node b) {  
        return a.d > b.d;  
    }  
};  
  
vector<Edge> edges;  
vector<int> G[maxn];  
int dist[maxn]; // 存放起点到i点的最短距离  
int vis[maxn]; // 标记是否访问过  
int p[maxn]; // 存放路径  
  
void dijkstra(int s) {  
    priority_queue<node> q;  
    for (int i = 0; i <= n; i++) dist[i] = INF;  
    dist[s] = 0;  
    memset(vis, 0, sizeof(vis));  
    q.push(node(0, s));  
    int cnt = 0; //统计松弛次数  
    while (!q.empty()) {  
        node now = q.top(); q.pop();  
        int u = now.u;  
        if (vis[u]) continue;  
        vis[u] = 1;  
        cnt++;  
        if(cnt >= n) break; // 小优化  
        for (int i = 0; i < G[u].size(); i++) { // // Sum -> O(E)  
            Edge& e = edges[G[u][i]];  
            if (dist[e.v] > dist[u] + e.w) { // O(lgV)  
                dist[e.v] = dist[u] + e.w;  
                p[e.v] = G[u][i];  
                q.push(node(dist[e.v], e.v));  
            }  
        }  
    }  
}  
  
void addedge(int u, int v, int w) {  
    edges.push_back(Edge(u, v, w));  
    int sz = edges.size();  
    G[u].push_back(sz - 1);  
}  
  
void init() {  
    for(int i = 0; i <= n; i++) G[i].clear();  
    edges.clear();  
}  
  
int main() {  
    while (scanf("%d%d", &n, &m) != EOF) {  
        if (n + m == 0) break;  
        init();  
        for (int i = 0; i < m; i++) {  
            int a, b, c;  
            scanf("%d%d%d", &a, &b, &c);  
            addedge(a, b, c);  
            addedge(b, a, c);  
        }  
        dijkstra(1);  
        printf("%d\n",dist[n]);  
    }  
    return 0;  
}  
```
### 最小生成树
```c
kruskal
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
    int N , M;  
    while(scanf("%d%d",&M,&N) != EOF){  
        if (M == 0) break;  
        for (int i = 0; i < M; i++) {  
            scanf("%d%d%d", &edge[i].u, &edge[i].v, &edge[i].w);  
        }  
        for (int i = 1; i <= N; i++) fa[i] = i;  
        sort(edge, edge + M, cmp);  
        int sum = 0;  
        int total = 0;  
        for (int i = 0; i < M; i++) {  
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
```c
prim
#include <bits/stdc++.h>  
using namespace std;  
#define INF 0x3f3f3f3f  
const int maxn = 105;  
int mpt[maxn][maxn];//邻接矩阵存储图  
int dist[maxn];  
int main(){  
    int N , M;  
    while(scanf("%d%d",&M,&N) != EOF) {  
        if (M == 0) break;  
        for (int i = 1; i <= N; i++) {  
            dist[i] = INF;//初始化为无穷大  
            for (int j = 1; j <= N; j++) {  
                if (i == j) mpt[i][j] = 0;  
                else mpt[i][j] = INF;  
            }  
        }  
        for(int i = 0;i < M;i++) {  
            int u, v, w;  
            scanf("%d%d%d", &u, &v, &w);  
            mpt[u][v] = min(mpt[u][v], w);//防止重边  
            mpt[v][u] = min(mpt[v][u], w);//重边用最小的  
        }  
        int sum = 0;  
        int flag = 0;  
        for (int i = 1; i <= N; i++) dist[i] = mpt[1][i];  
        for (int i = 1; i < N; i++) {  
            int min_len = INF;  
            int min_p = -1;  
            for (int j = 1; j <= N; j++) {  
                if (min_len > dist[j] && dist[j] != 0) {  
                    min_len = dist[j];  
                    min_p = j;  
                }  
            }  
            if (min_p == -1) {  
                flag = 1; break;//判断是否能生成树  
            }  
            sum += min_len;  
            for (int j = 1; j <= N; j++) {  
                if (dist[j] > mpt[min_p][j] && dist[j] != 0)  
                    dist[j] = mpt[min_p][j];  
            }  
        }  
        if (flag) printf("?\n");//不能生成树  
        else printf("%d\n", sum);  
    }  
    return 0;  
}  
```
### 快慢指针
快慢指针
类似于龟兔赛跑，两个链表上的指针从同一节点出发，其中一个指针前进速度是另一个指 针的两倍。利用快慢指针可以用来解决某些算法问题，比如:
  1、计算链表的中点:快慢指针从头节点出发，每轮迭代中，快指针向前移动两个节点，慢指 针向前移动一个节点，最终当快指针到达终点的时候，慢指针刚好在中间的节点。
2、判断链表是否有环:如果链表中存在环，则在链表上不断前进的指针会一直在环里绕圈子， 且不能知道链表是否有环。使用快慢指针，当链表中存在环时，两个指针最终会在环中相遇。
3、判断链表中环的起点:当我们判断出链表中存在环，并且知道了两个指针相遇的节点，我 们可以让其中任一个指针指向头节点，然后让它俩以相同速度前进，再次相遇时所在的节点位 置就是环开始的位置。
4、求链表中环的长度:只要相遇后一个不动，另一个前进直到相遇算一下走了多少步就好了 求链表倒数第 k 个元素:先让其中一个指针向前走 k 步，接着两个指针以同样的速度一起向前 进，直到前面的指针走到尽头了，则后面的指针即为倒数第 k 个元素。(严格来说应该叫先后 指针而非快慢指针)
### 线段树
模版
```c
#include<bits/stdc++.h>
usingnamespacestd;
#define lson x<<1 // 左孩子
#define rson (x<<1)+1 // 右孩子
const int maxn=200000+5;
int tree[maxn<<2];//定义线段树结点
int arr[maxn];//输入的数据
int ans;//答案
int n, m;
//向上合并
void Push_Up(int x) //更新节点的值，这里是求和，也可以改为最大或者最小值
{
	tree[x] = tree[lson] + tree[rson];//求和
	/*
	最小值 tree[x] = min(tree[lson],tree[rson]);
	最大值同理
	*/
}
 //创建一颗线段树
void Create(int x, int l, int r) 
{
	if (l == r) 
	{
		tree[x] = arr[l];
		return;
	}
	int mid = (l + r) / 2;
	Create(lson, l, mid);
	Create(rson, mid + 1, r);
	Push_Up(x);
}
//将 pos 点的值更新为 val
void Update(int x, int l, int r, int pos, int val) //x为当前节点，当然每次更新全树都要更新，所以调用update（1，1，n，pos，val）
{
	if (l >= r) 
	{
		tree[x] = val;
		return;
	}
	int mid = (l + r) / 2;
	if (pos <= mid) Update(lson, l, mid, pos, val);
	if (pos > mid) Update(rson, mid + 1, r, pos, val);
	Push_Up(x);
}
//将 pos 点的值增加 val
void Add(int x, int l, int r, int pos, int val) 
{
	if (l >= r) 
	{
		tree[x] += val;
		return;
	}
	int mid = (l + r) / 2;
	if (pos <= mid) Add(lson, l, mid, pos, val);
	if (pos > mid) Add(rson, mid + 1, r, pos, val);
	Push_Up(x);
}
//查询区间[L, R]之间所有值的累加和
void Query(int x, int l, int r, int L, int R) 
{  
	if (L <= l && R >= r) 
	{
		ans += tree[x];
		return;
	}
	int mid = (l + r) / 2;
	if (L <= mid) Query(lson, l, mid, L, R);
	if (R > mid) Query(rson, mid + 1, r, L, R);
}
int main() 
{
	while (scanf("%d%d", &n, &m) != EOF) 
	{
		for (int i = 1; i <= n; i++) scanf("%d", &arr[i]);
		Create(1, 1, n);
		while (m--) 
		{
			ans = 0;
			char ch[10];
			int a, b;
			scanf("%s", ch);
			scanf("%d%d", &a, &b);
			if (ch[0] == 'U') 
			{//只判断第一个字符速度更快
				Update(1, 1, n, a, b);
			}
			else if (ch[0] == 'A') {
				Add(1, 1, n, a, b);
			}
			else if (ch[0] == 'S') {
				Add(1, 1, n, a, -b);//减去一个数就是加上相反数
			}
			else {
				Query(1, 1, n, a, b);
				printf("%d\n", ans);
			}
		}
	}
	return 0;
}
```
适用于区间更新版本，加入懒惰标识
```c
#include<bits/stdc++.h>
using namespace std;
#definelsonx<<1
#definerson(x<<1)+1
typedef longlong ll;
const int maxn=200000+5;
ll tree[maxn<<2];//定义线段树结点
lll azy[maxn<<2];//懒惰标记
int arr[maxn];//输入的数据
ll ans;//答案
int n, m;
//向上合并
void Push_Up(int x) 
{
	tree[x] = tree[lson] + tree[rson];
}
//向下传递 lazy 标记
void Push_Down(int x, int l, int r) 
{
	int mid = (l + r) / 2;
	if(lazy[x])
	{//若有标记,则将标记向下移动一层
		lazy[lson] += lazy[x];
		lazy[rson] += lazy[x];
		tree[lson] += (mid - l + 1) * lazy[x];
		tree[rson] += (r - mid) * lazy[x];
		lazy[x] = 0;//取消本层标记
	}
}
//创建一颗线段树
void Create(int x, int l, int r) 
{
	lazy[x] = 0;
	if (l == r) 
	{
		tree[x] = arr[l];
		return;
	}
	int mid = (l + r) / 2;
	Create(lson, l, mid);
	Create(rson, mid + 1, r);
	Push_Up(x);
}
//将 pos 点的值增加 val
 void Add(int x, int l, int r, int L, int R, int val) 
 {
	if (L <= l && R >= r) {//找到要加的区间
		tree[x] += (r - l + 1) * val;
		lazy[x] += val;
		return;
	}
	Push_Down(x, l, r);//向下更新
	int mid = (l + r) / 2;
	if (L <= mid) Add(lson, l, mid, L, R, val);
	if (R > mid) Add(rson, mid + 1, r, L, R, val);
	Push_Up(x);
}
//查询区间[L, R]之间所有值的累加和
void Query(int x, int l, int r, int L, int R) 
{
	if (L <= l && R >= r) 
	{//找到查询的区间
		ans += tree[x];
		return;
	}
	Push_Down(x, l, r);//向下更新
	int mid = (l + r) / 2;
	if (L <= mid) Query(lson, l, mid, L, R);
	if (R > mid) Query(rson, mid + 1, r, L, R);
}
int main() {
	while (scanf("%d%d", &n, &m) != EOF) 
	{
		for (int i = 1; i <= n; i++) scanf("%d", &arr[i]);
		Create(1, 1, n);
 		while (m--) 
 		{
			ans = 0;
			char ch[10];
			int a, b, x;
			scanf("%s", ch);
			if (ch[0] == 'A') 
			{
				scanf("%d%d%d", &a, &b, &x);
				Add(1, 1, n, a, b, x);
			}
			else if (ch[0] == 'S') 
			{
				scanf("%d%d%d", &a, &b, &x);
				Add(1, 1, n, a, b, -x);//减去一个数就是加上相反数
			}
			else 
			{
				scanf("%d%d", &a, &b);
				Query(1, 1, n, a, b);
				printf("%lld\n", ans);
			}
		}
	}
	return 0;
}
```
### kmp
```c
#include<bits/stdc++.h>
using namespace std;
#defineMAXN1000010
int _next[MAXN];//注意不要用next
char txt[MAXN];//文本串
char str[MAXN];//模式串
void Get_Next(intstr_len)
{
	int j = 0;
	for (int i = 2; i <= str_len; i++) 
	{
	//此处判断 j 是否为 0 的原因在于，如果回跳到第一个字符就不用再回跳了
		while(j && str[i] != str[j+1])
		j = _next[j];//通过自己匹配自己来得出每一个点的 next 值
		if(str[j+1] == str[i]) j++;
		_next[i] = j;//i+1 失配后应该如何跳
	}
}
void Kmp(int str_len, int txt_len) 
{
	Get_Next(str_len);
	int j = 0;//j 可以看做表示当前已经匹配完的模式串的最后一位的位置
	//如果楼上看不懂，你也可以理解为 j 表示模式串匹配到第几位了
	for(int i = 1; i <= txt_len; i++) {
		while(j > 0 && str[j+1] != txt[i])
		j = _next[j];
		//如果失配 ，那么就不断向回跳，直到可以继续匹配
		if (str[j+1] == txt[i]) j++;
		if (j == str_len) 
		{//如果匹配成功，那么对应的模式串位置++
			printf("%d\n", i-str_len+1);
			j = _next[j];//继续匹配
		}
	}
}
int main() {
	scanf("%s", txt + 1);
	scanf("%s", str + 1);
	int str_len = strlen(str+1);
	int txt_len = strlen(txt+1);
	Kmp(str_len, txt_len);
	for (int i = 1; i <= str_len; i++)
		printf("%d ", _next[i]);
	printf("\n");
	return 0;
}
 
```
由kmp判断一个字符串是否由多个重复子串组成
```c
void getNext(int *next,const string &s)
    {
        int j=-1;
        next[0]=j;
        for(int i=1;i<s.size();i++)
        {
            while(j>=0&&s[i]!=s[j+1])
            {
                j=next[j];
            }
            if(s[i]==s[j+1])
            {
                j++;
            }
            next[i]=j;
        }
    }
    bool repeatedSubstringPattern(string s) {
        if(s.size()==0)
        {
            return false;
        }
        int next[s.size()];
        getNext(next,s);
        int len=s.size();
        //最长相等前后缀的长度为：next[len-1]+1
        //数组长度: len
        //如果len%(len-(next[len-1]+1))==0,说明数组的长度整除，说明该字符串有重复的子字符串
        if(next[len-1]!=-1&&len%(len-(next[len-1]+1))==0)
        {
            return true;
        }
        return false;
    }

```
求一个字符串最小循环节
```c

void getnext(int len){      
    int i=0,j=-1;
    next[0]=-1;
    while(i<len){
        if(j==-1 || str[i]==str[j]){
            i++;j++;
            next[i]=j;
        }else
            j=next[j];
    }
}
如果对于next数组中的 i， 符合 i % ( i - next[i] ) == 0 && next[i] != 0 , 则说明字符串循环，而且

循环节长度为:   i - next[i]

循环次数为:       i / ( i - next[i] )
```
判断二叉树a是否为二叉树b子树
```c
问题是给定两个二叉树，求二叉树是否是另一个二叉树的子树。

      一种可行的思路是：

                把二叉树序列化为字符串，在比较两个字符串，如果字符串在另一个字符串中匹配了，那么说明这个二叉树是另一个二叉树的子树。两个字符串比较用的kmp算法。

               步骤：

                       1.把二叉树根据前序遍历或者后序遍历，中序遍历，把二叉树序列化为字符串，二叉树遍历又分为递归和非递归实现，这里不做过多的实现介绍。后序有学到会另外的博客介绍，也可以去网上搜索相关实现介绍

                       2，把二叉树序列化后，成为两个字符串，在用KMP匹配算法，KMP算法也不做过多的介绍。我前面的博客已经有写过一篇介绍。

```
### k短路问题
求第k短的路径
```c
#include<bits/stdc++.h>
using namespace std;
#define INF 0x3f3f3f3f
const int maxn=1005;
int n,m;
int dist[maxn];//存放起点到i点的最短距离
int vis[maxn];//标记是否访问过
int p[maxn];//存放路径
struct Edge
{
	int u, v, w;
	Edge(int u, int v, int w):u(u),v(v),w(w) {}
};
struct node {
	int v;
	int g, f;
	node(int v, int g, int f):v(v),g(g),f(f) {}
	bool operator < (const node &t) const 
	{
		if (t.f == f) 
			return t.g < g;
		return t.f < f;
	}
};
vector<Edge> edges;
vector<Edge> revedges;
vector<int> G[maxn];
vector<int> RG[maxn];
queue<int> q;
void spfa(int s) 
{
	while (!q.empty()) q.pop();
	for (int i = 0; i <= n; i++) dist[i] = INF;
	dist[s] = 0;
	memset(vis, 0, sizeof(vis));
	q.push(s);
	while (!q.empty()) 
	{
		int u = q.front(); q.pop();
		vis[u] = 0;
		for (int i = 0; i < G[u].size(); i++) 
		{
			Edge& e = edges[G[u][i]];
			if (dist[e.v] > dist[u] + e.w) 
			{
				dist[e.v] = dist[u] + e.w;
				p[e.v] = G[u][i];
				if (!vis[e.v]) 
				{
					vis[e.v] = 1;
					q.push(e.v);
				}
			}
		}
	}
}
priority_queue<node> que;
int A_Star(int s, int t, int k) 
{
	while (!que.empty()) que.pop();
	que.push(node(s, 0, dist[s]));
	while (!que.empty()) 
	{
		node now = que.top();
		que.pop();
		if (now.v == t) 
		{
			if (k > 1) k--;
			else return now.g;
		}
		int u = now.v;
		for (int i = 0; i < RG[u].size(); i++) 
		{
			Edge& e = edges[RG[u][i]];
			que.push(node(e.u, now.g + e.w, now.g + e.w + dist[e.u]));
		}
	}
	return -1;
}
void addedge(int u, int v, int w) 
{
	edges.push_back(Edge(u, v, w));
	int sz = edges.size();
	G[u].push_back(sz - 1);
}
void addrevedge(int u, int v, int w) 
{
	revedges.push_back(Edge(u, v, w));
	int sz = revedges.size();
	RG[u].push_back(sz - 1);
}
void init() 
{
	for(int i = 0; i <= n; i++) {G[i].clear(); RG[i].clear();}
	edges.clear();
	revedges.clear();
}
int main() 
{
	while (scanf("%d%d", &n, &m) != EOF) 
	{
		init();
		for (int i = 0; i < m; i++) 
		{
			int a, b, c;
			scanf("%d%d%d", &a, &b, &c);
			addrevedge(a, b, c);
			addedge(b, a, c);
		}
		int s, t, k;
		scanf("%d%d%d", &s, &t, &k);//起点、终点、第 K 短
		if (s == t) k++;
		spfa(t);
		int ans = A_Star(s, t, k);
		printf("%d\n", ans);
	}
	return 0;
}
```
求最小环
```c
//floyd求最小环
int Floyd_MinCircle()
{
	int Mincircle = Mod;
	int i, j, k;
	for (k = 1; k <= n; k++) 
	{
		for (i = 1; i <= n; i++) 
		{
			for (j = 1; j <= n; j++) 
			{
				if (dis[i][j] != Mod && mp[j][k] != Mod && mp[k][i] != Mod && dis[i][j] + m p[j][k] + mp[k][i] < Mincircle)
					Mincircle = dis[i][j] + mp[j][k] + mp[k][i];
			}
		}
//正常 Floyd
		for (i = 1; i <= n; i++) 
		{
			for (j = 1; j <= n; j++) 
			{
				if (dis[i][k] != Mod && dis[k][j] != Mod && dis[i][k] + dis[k][j] < dis[i][j]) 
				{
					dis[i][j] = dis[i][k] + dis[k][j];
					pre[i][j] = pre[k][j];
				}
			}
		}
	}
	return Mincircle;
}
```
### 高精度模版
```c
#include <cstdio>
#include <iostream>
#include <cmath>
#include <string>
#include <cstring>
#include <vector>
#include <algorithm>
using namespace std;
const double PI = acos(-1.0);
struct Complex{
    double x,y;
    Complex(double _x = 0.0,double _y = 0.0){
        x = _x;
        y = _y;
    }
    Complex operator-(const Complex &b)const{
        return Complex(x - b.x,y - b.y);
    }
    Complex operator+(const Complex &b)const{
        return Complex(x + b.x,y + b.y);
    }
    Complex operator*(const Complex &b)const{
        return Complex(x*b.x - y*b.y,x*b.y + y*b.x);
    }
};
void change(Complex y[],int len){
    int i,j,k;
    for(int i = 1,j = len/2;i<len-1;i++){
        if(i < j)    swap(y[i],y[j]);
        k = len/2;
        while(j >= k){
            j = j - k;
            k = k/2;
        }
        if(j < k)    j+=k;
    }
}
void fft(Complex y[],int len,int on){
    change(y,len);
    for(int h = 2;h <= len;h<<=1){
        Complex wn(cos(on*2*PI/h),sin(on*2*PI/h));
        for(int j = 0;j < len;j += h){
            Complex w(1,0);
            for(int k = j;k < j + h/2;k++){
                Complex u = y[k];
                Complex t = w*y[k + h/2];
                y[k] = u + t;
                y[k + h/2] = u - t;
                w = w*wn;
            }
        }
    }
    if(on == -1){
        for(int i = 0;i < len;i++){
            y[i].x /= len;
        }
    }
}
class BigInt
{
#define Value(x, nega) ((nega) ? -(x) : (x))
#define At(vec, index) ((index) < vec.size() ? vec[(index)] : 0)
    static int absComp(const BigInt &lhs, const BigInt &rhs)
    {
        if (lhs.size() != rhs.size())
            return lhs.size() < rhs.size() ? -1 : 1;
        for (int i = lhs.size() - 1; i >= 0; --i)
            if (lhs[i] != rhs[i])
                return lhs[i] < rhs[i] ? -1 : 1;
        return 0;
    }
    using Long = long long;
    const static int Exp = 9;
    const static Long Mod = 1000000000;
    mutable std::vector<Long> val;
    mutable bool nega = false;
    void trim() const
    {
        while (val.size() && val.back() == 0)
            val.pop_back();
        if (val.empty())
            nega = false;
    }
    int size() const { return val.size(); }
    Long &operator[](int index) const { return val[index]; }
    Long &back() const { return val.back(); }
    BigInt(int size, bool nega) : val(size), nega(nega) {}
    BigInt(const std::vector<Long> &val, bool nega) : val(val), nega(nega) {}

public:
    friend std::ostream &operator<<(std::ostream &os, const BigInt &n)
    {
        if (n.size())
        {
            if (n.nega)
                putchar('-');
            for (int i = n.size() - 1; i >= 0; --i)
            {
                if (i == n.size() - 1)
                    printf("%lld", n[i]);
                else
                    printf("%0*lld", n.Exp, n[i]);
            }
        }
        else
            putchar('0');
        return os;
    }
    friend BigInt operator+(const BigInt &lhs, const BigInt &rhs)
    {
        BigInt ret(lhs);
        return ret += rhs;
    }
    friend BigInt operator-(const BigInt &lhs, const BigInt &rhs)
    {
        BigInt ret(lhs);
        return ret -= rhs;
    }
    BigInt(Long x = 0)
    {
        if (x < 0)
            x = -x, nega = true;
        while (x >= Mod)
            val.push_back(x % Mod), x /= Mod;
        if (x)
            val.push_back(x);
    }
    BigInt(const char *s)
    {
        int bound = 0, pos;
        if (s[0] == '-')
            nega = true, bound = 1;
        Long cur = 0, pow = 1;
        for (pos = strlen(s) - 1; pos >= Exp + bound - 1; pos -= Exp, val.push_back(cur), cur = 0, pow = 1)
            for (int i = pos; i > pos - Exp; --i)
                cur += (s[i] - '0') * pow, pow *= 10;
        for (cur = 0, pow = 1; pos >= bound; --pos)
            cur += (s[pos] - '0') * pow, pow *= 10;
        if (cur)
            val.push_back(cur);
    }
    BigInt &operator=(const char *s){
        BigInt n(s);
        *this = n;
        return n;
    }
    BigInt &operator=(const Long x){
        BigInt n(x);
        *this = n;
        return n;
    }
    friend std::istream &operator>>(std::istream &is, BigInt &n){
        string s;
        is >> s;
        n=(char*)s.data();
        return is;
    }
    BigInt &operator+=(const BigInt &rhs)
    {
        const int cap = std::max(size(), rhs.size()) + 1;
        val.resize(cap);
        int carry = 0;
        for (int i = 0; i < cap - 1; ++i)
        {
            val[i] = Value(val[i], nega) + Value(At(rhs, i), rhs.nega) + carry, carry = 0;
            if (val[i] >= Mod)
                val[i] -= Mod, carry = 1;
            else if (val[i] < 0)
                val[i] += Mod, carry = -1;
        }
        if ((val.back() = carry) == -1) //assert(val.back() == 1 or 0 or -1)
        {
            nega = true, val.pop_back();
            bool tailZero = true;
            for (int i = 0; i < cap - 1; ++i)
            {
                if (tailZero && val[i])
                    val[i] = Mod - val[i], tailZero = false;
                else
                    val[i] = Mod - 1 - val[i];
            }
        }
        trim();
        return *this;
    }
    friend BigInt operator-(const BigInt &rhs)
    {
        BigInt ret(rhs);
        ret.nega ^= 1;
        return ret;
    }
    BigInt &operator-=(const BigInt &rhs)
    {
        rhs.nega ^= 1;
        *this += rhs;
        rhs.nega ^= 1;
        return *this;
    }
    friend BigInt operator*(const BigInt &lhs, const BigInt &rhs)
    {
        int len=1;
        BigInt ll=lhs,rr=rhs;
        ll.nega = lhs.nega ^ rhs.nega;
        while(len<2*lhs.size()||len<2*rhs.size())len<<=1;
        ll.val.resize(len),rr.val.resize(len);
        Complex x1[len],x2[len];
        for(int i=0;i<len;i++){
            Complex nx(ll[i],0.0),ny(rr[i],0.0);
            x1[i]=nx;
            x2[i]=ny;
        }
        fft(x1,len,1);
        fft(x2,len,1);
        for(int i = 0 ; i < len; i++)
            x1[i] = x1[i] * x2[i];
        fft( x1 , len , -1 );
        for(int i = 0 ; i < len; i++)
            ll[i] = int( x1[i].x + 0.5 );
        for(int i = 0 ; i < len; i++){
            ll[i+1]+=ll[i]/Mod;
            ll[i]%=Mod;
        }
        ll.trim();
        return ll;
    }
    friend BigInt operator*(const BigInt &lhs, const Long &x){
        BigInt ret=lhs;
        bool negat = ( x < 0 );
        Long xx = (negat) ? -x : x;
        ret.nega ^= negat;
        ret.val.push_back(0);
        ret.val.push_back(0);
        for(int i = 0; i < ret.size(); i++)
            ret[i]*=xx;
        for(int i = 0; i < ret.size(); i++){
            ret[i+1]+=ret[i]/Mod;
            ret[i] %= Mod;
        }
        ret.trim();
        return ret;
    }
    BigInt &operator*=(const BigInt &rhs) { return *this = *this * rhs; }
    BigInt &operator*=(const Long &x) { return *this = *this * x; }
    friend BigInt operator/(const BigInt &lhs, const BigInt &rhs)
    {
        static std::vector<BigInt> powTwo{BigInt(1)};
        static std::vector<BigInt> estimate;
        estimate.clear();
        if (absComp(lhs, rhs) < 0)
            return BigInt();
        BigInt cur = rhs;
        int cmp;
        while ((cmp = absComp(cur, lhs)) <= 0)
        {
            estimate.push_back(cur), cur += cur;
            if (estimate.size() >= powTwo.size())
                powTwo.push_back(powTwo.back() + powTwo.back());
        }
        if (cmp == 0)
            return BigInt(powTwo.back().val, lhs.nega ^ rhs.nega);
        BigInt ret = powTwo[estimate.size() - 1];
        cur = estimate[estimate.size() - 1];
        for (int i = estimate.size() - 1; i >= 0 && cmp != 0; --i)
            if ((cmp = absComp(cur + estimate[i], lhs)) <= 0)
                cur += estimate[i], ret += powTwo[i];
        ret.nega = lhs.nega ^ rhs.nega;
        return ret;
    }
    friend BigInt operator/(const BigInt &num,const Long &x){
        bool negat = ( x < 0 );
        Long xx = (negat) ? -x : x;
        BigInt ret;
        Long k = 0;
        ret.val.resize( num.size() );
        ret.nega = (num.nega ^ negat);
        for(int i = num.size() - 1 ;i >= 0; i--){
            ret[i] = ( k * Mod + num[i]) / xx;
            k = ( k * Mod + num[i]) % xx;
        }
        ret.trim();
        return ret;
    }
    bool operator==(const BigInt &rhs) const
    {
        return nega == rhs.nega && val == rhs.val;
    }
    bool operator!=(const BigInt &rhs) const { return nega != rhs.nega || val != rhs.val; }
    bool operator>=(const BigInt &rhs) const { return !(*this < rhs); }
    bool operator>(const BigInt &rhs) const { return !(*this <= rhs); }
    bool operator<=(const BigInt &rhs) const
    {
        if (nega && !rhs.nega)
            return true;
        if (!nega && rhs.nega)
            return false;
        int cmp = absComp(*this, rhs);
        return nega ? cmp >= 0 : cmp <= 0;
    }
    bool operator<(const BigInt &rhs) const
    {
        if (nega && !rhs.nega)
            return true;
        if (!nega && rhs.nega)
            return false;
        return (absComp(*this, rhs) < 0) ^ nega;
    }
    void swap(const BigInt &rhs) const
    {
        std::swap(val, rhs.val);
        std::swap(nega, rhs.nega);
    }
};
BigInt ba,bb;
int main(){
    cin>>ba>>bb;
    cout << ba + bb << '\n';//和
    cout << ba - bb << '\n';//差
    cout << ba * bb << '\n';//积
    BigInt d;
    cout << (d = ba / bb) << '\n';//商
    cout << ba - d * bb << '\n';//余
    return 0;
}
```
### 数位dp
模版
```c
typedef long long ll;
int a[20];
ll dp[20][state];//不同题目状态不同
ll dfs(intpos,/*state变量*/,boollead/*前导零*/,boollimit/*数位上界变量*/)// 断前导零 不是每个题都要判
{
	//递归边界，既然是按位枚举，最低位是 0，那么 pos==-1 说明这个数我枚举完了
	//也就是说当前枚举到 pos 位，一定要保证前面已经枚举的数位是合法的。不过具体题目不同或者写法不同的 话不一定要返回 1
	if(pos==0) return 1;
	//第二个就是记忆化(在此前可能不同题目还能有一些剪枝)
	if(!limit && !lead && dp[pos][state]!=-1) return dp[pos][state];

	//常规写法都是在没有限制的条件记忆化，这里与下面记录状态是对应，具体为什么是有条件的记忆化后面会讲
	int up=limit?a[pos]:9;//根据 limit 判断枚举的上界 up;
	ll ans=0;
	//开始计数
	for(int i=0;i<=up;i++)//枚举，然后把不同情况的个数加到 ans 就可以了
	{
		if() ...
		else if()...

		ans+=dfs(pos-1,/*状态转移*/,lead && i==0,limit && i==a[pos]) //最后两个变量传参都是这样写的
 		/*这里还算比较灵活，不过做几个题就觉得这里也是套路了
		大概就是说，我当前数位枚举的数是 i，然后根据题目的约束条件分类讨论
		去计算不同情况下的个数，还有要根据 state 变量来保证 i 的合法性，比如题目
		要求数位上不能有 62 连续出现,那么就是 state 就是要保存前一位 pre,然后分类，
		前一位如果是 6 那么这意味就不能是 2，这里一定要保存枚举的这个数是合法*/
	}
	//计算完，记录状态
	if(!limit && !lead) dp[pos][state]=ans;
	/*这里对应上面的记忆化，在一定条件下时记录，保证一致性，当然如果约束条件不需要考虑 lead 是 lead 就完全不用考虑了*/
	return ans;

}
ll solve(ll x)
{
	int pos=0;
	while(x)//把数位都分解出来
	{
		a[++pos]=x%10;
		x/=10;
	}
	return dfs(pos-1/*从最高位开始枚举*/,/*一系列状态 */,true,true);// 且有前导零的，显然比最高位还要高的一位视为 0 嘛
}

int main()
{
	ll le,ri;
	while(~scanf("%lld%lld",&le,&ri))
	{ //初始化 dp 数组为-1,计算[le,ri]，等价于[0,ri] - [0,le-1]
		printf("%lld\n",solve(ri)-solve(le-1));
	}
	return 0;
}
 
```
### 并查集
```c
#include<bits/stdc++.h>
using namespace std;

int const MAXN=10001;

int father[MAXN];
int height[MAXN];

void Initial(){
    for(int i=0;i<MAXN;i++){
        father[i]=i;
        height[i]=0;
    }
}

int Find(int x){
    if(x!=father[x]){
        father[x]=Find(father[x]);
    }
    return father[x];
}

void Uinion(int x,int y){
    x=Find(x);
    y=Find(y);
    if(x!=y){
        if(height[x]<height[y]){
            father[x]=y;
        }else if(height[x]>height[y]){
            father[y]=x;
        }else{
            father[y]=x;
            height[x]++;
        }
    }
}


int main(){
    int n,m;
    while(cin>>n>>m){
        Initial();
        while(m--){
            int z,x,y;
            cin>>z>>x>>y;
            if(z==1){
                Uinion(x,y);
            }else{
                if(Find(x)==Find(y)){
                    cout<<"Y"<<endl;
                }else{
                    cout<<"N"<<endl;
                }
            }
        }
    }
}
```
