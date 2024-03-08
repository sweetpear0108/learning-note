# 质数筛法
问题：判断1到n中有哪些质数（[LC.204 质数计数](https://leetcode.cn/problems/count-primes/submissions/505633608/)）

## 算术基本定理
**算术基本定理**，又称为正整数的唯一分解定理，即：每个大于1的自然数，要么本身就是质数，要么可以写为2个或以上的质数的积，而且这些质因子按大小排列之后，写法仅有一种方式。

**推论**：任何一个合数都可以表示成一个质数和另一个数的乘积

## 埃氏筛 sieve of Eratosthenes
埃氏筛（ sieve of Eratosthenes）对于**每一个质数**，标记它在n以内的所有倍数，根据**推论**可知当遍历结束后，所有合数都被标记。

* 优点：相比于对n以内每个数朴素的单独判断，时间复杂度更低
* 缺点：对于同一个合数存在重复标记的现象（18: `2*9` `3*6`）

代码：
```go
func sieveOfEratosthenes(n int) {
	notPrime := map[int]bool{1: true} // 取反可以省略初始化过程
	for i := 2; i*i <= n; i++ {
		if !notPrime[i] {
			for j := i * i; j <= n; j += i { // j从i*i开始，因为小于i*i的值已经在之前判断过了
				notPrime[j] = true
			}
		}
	}
}
```

## 欧拉筛 sieve of Euler
欧拉筛对于**1到n的每个数**，标记它与比它小的所有质数分别相乘得到的合数，在标记时注意保证合数是由其**最小的质因数**得到的（保证唯一标记的原理）。根据**推论**可知当遍历结束后，所有合数都被标记。

* 优点：比埃氏筛更快，因为避免了合数的重复标记
* 缺点：需要额外的空间保存已经验证的质数，以空间换时间

代码：
```go
func sieveOfEuler(n int) {
	primes := []int{}
	notPrime := map[int]bool{1: true}
	for i := 2; i <= n; i++ {
		if !notPrime[i] {
			primes = append(primes, i)
		}
		for j := 0; j < len(primes) && i*primes[j] <= n; j++ {
			notPrime[i*primes[j]] = true
			if i%primes[j] == 0 { // 保证每个合数都被其最小质因子筛去
				break
			}
		}
	}
}
```

### 代码中 i%primes[j] == 0 什么意思
假设：`i = 4` 时 `primes = []int{2, 3}`

如果不做`i%primes[j] == 0`判断，那么会标记合数 `4*3=12`，

那么当`i = 6`时，就会出现合数`6*2=12`的重复标记，而欧式筛正是为了避免这种情况的出现，对于合数，只由他的最小质因数筛去。

* 为什么primes[j]之后的质数都跳过了？

因为i是primes[j]的整数倍，那么他再乘后续的每个数都相当于`(a*primes[j])*x`（a：倍数，x：后续某个数），

等价于`primes[j]*(a*x)`，这个`a*x`就留给之后的i了，就是为了保证**每个合数只由他的最小质因数筛去**。
