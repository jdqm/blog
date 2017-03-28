---
title: 第六届蓝桥杯Java B组省赛试题答案
date: 2015-03-27 20:53:10
tags:
	-算法
	-数据结构
categories: 数据结构与算法
---

1、三角形面积

如【图1】所示。图中的所有小方格面积都是1。
那么，图中的三角形面积应该是多少呢？

请填写三角形的面积。不要填写任何多余内容或说明性文字。

![第一题图片][1]

这题为不规则图形求面积的问题，通常的解题思路是在不规则图形中寻找规则图形：
①整体法：整体视为一个规则图形，减去局部的规则图形
②局部法：将不规则图形分解为多个规则图形求解
所有方格为1，说明每个方格的长度为1。
8*8-(8*4+6*4+8*2)/2=28

答案：28

<!-- more -->

2、立方变自身

观察下面的现象,某个数字的立方，按位累加仍然等于自身。
1^3 = 1 
8^3  = 512    5+1+2=8
17^3 = 4913   4+9+1+3=17
...

请你计算包括1,8,17在内，符合这个性质的正整数一共有多少个？

请填写该数字，不要填写任何多余的内容或说明性的文字。


```
package lanqiao;

public class Two {

	public static void main(String[] args) {
		int count = 0;
		// 100*100*100 = 1000000(7位)，即使7位全部为9，也不可能，大于100的就更不可能了
		for (int i = 1; i < 100; i++) {
			int result = i * i * i;
			char[] array = String.valueOf(result).toCharArray();
			int sum = 0;
			StringBuilder builder = new StringBuilder();
			for (int j = 0; j < array.length; j++) {
				sum += Integer.valueOf(array[j] - 48);
				if (j != 0) {
					builder.append("+");
				}
				builder.append(array[j]);
			}
			if (i == sum) {
				count++;
				System.out.println(i + "^3=" + result + "," + i + "=" + builder.toString());
			}
		}
		System.out.println("count=" + count);
	}
}
```
1^3=1,1=1
8^3=512,8=5+1+2
17^3=4913,17=4+9+1+3
18^3=5832,18=5+8+3+2
26^3=17576,26=1+7+5+7+6
27^3=19683,27=1+9+6+8+3

答案：6

3、三羊献瑞

观察下面的加法算式
![第三题图片][2]
其中，相同的汉字代表相同的数字，不同的汉字代表不同的数字。
请你填写“三羊献瑞”所代表的4位数字（答案唯一），不要填写任何多余内容。

```
package lanqiao;

public class Three {
	public static void main(String[] args) {
		//两个4位数且各位不相同则为 1023~9876
		for(int i = 1023; i <= 9876; i++){
			for(int j = 1023; j <= 9876; j++) {
				int sum = i+j;
				//和为5位数，且各位都不相同10234~97654
				if(sum > 10234 && sum <= 97654) {
					char[] a1 = String.valueOf(i).toCharArray();
					char[] a2 = String.valueOf(j).toCharArray();
					char[] a3 = String.valueOf(sum).toCharArray();
					//三个数相同的数字位
					if(a1[1] == a2[3] && a1[1]==a3[3] && a2[3] == a3[3] &&a1[2]==a3[2] &&a2[0]==a3[0]&&a2[1]==a3[1]){
						//第一个数各自的每一位都不相同
						if(a1[0]!=a1[1]&& a1[0]!=a1[2]&& a1[0]!=a1[3]&& a1[1]!=a1[2]&& a1[1]!=a1[3]&& a1[2]!=a1[3]){
							//第二个数各自的每一位都不相同
							if(a2[0]!=a2[1]&& a2[0]!=a2[2]&& a2[0]!=a2[3]&& a2[1]!=a2[2]&& a2[1]!=a2[3]&& a2[2]!=a2[3]){
								//第三个数各自的每一位都不相同
								if(a3[0]!=a3[1]&&a3[0]!=a3[2]&&a3[0]!=a3[3]&&a3[0]!=a3[4]&&a3[1]!=a3[2]&&a3[1]!=a3[3]&&a3[1]!=a3[4]&&a3[2]!=a3[3]&&a3[2]!=a3[4]&&a3[3]!=a3[4]){
									//第一个数的第0位和第3位其他两个数都不包含
									if(!String.valueOf(j).contains(String.valueOf(i).substring(0,1)) && !String.valueOf(j).contains(String.valueOf(i).substring(3,4))){
										//第二个数的第2位与第三个数的第二位不相同
										if(a2[2] != a3[2]){
											System.out.println(" " + i);
											System.out.println(" " + j);
											System.out.println(sum);
										}					
									}					
								}
							}
						}
					}
				}
			}
		}
	}
}
```

打印结果

 9567
 1085
10652

答案：1085


4、循环节长度

两个整数做除法，有时会产生循环小数，其循环部分称为：循环节。
比如，11/13=6=>0.846153846153.....  其循环节为[846153] 共有6位。
下面的方法，可以求出循环节的长度。

请仔细阅读代码，并填写划线部分缺少的代码。
```
	public static int f(int n, int m)
	{
		n = n % m;	
		Vector v = new Vector();
		
		for(;;)
		{
			v.add(n);
			n *= 10;
			n = n % m;
			if(n==0) return 0;
			if(v.indexOf(n)>=0)  _________________________________ ;  //填空
		}
	}
```
注意，只能填写缺少的部分，不要重复抄写已有代码。不要填写任何多余的文字。

解题思路：将每次求余的到余数存在Vector中，如果有发现有重复并且不是第0位（第0位有是整数位），这个题目比较简单。

答案：
```
v.size()-v.indexOf(n);
```


5、九数组分数

1,2,3...9 这九个数字组成一个分数，其值恰好为1/3，如何组法？

下面的程序实现了该功能，请填写划线部分缺失的代码。
```
public class A
{
	public static void test(int[] x)
	{
		int a = x[0]*1000 + x[1]*100 + x[2]*10 + x[3];
		int b = x[4]*10000 + x[5]*1000 + x[6]*100 + x[7]*10 + x[8];		
		if(a*3==b) System.out.println(a + " " + b);
	}
	
	public static void f(int[] x, int k)
	{
		if(k>=x.length){
			test(x);
			return;
		}
		
		for(int i=k; i<x.length; i++){
			{int t=x[k]; x[k]=x[i]; x[i]=t;}
			f(x,k+1);
			_______________________________________       // 填空
		}
	}
	
	public static void main(String[] args)
	{
		int[] x = {1,2,3,4,5,6,7,8,9};		
		f(x,0);
	}
}
```
注意，只能填写缺少的部分，不要重复抄写已有代码。不要填写任何多余的文字。

解题思路：这题是排列组合问题，代码中就是对9个数字进行全排列。
答案：
```
{int t=x[k]; x[k]=x[i]; x[i]=t;}
```

6、加法变乘法

我们都知道：1+2+3+ ... + 49 = 1225
现在要求你把其中两个不相邻的加号变成乘号，使得结果为2015

比如：
1+2+3+...+10*11+12+...+27*28+29+...+49 = 2015
就是符合要求的答案。

请你寻找另外一个可能的答案，并把位置靠前的那个乘号左边的数字提交（对于示例，就是提交10）。

注意：需要你提交的是一个整数，不要填写任何多余的内容。

解题思路：这种题目意思很明确，可以借助计算机的强大计算能力进行遍历,需要注意题目中的```不相邻的加号```。
方法一(逆向)：

```
package lanqiao;

public class Six {

	public static void main(String[] args) {

		int sum;
		for (int i = 1; i < 49; i++){
			for (int j = i+2; j < 49; j++) {
				sum = 1225;
				sum -= i+i+1+j+j+1;//减去四个相乘的数
				sum += i*(i+1)+j*(j+1);
				if(sum == 2015) {
					System.out.println("i=" + i + ",j=" + j);
				}
			}
		}
		
	}

}
```


方法二(正向)：
```
package lanqiao;

public class Six {

	public static void main(String[] args) {

		int sum = 0;
		// 总共有48个加号，符号的下标即为左边的数字
		for (int i = 1; i < 49; i++) {
			// 乘号之间至少间距2
			for (int j = i + 2; j < 49; j++) {
				sum = 0;
				for (int k = 1; k < 50; k++) {
					if (i == k || j == k) {
						sum += k * (k + 1);
						k++;//两个数相乘，要略过for循环一次
					} else {
						sum += k;
					}
				}
				if (sum == 2015) {
					System.out.println("i=" + i + ",j=" + j);
				}
			}
		}
	}

}

```
打印结果
```
i=10,j=27
i=16,j=24
```
即
```
1+2+3+...+10*11+12+...+27*28+29+...+49 = 2015
1+2+3+...+16*17+18+...+24*25+26+...+49 = 2015
```
答案：
```
16
```
7、牌型种数

小明被劫持到X赌城，被迫与其他3人玩牌。
一副扑克牌（去掉大小王牌，共52张），均匀发给4个人，每个人13张。
这时，小明脑子里突然冒出一个问题：
如果不考虑花色，只考虑点数，也不考虑自己得到的牌的先后顺序，自己手里能拿到的初始牌型组合一共有多少种呢？

请填写该整数，不要填写任何多余的内容或说明文字。

解题思路：动态规划

```
package lanqiao;

public class Seven {

	public static void main(String[] args) {
		int[][] dp = new int[14][14];
		dp[0][0] = 1;
		for (int i = 1; i < 14; i++)
			for (int j = 0; j < 14; j++)
				for (int k = 0; k < 5; k++)
					if (j + k <= 13)
						dp[i][j + k] += dp[i - 1][j];
		System.out.println(dp[13][13]);
	}
}

```
答案：
```
3598180
```

8、饮料换购

乐羊羊饮料厂正在举办一次促销优惠活动。乐羊羊C型饮料，凭3个瓶盖可以再换一瓶C型饮料，并且可以一直循环下去，但不允许赊账。

请你计算一下，如果小明不浪费瓶盖，尽量地参加活动，那么，对于他初始买入的n瓶饮料，最后他一共能得到多少瓶饮料。

输入：一个整数n，表示开始购买的饮料数量（0<n<10000）
输出：一个整数，表示实际得到的饮料数

例如：
用户输入：
100
程序应该输出：
149

用户输入：
101
程序应该输出：
151


资源约定：
峰值内存消耗（含虚拟机） < 256M
CPU消耗  < 1000ms

答案：

请严格按要求输出，不要画蛇添足地打印类似：“请您输入...” 的多余内容。

所有代码放在同一个源文件中，调试通过后，拷贝提交该源码。
注意：不要使用package语句。不要使用jdk1.7及以上版本的特性。
注意：主类的名字必须是：Main，否则按无效代码处理。

解题思路：这道题目逻辑比较清晰简单，但是需要注意这个输入整数的范围，比如是用int 还是long，否则会因为有些测试用例超出了该数据类型所能表示的范围而发生异常，一般比赛里面都会有10个左右的测试用例，每个测试用例对应一定的分数，另外比赛类的题目一定要严格按照指定格式输出，否则就白做了。在比赛中一般是使用黑盒自动化测试。

```
import java.util.Scanner;

public class Main {
	
	public static void main(String[] args) {
		long sum = 0;
		Scanner sc = new Scanner(System.in);
		long n = sc.nextLong();
		sum += n;
		while(n >= 3) {
			long n2 = n / 3;
			sum += n2;
			n = n2 + n % 3;//目前拥有的瓶盖 = 换来的饮料+剩余的瓶盖（不是3的别倍数的时候会有）
		}
		
		System.out.println(sum);
	}

}


```


9、垒骰子

赌圣atm晚年迷恋上了垒骰子，就是把骰子一个垒在另一个上边，不能歪歪扭扭，要垒成方柱体。
经过长期观察，atm 发现了稳定骰子的奥秘：有些数字的面贴着会互相排斥！
我们先来规范一下骰子：1 的对面是 4，2 的对面是 5，3 的对面是 6。
假设有 m 组互斥现象，每组中的那两个数字的面紧贴在一起，骰子就不能稳定的垒起来。 atm想计算一下有多少种不同的可能的垒骰子方式。
两种垒骰子方式相同，当且仅当这两种方式中对应高度的骰子的对应数字的朝向都相同。
由于方案数可能过多，请输出模 10^9 + 7 的结果。

不要小看了 atm 的骰子数量哦～

「输入格式」
第一行两个整数 n m
n表示骰子数目
接下来 m 行，每行两个整数 a b ，表示 a 和 b 不能紧贴在一起。

「输出格式」
一行一个数，表示答案模 10^9 + 7 的结果。

「样例输入」
2 1
1 2

「样例输出」
544

「数据范围」
对于 30% 的数据：n <= 5
对于 60% 的数据：n <= 100
对于 100% 的数据：0 < n <= 10^9, m <= 36


资源约定：
峰值内存消耗（含虚拟机） < 256M
CPU消耗  < 2000ms


请严格按要求输出，不要画蛇添足地打印类似：“请您输入...” 的多余内容。

所有代码放在同一个源文件中，调试通过后，拷贝提交该源码。
注意：不要使用package语句。不要使用jdk1.7及以上版本的特性。
注意：主类的名字必须是：Main，否则按无效代码处理。

解题思路：递推,还得用矩阵加速吧
```
import java.util.Arrays;
import java.util.Scanner;

public class Main {
	static int MOD = (int) (1e9 + 7);

	public static void main(String[] args) {
		int[][] ar = new int[40][40];
		int[] mm = { 0, 4, 5, 6, 1, 2, 3 };
		long[][] dp = new long[2][7];
		Scanner in = new Scanner(System.in);
		int n = in.nextInt();
		int m = in.nextInt();
		for (int i = 0; i < m; i++) {
			int u = in.nextInt();
			int v = in.nextInt();
			ar[u][v] = 1;
			ar[v][u] = 1;
		}

		int pre = 1;
		int now = 0;
		for (int i = 1; i < 7; i++)
			dp[0][i] = 4;
		for (int i = 2; i <= n; i++) {
			pre = (pre + 1) % 2;
			now = (now + 1) % 2;
			Arrays.fill(dp[now], 0);
			for (int j = 1; j < 7; j++)
				for (int k = 1; k < 7; k++) {
					dp[now][j] += (dp[pre][k] * 4) % MOD;
				}
			for (int j = 0; j < 7; j++) {
				for (int k = 0; k <= j; k++)
					if (ar[j][k] == 1) {
						dp[now][mm[j]] = (MOD + dp[now][mm[j]] - dp[pre][k] * 4) % MOD;
						dp[now][mm[k]] = (MOD + dp[now][mm[k]] - dp[pre][j] * 4) % MOD;
					}
			}
		}
		long ans = 0;
		for (int i = 1; i < 7; i++)
			ans = (ans + dp[now][i]) % MOD;
		System.out.println(ans);

	}
}
```

10、生命之树

在X森林里，上帝创建了生命之树。

他给每棵树的每个节点（叶子也称为一个节点）上，都标了一个整数，代表这个点的和谐值。
上帝要在这棵树内选出一个非空节点集S，使得对于S中的任意两个点a,b，都存在一个点列 {a, v1, v2, ..., vk, b} 使得这个点列中的每个点都是S里面的元素，且序列中相邻两个点间有一条边相连。

在这个前提下，上帝要使得S中的点所对应的整数的和尽量大。
这个最大的和就是上帝给生命之树的评分。

经过atm的努力，他已经知道了上帝给每棵树上每个节点上的整数。但是由于 atm 不擅长计算，他不知道怎样有效的求评分。他需要你为他写一个程序来计算一棵树的分数。

「输入格式」
第一行一个整数 n 表示这棵树有 n 个节点。
第二行 n 个整数，依次表示每个节点的评分。
接下来 n-1 行，每行 2 个整数 u, v，表示存在一条 u 到 v 的边。由于这是一棵树，所以是不存在环的。

「输出格式」
输出一行一个数，表示上帝给这棵树的分数。

「样例输入」
5
1 -2 -3 4 5
4 2
3 1
1 2
2 5

「样例输出」
8

「数据范围」
对于 30% 的数据，n <= 10
对于 100% 的数据，0 < n <= 10^5, 每个节点的评分的绝对值不超过 10^6 。

资源约定：
峰值内存消耗（含虚拟机） < 256M
CPU消耗  < 3000ms


请严格按要求输出，不要画蛇添足地打印类似：“请您输入...” 的多余内容。

所有代码放在同一个源文件中，调试通过后，拷贝提交该源码。
注意：不要使用package语句。不要使用jdk1.7及以上版本的特性。
注意：主类的名字必须是：Main，否则按无效代码处理。

解题思路：LAC问题

```
import java.util.Arrays;
import java.util.Scanner;

public class Main {
	static int[] head1 = new int[100010];
	static int[] head2 = new int[100010];
	static int[] node = new int[100010];
	static Edge[] edge1 = new Edge[300010];
	static Edge[] edge2 = new Edge[300010];
	static int[] degree = new int[100010];
	static int[] fa = new int[100010];
	static int tot1 = 0, tot2 = 0;
	static int Node;
	static long[] max = new long[100010];
	static long ans = Long.MIN_VALUE;

	static void add1(int u, int v) {
		edge1[tot1].u = u;
		edge1[tot1].v = v;
		edge1[tot1].nxt = head1[u];
		head1[u] = tot1++;
	}

	static void add2(int u, int v) {
		edge2[tot2].u = u;
		edge2[tot2].v = v;
		edge2[tot2].nxt = head2[u];
		head2[u] = tot2++;
	}

	static void dfs(int u, int par, long nn) {
		max[u] = nn;
		fa[u] = u;
		for (int i = head1[u]; i != -1; i = edge1[i].nxt) {
			int v = edge1[i].v;
			if (v == par)
				continue;
			dfs(v, u, nn + node[v]);
			fa[v] = u;
		}
		for (int i = head2[u]; i != -1; i = edge2[i].nxt) {
			int v = edge2[i].v;
			if (max[v] != Long.MAX_VALUE) {
				ans = Math.max(ans, max[v] + max[u] - max[parent(v)]);
			}
		}
	}

	static int parent(int x) {
		if (fa[x] == -1 || fa[x] == x)
			return x;
		return fa[x] = parent(fa[x]);
	}

	public static void main(String[] args) {
		Scanner in = new Scanner(System.in);
		Node = in.nextInt();
		for (int i = 0; i < Node; i++)
			node[i] = in.nextInt();
		for (int i = 0; i <= Node + Node; i++)
			edge1[i] = new Edge();
		for (int i = 0; i <= Node * Node * 2; i++)
			edge2[i] = new Edge();
		Arrays.fill(fa, -1);
		Arrays.fill(head1, -1);
		Arrays.fill(head2, -1);
		Arrays.fill(max, Long.MAX_VALUE);
		for (int i = 1; i < Node; i++) {
			int u, v;
			u = in.nextInt() - 1;
			v = in.nextInt() - 1;
			degree[u]++;
			degree[v]++;
			add1(u, v);
			add1(v, u);
		}
		for (int i = 0; i < Node; i++)
			for (int j = 0; j < Node; j++) {
				add2(i, j);
				add2(j, i);
			}
		if (Node == 2) {
			System.out.println(node[0] + node[1]);
			return;
		}
		dfs(0, -1, node[0]);
		System.out.println(ans);
	}
}

class Edge {
	int u, v, nxt;
	int par;
}
```

总体来看，省赛的题目相对来说是比较简单，如愿获得省一等奖，5月份去帝都参加决赛。另外，看起来奖项很多的样子，如果没有什么特别出色的技能，也可以给简历增加一点东西。


  [1]: https://raw.githubusercontent.com/jdqm/hello-world/master/lanqiao6/1.jpg
  [2]: https://raw.githubusercontent.com/jdqm/hello-world/master/lanqiao6/3.jpg