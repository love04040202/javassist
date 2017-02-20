# javassist
> http://www.cnblogs.com/exolution/archive/2012/10/24/2736597.html

【以前的文章，从我的其他博客搬来】
关键词：AOP、SOC、代理模式、动态代理。
 
上一话，我提到了一个重要的概念就是SOC，即关注点分离。为什么要分离关注点，因为这样可以让我们更加简单而富有条理的去处理这个繁杂的世界。AOP就是SOC的一种体现。

 

1. 什么是AOP？
AOP全程是Aspect Oriented Programming意即面向切面编程。他并不是什么OOP的替代技术，只是OOP的一种延续。使用SOC的思想，去解耦，去降低问题的复杂度。那么，为什么要叫做面向切面编程呢？想想一下，当你在软件开发设计类的时候，会不会发现，有些逻辑或者功能，是每个类或者说大多数类都需要完成的呢？比如异常/错误处理，比如系统日志记录，比如持久化中的事务处理。与其一个一个的实现，为什么不能把他们单独提取出来加以处理呢。如果说把这些类平行的放在一起。这些公共的部分刚好并排在一起，能够一刀切下。可能这就是切面概念的由来，呵呵。AOP就是为了解决上述问题而出现的。
 
2. 代理模式

为什么要提代理模式。因为AOP的广泛实现都是通过动态代理，而动态代理又不得不说代理模式。
代理模式，顾名思义，就是对一个类的访问，变为访问这个类的代理人。经由代理再访问这个类。（代理与被代理的类实现了相同的接口，因此客户感觉不到通过代理访问这个类和直接访问这个类的区别）
为什么需要代理呢，因为一个良好的设计不应该轻易的修改。这正是开闭原则的体现：一个良好的设计应该对修改关闭，对扩展开放。而代理正是为了扩展类而存在的。他可以控制对现有类（就是需要被代理的类）服务的访问，通俗的解释就是 可以拦截对于现有类方法的调用并做些处理。
而动态代理，是指在运行期动态的为指定的类生成其代理类。（需要相关的运行时编译技术，）
AOP如Spring的AOP实现就是以这种方式实现的。他使用动态生成的代理类拦截了现有类的“切点”。并进行控制，使得这些切面的逻辑完全与该类脱离，实现了关注点分离。
下面附上我用Javassist实现的简单动态代理。（Javassist是一个运行时编译库，他能动态的生成或修改类的字节码，类似的有ASM和CGLIB，大多数框架就是基于后者实现的）

---  
### 代理类 DProxy
   package reflect.aop;
   import javassist.CannotCompileException;
   import javassist.ClassPool;
   import javassist.CtClass;
   import javassist.CtConstructor;
   import javassist.CtField;
   import javassist.CtMethod;
   import javassist.NotFoundException;

   public class DProxy {
    
    //动态生成的代理类名前缀 prefix name for Proxy
    private static final String PROXY_CLASS_NAME = ".Gproxy$";
    //代理类名索引 用于标示一个唯一的代理类（具体的代理类名为Gproxy$n） index for generate a unique proxy class
    private static int proxyIndex = 1;
    //代理拦截器(利用继承减少动态构造的字节码) Proxy interceptor(desingn for inherit)
    protected Interceptor interceptor;
    // Prohibit instantiation 利用私有构造函数阻止该类实例化
    private DProxy() {
    };

    protected DProxy(Interceptor interceptor) {
     this.interceptor = interceptor;
    }

      /**
       * 创建动态代理的工厂方法 static factory method for create proxy
       * 
       * @param targetClass
       *            :被代理的类型
       * @param interceptor
       *            拦截器实例
       * @return 返回动态代理实例 它实现了targerClass的所有接口。 因此可以向上转型为这些之中的任意接口
       */
    public static Object createProxy(Class<?> targetClass, Interceptor interceptor) {
     int index = 0;
     /* 获得运行时类的上下文 */
     ClassPool pool = ClassPool.getDefault();
     /* 动态创建代理类 */
     CtClass proxy = pool.makeClass(targetClass.getPackage().getName() + PROXY_CLASS_NAME + proxyIndex++);

     try {
      /* 获得DProxy类作为代理类的父类 */
      CtClass superclass = pool.get("reflect.aop.DProxy");
      proxy.setSuperclass(superclass);
      /* 获得被代理类的所有接口 */
      CtClass[] interfaces = pool.get(targetClass.getName()).getInterfaces();
      for (CtClass i : interfaces) {
       /* 动态代理实现这些接口 */
       proxy.addInterface(i);
       /* 获得结构中的所有方法 */
       CtMethod[] methods = i.getDeclaredMethods();
       for (int n = 0; n < methods.length; n++) {
        CtMethod m = methods[n];
        /* 构造这些Method参数 以便传递给拦截器的interceptor方法 */
        StringBuilder fields = new StringBuilder();
        fields.append("private static java.lang.reflect.Method method" + index);
        fields.append("=Class.forName(\"");
        fields.append(i.getName());
        fields.append("\").getDeclaredMethods()[");
        fields.append(n);
        fields.append("];");
        /* 动态编译之 */
        CtField cf = CtField.make(fields.toString(), proxy);
        proxy.addField(cf);
        GenerateMethods(pool, proxy, m, index);
        index++;
       }
      }
      /* 创建构造方法以便注入拦截器 */
      CtConstructor cc = new CtConstructor(new CtClass[] { pool.get("reflect.aop.Interceptor") }, proxy);
      cc.setBody("{super($1);}");
      proxy.addConstructor(cc);
      // proxy.writeFile();
      return proxy.toClass().getConstructor(Interceptor.class).newInstance(interceptor);
     } catch (Exception e) {
      e.printStackTrace();
      return null;
     }
    }

    /**
     * 动态生成生成方法实现（内部调用）
     */
    private static void GenerateMethods(ClassPool pool, CtClass proxy, CtMethod method, int index) {

     try {
      CtMethod cm = new CtMethod(method.getReturnType(), method.getName(), method.getParameterTypes(), proxy);
      /* 构造方法体 */
      StringBuilder mbody = new StringBuilder();
      mbody.append("{super.interceptor.intercept(this,method");
      mbody.append(index);
      mbody.append(",$args);}");
      cm.setBody(mbody.toString());
      proxy.addMethod(cm);
     } catch (CannotCompileException e) {
      e.printStackTrace();
     } catch (NotFoundException e) {
      e.printStackTrace();
     }
    }
   }
  
  ---  
  ###接口 Interface
   package reflect.aop;
    public interface Interface {
     void Action(int a);  
    }
   }
 
 ---  
  ###接口实现
  package reflect.aop;
   public class clazz implements Interface{
    @Override  
    public void Action(int a) {  
    System.out.println("do Action"+a);  
    }  
   }

---  
 ###拦截接口
 package reflect.aop;

 import java.lang.reflect.Method;

 public interface Interceptor {
  int intercept(Object instance, Method method, Object[] Args);
 }
 
 ---  
  ###拦截实现
  package reflect.aop;

  import java.lang.reflect.InvocationTargetException;
  import java.lang.reflect.Method;



  public class MyInterceptor implements Interceptor {
   Object proxyed;

   public MyInterceptor(Object i) {
    proxyed = i;
   }

   @Override
   public int intercept(Object instance, Method method, Object[] Args) {
    try {
     System.out.println("before action");
     method.invoke(this.proxyed, Args);
     System.out.println("after action");
    } catch (IllegalArgumentException e) {
     e.printStackTrace();
    } catch (IllegalAccessException e) {
     e.printStackTrace();
    } catch (InvocationTargetException e) {
     e.printStackTrace();
    }
    return 0;
   }
  }
  
  
 ---  
 ###测试类
  package reflect.aop;
  public class Test {

   public static void main(String[] args) {
    clazz c=new clazz();  
    Interface i=(Interface)DProxy.createProxy(clazz.class, new MyInterceptor(c));  
    i.Action(1234);  
   }

  }
 
 输出
 >before action
  do Action1234
  after action
 
 ----
 
 插入source 特殊字符
 方法的特殊变量说明：
$0, $1, $2, ...	this and actual parameters
$args	An array of parameters. The type of $args is Object[].
$$	All actual parameters.For example, m($$) is equivalent to m($1,$2,...)
$cflow(...)	cflow variable
$r	The result type. It is used in a cast expression.
$w	The wrapper type. It is used in a cast expression.
$_	The resulting value
$sig	An array of java.lang.Class objects representing the formal parameter types
$type	A java.lang.Class object representing the formal result type.
$class	A java.lang.Class object representing the class currently edited.

 
