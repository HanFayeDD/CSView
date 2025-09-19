### 面向对象

- 封装

  - 属性和方法
  - 安全性和可维护性

- 继承

  - 复用父类方法
  - 重写父类方法

  - java只能单继承、但可以实现多个接口实现多继承

- 多态
  - 同一个行为在不同对象上具有不同表现形式，**方法重写（继承父类）和接口实现（实现接口）**来实现

#### 重载与重写

- 重载

  重载就是同一个类中多个同名方法根据不同的传参来执行不同的逻辑处理。

  **方法名必须相同**，参数类型不同、个数不同、顺序不同，方法返回值和访问修饰符可以不同。

- 重写

  子类对父类方法修改。**方法名、参数列表必须相同**。修饰符范围可以更大、异常更小

#### 设计模式

##### 单例模式

唯一性与全局访问

- 饿汉式

  - 线程安全（类加载就初始化）
  - 未使用造成内存资源的浪费

  ```java
  public class Singleton {
      // 1. 创建静态的唯一实例
      private static final Singleton INSTANCE = new Singleton();
  
      // 2. 私有构造函数，防止外部创建对象
      private Singleton() {}
  
      // 3. 提供公共访问点
      public static Singleton getInstance() {
          return INSTANCE;
      }
  }
  ```

- 懒汉式

  - 延迟加载、通过锁保证线程安全
  - 但性能较低、每次获取实例都要获锁

  ```java
  public class Singleton {
      private static Singleton instance;
  
      // 私有构造函数
      private Singleton() {}
  
      // 提供线程安全的访问点
      public static synchronized Singleton getInstance() {
          if (instance == null) {
              instance = new Singleton();
          }
          return instance;
      }
  }
  ```

  

  #### 接口和抽象类的区别
  
  - 关键字interface、abstract
  - 多实现、单继承
  - 方法：抽象方法；抽象方法和具体方法
  - 属性：只能定义常量；可以定义成员属性
  - 构造函数：无；有
  - 访问修饰符：只能public；都行



#### 异常和错误

在 Java 中，**异常（Exception）** 和 **错误（Error）** 都是继承自 `Throwable` 类的子类，用于表示程序执行期间发生的问题。但它们有明显的区别，主要体现在用途、严重性和处理方式上。

异常又可分为

- 检查型：编译时必须显式处理，否则编译不通过
- 运行时：

| **属性**         | **异常（Exception）**                                        | **错误（Error）**                                            |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **继承自**       | `java.lang.Exception`                                        | `java.lang.Error`                                            |
| **严重性**       | 通常是较轻的、可以恢复的问题。                               | 严重问题，通常是系统相关，程序无法恢复。                     |
| **是否可处理**   | 可以通过 `try-catch` 或 `throws` 处理。                      | 通常不需要处理，因为恢复通常是不可能的。                     |
| **分类**         | - 检查异常（Checked Exception）- 运行时异常（Unchecked Exception） | 没有分类，所有错误都属于不可恢复的系统级问题。               |
| **常见类型**     | - `IOException`、`SQLException`（检查异常）- `NullPointerException`、`ClassCastException`（运行时异常） | - `OutOfMemoryError`、`StackOverflowError`、`VirtualMachineError` 等 |
| **何时发生**     | 通常由程序逻辑或外部因素引发。                               | 通常由 JVM 或系统级问题引发。                                |
| **是否强制处理** | 检查异常需要强制处理（`try-catch` 或 `throws`）。            | 错误不需要强制处理，通常也不建议捕获。                       |

​                

#### 反射

在程序运行时，动态地获取类的信息并操作类或对象（方法、属性）的能力。

- 优点
  - 灵活性和动态性
  - 解耦
- 缺点
  - 性能开销
  - 安全性
  - 可维护性



### AOP面向切面编程

通过反射实现动态代理

想在调用某个方法前后自动加点料（比如打日志、开事务、做权限检查）？AOP（面向切面编程）就是干这个的，而动态代理是实现 AOP 的常用手段。JDK 自带的动态代理（Proxy 和 InvocationHandler）就离不开反射。代理对象在内部调用真实对象的方法时，就是通过反射的 `Method.invoke` 来完成的。

````java
public class DebugInvocationHandler implements InvocationHandler {
    private final Object target; // 真实对象

    public DebugInvocationHandler(Object target) { this.target = target; }

    // proxy: 代理对象, method: 被调用的方法, args: 方法参数
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("切面逻辑：调用方法 " + method.getName() + " 之前");
        // 通过反射调用真实对象的同名方法
        Object result = method.invoke(target, args);
        System.out.println("切面逻辑：调用方法 " + method.getName() + " 之后");
        return result;
    }
}
````

#### 控制反转iOC与依赖注入DI

程序不再负责查找或创建依赖对象，而是**依赖容器将对象注入到程序中**

依赖注入就是实现控制反转的方法之一,常见方法：

- 构造器注入
- setter注入
- 接口注入
