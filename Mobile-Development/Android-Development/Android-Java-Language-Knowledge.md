# Android Java 语法知识点

## 1. Java 在 Android 中的定位

Android 早期主要使用 Java 开发。现在 Kotlin 是 Android 新项目更常见的选择，但 Java 仍然非常重要：

- 大量老项目使用 Java。
- Android SDK 很多 API 以 Java 风格设计。
- 很多第三方库文档仍提供 Java 示例。
- 面试和源码阅读经常涉及 Java 基础。

Java 特点：

| 特点 | 说明 |
|------|------|
| 静态类型 | 变量类型编译期确定 |
| 面向对象 | 类、对象、继承、接口是核心 |
| 自动内存管理 | JVM/ART 负责 GC |
| 跨平台 | 编译为字节码，Android 上运行在 ART |

## 2. Hello World

```java
public class Main {
    public static void main(String[] args) {
        // println 输出一行文本
        System.out.println("Hello Java");
    }
}
```

说明：

- `class Main` 定义一个类。
- `main` 是普通 Java 程序入口。
- Android App 不从 `main` 开始，而是由系统启动 Activity 等组件。

Android Activity 示例：

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        TextView textView = new TextView(this);
        textView.setText("Hello Android Java");
        setContentView(textView);
    }
}
```

## 3. 变量与基本类型

```java
int age = 18;              // 整数
long userId = 10001L;      // 长整数
float price = 9.9f;        // 单精度浮点
double score = 99.5;       // 双精度浮点
boolean loggedIn = true;   // 布尔
char level = 'A';          // 字符
String name = "Tom";       // 字符串，引用类型
```

基本类型：

| 类型 | 说明 |
|------|------|
| `byte` | 8 位整数 |
| `short` | 16 位整数 |
| `int` | 32 位整数，最常用 |
| `long` | 64 位整数 |
| `float` | 32 位浮点 |
| `double` | 64 位浮点 |
| `boolean` | true / false |
| `char` | 单个字符 |

引用类型：

```java
String text = "hello";
User user = new User("Tom", 18);
```

引用类型可以是 `null`：

```java
String name = null;

if (name != null) {
    System.out.println(name.length());
}
```

Android 开发中，空指针是常见崩溃来源。

## 4. 常量

Java 使用 `final` 定义不可重新赋值的变量。

```java
final int maxCount = 10;
// maxCount = 20; // 编译错误
```

类常量通常写成：

```java
public class Constants {
    public static final String BASE_URL = "https://api.example.com/";
    public static final int PAGE_SIZE = 20;
}
```

使用：

```java
String url = Constants.BASE_URL;
```

## 5. 运算符

```java
int a = 10;
int b = 3;

int sum = a + b;       // 13
int diff = a - b;      // 7
int product = a * b;   // 30
int div = a / b;       // 3，整数除法
int mod = a % b;       // 1，取余

boolean result = a > b && b > 0;
```

常用逻辑运算：

| 运算符 | 作用 |
|--------|------|
| `&&` | 与 |
| `||` | 或 |
| `!` | 非 |
| `==` | 基本类型比较值，引用类型比较地址 |
| `.equals` | 比较对象内容 |

字符串比较：

```java
String a = new String("hello");
String b = new String("hello");

System.out.println(a == b);        // false，比较引用地址
System.out.println(a.equals(b));   // true，比较内容
```

## 6. 条件语句

```java
int score = 85;

if (score >= 90) {
    System.out.println("A");
} else if (score >= 60) {
    System.out.println("Pass");
} else {
    System.out.println("Fail");
}
```

`switch`：

```java
int tab = 1;

switch (tab) {
    case 0:
        System.out.println("Home");
        break;
    case 1:
        System.out.println("Profile");
        break;
    default:
        System.out.println("Unknown");
        break;
}
```

注意：传统 `switch` 中不要忘记 `break`，否则会继续执行下一个分支。

## 7. 循环

`for`：

```java
for (int i = 0; i < 5; i++) {
    System.out.println(i);
}
```

增强 `for`：

```java
List<String> names = Arrays.asList("Tom", "Jerry");

for (String name : names) {
    System.out.println(name);
}
```

`while`：

```java
int count = 0;
while (count < 3) {
    count++;
}
```

`break` 和 `continue`：

```java
for (int i = 0; i < 10; i++) {
    if (i == 3) {
        continue; // 跳过本次循环
    }
    if (i == 8) {
        break; // 结束整个循环
    }
}
```

## 8. 方法

```java
public int add(int a, int b) {
    return a + b;
}
```

无返回值：

```java
public void showMessage(String message) {
    System.out.println(message);
}
```

Android 示例：

```java
private void showToast(String message) {
    Toast.makeText(this, message, Toast.LENGTH_SHORT).show();
}
```

方法重载：

```java
public void log(String message) {
    System.out.println(message);
}

public void log(String tag, String message) {
    System.out.println(tag + ": " + message);
}
```

## 9. 类与对象

类是模板，对象是实例。

```java
public class User {
    private String name;
    private int age;

    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }
}
```

使用：

```java
User user = new User("Tom", 18);
System.out.println(user.getName());
```

`this` 表示当前对象：

```java
this.name = name;
```

## 10. 访问控制

| 修饰符 | 范围 |
|--------|------|
| `public` | 任意位置 |
| `protected` | 同包或子类 |
| 默认不写 | 同包 |
| `private` | 当前类内部 |

建议：

- 字段优先 `private`。
- 对外暴露必要方法。
- 不要把所有东西都写成 `public`。

## 11. 继承

```java
public class Animal {
    public void eat() {
        System.out.println("eat");
    }
}

public class Dog extends Animal {
    public void bark() {
        System.out.println("bark");
    }
}
```

使用：

```java
Dog dog = new Dog();
dog.eat();
dog.bark();
```

方法重写：

```java
public class Dog extends Animal {
    @Override
    public void eat() {
        System.out.println("dog eat");
    }
}
```

Android 中常见继承：

```java
public class MainActivity extends AppCompatActivity {
}
```

## 12. 接口

接口定义能力，类负责实现。

```java
public interface OnLoginListener {
    void onSuccess(String token);
    void onError(Throwable error);
}
```

实现：

```java
public class LoginPresenter {
    public void login(String username, String password, OnLoginListener listener) {
        if (username.isEmpty()) {
            listener.onError(new IllegalArgumentException("username empty"));
            return;
        }

        listener.onSuccess("token_123");
    }
}
```

调用：

```java
presenter.login("tom", "123456", new OnLoginListener() {
    @Override
    public void onSuccess(String token) {
        Log.d("Login", "token=" + token);
    }

    @Override
    public void onError(Throwable error) {
        Log.e("Login", "failed", error);
    }
});
```

## 13. 抽象类

抽象类可以包含抽象方法和普通方法。

```java
public abstract class BaseRepository {
    public abstract String getName();

    public void logName() {
        System.out.println(getName());
    }
}
```

子类实现：

```java
public class UserRepository extends BaseRepository {
    @Override
    public String getName() {
        return "UserRepository";
    }
}
```

抽象类适合表达“是什么”，接口适合表达“能做什么”。

## 14. 集合

### 14.1 List

有序，可重复。

```java
List<String> names = new ArrayList<>();
names.add("Tom");
names.add("Jerry");

String first = names.get(0);
int size = names.size();
```

遍历：

```java
for (String name : names) {
    System.out.println(name);
}
```

### 14.2 Set

无重复。

```java
Set<String> tags = new HashSet<>();
tags.add("android");
tags.add("android");

System.out.println(tags.size()); // 1
```

### 14.3 Map

键值对。

```java
Map<String, Integer> scores = new HashMap<>();
scores.put("Tom", 90);
scores.put("Jerry", 80);

Integer tomScore = scores.get("Tom");
```

遍历：

```java
for (Map.Entry<String, Integer> entry : scores.entrySet()) {
    System.out.println(entry.getKey() + "=" + entry.getValue());
}
```

## 15. 泛型

泛型让类型更安全。

```java
List<String> names = new ArrayList<>();
names.add("Tom");
// names.add(123); // 编译错误
```

自定义泛型类：

```java
public class ApiResult<T> {
    private T data;
    private String message;

    public ApiResult(T data, String message) {
        this.data = data;
        this.message = message;
    }

    public T getData() {
        return data;
    }
}
```

使用：

```java
ApiResult<User> result = new ApiResult<>(user, "ok");
User data = result.getData();
```

泛型方法：

```java
public <T> T first(List<T> list) {
    if (list == null || list.isEmpty()) {
        return null;
    }
    return list.get(0);
}
```

## 16. 异常处理

```java
try {
    int result = 10 / 0;
} catch (ArithmeticException e) {
    Log.e("Demo", "calculate failed", e);
} finally {
    Log.d("Demo", "always run");
}
```

自定义异常：

```java
public class ApiException extends RuntimeException {
    public ApiException(String message) {
        super(message);
    }
}
```

抛出异常：

```java
public User requireUser(User user) {
    if (user == null) {
        throw new ApiException("user is null");
    }
    return user;
}
```

Android 建议：

- 不要吞异常。
- 日志要带堆栈。
- UI 层把异常转换为用户能理解的提示。

## 17. null 与 Optional 思维

Java 引用类型可能为 null：

```java
User user = repository.findUser();

if (user != null) {
    textView.setText(user.getName());
} else {
    textView.setText("Unknown");
}
```

常见空指针：

```java
String name = null;
int length = name.length(); // 崩溃
```

防御方式：

```java
if (name != null && !name.isEmpty()) {
    System.out.println(name);
}
```

Android 中也可以使用注解表达空值约束：

```java
public void setName(@NonNull String name) {
    this.name = name;
}

@Nullable
public User findUser() {
    return null;
}
```

## 18. Lambda

Java 8 支持 Lambda。适合函数式接口，也就是只有一个抽象方法的接口。

```java
button.setOnClickListener(view -> {
    Toast.makeText(this, "clicked", Toast.LENGTH_SHORT).show();
});
```

等价匿名内部类：

```java
button.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        Toast.makeText(MainActivity.this, "clicked", Toast.LENGTH_SHORT).show();
    }
});
```

自定义函数式接口：

```java
public interface Mapper<T, R> {
    R map(T value);
}
```

使用：

```java
Mapper<User, String> mapper = user -> user.getName();
```

## 19. 线程基础

Android 主线程不能执行耗时任务。

直接创建线程：

```java
new Thread(() -> {
    // 子线程执行耗时任务
    String result = loadData();

    // 回到主线程更新 UI
    runOnUiThread(() -> {
        textView.setText(result);
    });
}).start();
```

线程池：

```java
ExecutorService executor = Executors.newSingleThreadExecutor();

executor.execute(() -> {
    String result = loadData();
    runOnUiThread(() -> textView.setText(result));
});
```

注意：

- 不要在子线程直接更新 UI。
- 不要无限创建线程。
- Activity 销毁后要避免回调继续持有 Activity。

## 20. Handler

Handler 常用于线程切换和延迟任务。

```java
Handler mainHandler = new Handler(Looper.getMainLooper());

new Thread(() -> {
    String result = loadData();

    mainHandler.post(() -> {
        // 这里运行在主线程
        textView.setText(result);
    });
}).start();
```

延迟执行：

```java
mainHandler.postDelayed(() -> {
    Log.d("Demo", "run after 1 second");
}, 1000);
```

## 21. 内部类与内存泄漏

非静态内部类会持有外部类引用。

有风险写法：

```java
public class MainActivity extends AppCompatActivity {
    private Handler handler = new Handler();

    private Runnable task = new Runnable() {
        @Override
        public void run() {
            // 匿名内部类隐式持有 MainActivity
            textView.setText("done");
        }
    };
}
```

改进思路：

- 在 `onDestroy` 中移除回调。
- 使用静态内部类 + WeakReference。
- 使用生命周期感知组件。

```java
@Override
protected void onDestroy() {
    super.onDestroy();
    handler.removeCallbacksAndMessages(null);
}
```

## 22. Java Bean / 数据类

Java 常用 Bean：

```java
public class User {
    private long id;
    private String name;

    public User(long id, String name) {
        this.id = id;
        this.name = name;
    }

    public long getId() {
        return id;
    }

    public String getName() {
        return name;
    }
}
```

如果需要比较内容，重写 `equals` 和 `hashCode`：

```java
@Override
public boolean equals(Object obj) {
    if (this == obj) return true;
    if (!(obj instanceof User)) return false;

    User other = (User) obj;
    return id == other.id;
}

@Override
public int hashCode() {
    return Long.hashCode(id);
}
```

## 23. Android 常见 Java 写法

### 23.1 点击事件

```java
Button button = findViewById(R.id.loginButton);

button.setOnClickListener(v -> {
    login();
});
```

### 23.2 列表适配器简化示例

```java
public class UserAdapter extends RecyclerView.Adapter<UserViewHolder> {
    private final List<User> users = new ArrayList<>();

    public void submitList(List<User> newUsers) {
        users.clear();
        users.addAll(newUsers);
        notifyDataSetChanged();
    }

    @Override
    public UserViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        TextView textView = new TextView(parent.getContext());
        return new UserViewHolder(textView);
    }

    @Override
    public void onBindViewHolder(UserViewHolder holder, int position) {
        User user = users.get(position);
        holder.bind(user);
    }

    @Override
    public int getItemCount() {
        return users.size();
    }
}
```

ViewHolder：

```java
public class UserViewHolder extends RecyclerView.ViewHolder {
    private final TextView textView;

    public UserViewHolder(View itemView) {
        super(itemView);
        textView = (TextView) itemView;
    }

    public void bind(User user) {
        textView.setText(user.getName());
    }
}
```

## 24. Java 与 Kotlin 对比

| Java | Kotlin |
|------|--------|
| 空值靠约定和注解 | 类型系统区分可空和非空 |
| Bean 代码较多 | `data class` 简洁 |
| 匿名内部类常见 | Lambda 更自然 |
| 线程常用 Handler/Executor | 协程更常见 |
| 需要分号 | 通常不需要分号 |

## 25. 学习建议

学习顺序：

```text
变量和方法
  |
  v
类和对象
  |
  v
继承和接口
  |
  v
集合和泛型
  |
  v
异常和空值
  |
  v
Lambda 和线程
  |
  v
Android API 实战
```

## 26. 参考资料

- Java Tutorials：https://docs.oracle.com/javase/tutorial/
- Android Java 8 支持：https://developer.android.com/studio/write/java8-support
- Android App fundamentals：https://developer.android.com/guide/components/fundamentals

## 27. 总结

Android Java 学习要抓住这些关键点：

| 主题 | 为什么重要 |
|------|------------|
| 类和对象 | Android API 基本都是对象模型 |
| 接口和回调 | 点击事件、异步回调、适配器常用 |
| 集合和泛型 | 列表、接口返回、数据转换常用 |
| null 判断 | 空指针是 Java Android 常见崩溃 |
| 线程 | 主线程不能做耗时任务 |
| 生命周期 | 回调和线程要避免 Activity 泄漏 |
