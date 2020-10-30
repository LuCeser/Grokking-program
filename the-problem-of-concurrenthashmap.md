## 主题

- 并发问题的三个来源：原子性、可见性、有序性
- `ConcurrentHashMap`只能保证提供的原子性读写操作是线程安全的

## 用户注册模拟并发问题

我们从一个用户注册的例子来了解并发问题。

在这个例子中模拟了用户注册行为，定义了相同用户名不能重复注册的规则，我们使用`ConcurrentHashMap`保存用户信息，通过模拟同时注册的动作体现并发问题。

### 定义用户类

```java
class User {
	// 用户名，也是Map的key
	private String username; 
	private int age;
	// 省略getter, setter方法
}
```

### 定义用户注册逻辑

用户注册的规则是用户名不能重复，假如重复就返回注册失败，我们也考虑到了线程安全，所以用`ConcurrentHashMap`来存储用户信息

```java
class UserService {
	
	private Map<String, User> userMap = new ConcurrentHashMap();
	
	boolean register(User user) {
		if (userMap.containsKey(user.getUsername)) {
			log.info("用户已存在");
			return false;
		} else {
			userMap.put(user.getUsername, user);
            log.info("用户注册成功, {}, {}", user.getUsername(), user.getAge());			
			return true;
		}
	}
}
```


### 模拟重复注册

接下来模拟用户重复注册的场景：

```java
int threadCount = 8;

ForkJoinPool forkJoinPool = new ForkJoinPool(threadCount);

forkJoinPool.execute(() -> IntStream.range(0, threadCount)
    .mapToObj(i -> new Person("张三", i))
    .parallel().forEach(UserService::register));

TimeUnit.SECONDS.sleep(1);

```

输出结果:

```text
00:18:32.622 [ForkJoinPool-1-worker-1] INFO org.example.UserService - 用户注册成功, 张三, 5
00:18:32.622 [ForkJoinPool-1-worker-0] INFO org.example.UserService - 用户已存在
00:18:32.622 [ForkJoinPool-1-worker-4] INFO org.example.UserService - 用户注册成功, 张三, 1
00:18:32.622 [ForkJoinPool-1-worker-6] INFO org.example.UserService - 用户已存在
00:18:32.622 [ForkJoinPool-1-worker-5] INFO org.example.UserService - 用户注册成功, 张三, 4
00:18:32.622 [ForkJoinPool-1-worker-3] INFO org.example.UserService - 用户已存在
00:18:32.622 [ForkJoinPool-1-worker-2] INFO org.example.UserService - 用户注册成功, 张三, 2
00:18:32.622 [ForkJoinPool-1-worker-7] INFO org.example.UserService - 用户已存在
```

可以看到，在注册中存在判断用户是否已注册的逻辑，但在实际测试中有4个用户同时注册成功。[^1]

[^1]: 线程问题有其不确定性，同一个测试用例会跑出不同结果来，也有可能没有出现并发问题，可以多跑几次测试用例看下结果。
