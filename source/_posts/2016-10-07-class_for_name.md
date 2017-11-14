---
layout: post
title: JDBC (1) - Class.forName(java.lang.String)?!
tags: [java, jdbc]
comments: true
---

JDBC Driver를 사용할 경우, Connection을 생성하기 전에 `Class.forName("Specific Vender's Driver");` 코드를 추가해서 사용할 JDBC Driver를 로드해야 한다.

얘는 뭘 하는 앨까?

호출되는 코드만 봐서는 직관적로 뭘 하는지 잘 모르겠다. 

한번 찾아보자!

### java.lang.Class

```javadoc
public final class Class<T>
extends Object
implements Serializable, GenericDeclaration, Type, AnnotatedElement
Instances of the class Class represent classes and interfaces in a running Java application. An enum is a kind of class and an annotation is a kind of interface. Every array also belongs to a class that is reflected as a Class object that is shared by all arrays with the same element type and number of dimensions. The primitive Java types (boolean, byte, char, short, int, long, float, and double), and the keyword void are also represented as Class objects.
Class has no public constructor. Instead Class objects are constructed automatically by the Java Virtual Machine as classes are loaded and by calls to the defineClass method in the class loader.

The following example uses a Class object to print the class name of an object:

     void printClassName(Object obj) {
         System.out.println("The class of " + obj +
                            " is " + obj.getClass().getName());
     }
 
It is also possible to get the Class object for a named type (or for void) using a class literal. See Section 15.8.2 of The Java™ Language Specification. For example:

System.out.println("The name of class Foo is: "+Foo.class.getName());
Since:
JDK1.0
See Also:
ClassLoader.defineClass(byte[], int, int), Serialized Form
```

Class 클래스의 인스턴스이며 러닝 타임의 클래스들과 인터페이스들을 표현한다고 한다. 그리고 primitive 타입과 void 도 Class 오브젝트로 표현된다고 한다.
*(enum은 클래스중 한 종류이고, annotation은 인터페이스의 한 종류인건 안비밀 ㅎ)*

자 그럼 forName 메소드를 한번 봐보자.

### forName(java.lang.String)

```javadoc
public static Class<?> forName(String className)
                        throws ClassNotFoundException
Returns the Class object associated with the class or interface with the given string name. Invoking this method is equivalent to:
Class.forName(className, true, currentLoader)
where currentLoader denotes the defining class loader of the current class.
For example, the following code fragment returns the runtime Class descriptor for the class named java.lang.Thread:

Class t = Class.forName("java.lang.Thread")
A call to forName("X") causes the class named X to be initialized.

Parameters:
className - the fully qualified name of the desired class.
Returns:
the Class object for the class with the specified name.
Throws:
LinkageError - if the linkage fails
ExceptionInInitializerError - if the initialization provoked by this method fails
ClassNotFoundException - if the class cannot be located
```

문자열 파라미터로 전달된 애와 연관된 클래스 혹은 인터페이스의 Class 오브젝트를 반환한다. **(Class 오브젝트를 반환하지 문자열에 해당하는 클래스를 반환하는게 아니다!)**


오브젝트를 리턴하는데... 보통 JDBC Driver를 사용할 때 Class.forName() 메소드를 따로 변수에 저장하지 않고 그냥 호출만 한다.

그냥 공중에다가 오브젝트를 뿌려버리고 마는 걸까? *(에이 설마 ㅎ)*

이부분은 JDBC Driver의 소스코드를 봐야 알 것 같다.

말 나온김에 한번 봐보자!




소스코드는 CUBRID JDBC Driver를 보기로 했다. (MySQL, Oracle 다 비슷비슷 할거 같다. 왜? 표준이란게 있으니깐)

CUBRID는 JDBC Driver를 로딩할 때 `Class.forName("cubrid.jdbc.driver.CUBRIDDriver");` 이렇게 호출한다.

즉, cubrid.jdbc.driver.CUBRIDDriver 라는 클래스를 로드하여 반환하라는 의미이다.

이때, 우리는 따로 변수에 담지 않으니깐 저게 호출되면서 무엇을 하는지를 알아봐야 할 것이다.
(소스.. 소스를 보자!)

소스코드는 [github](https://github.com/CUBRID/cubrid/tree/develop/src/jdbc)에 올라와 있으니 필요하신 분은 참고하시길...


### cubrid.jdbc.driver.CUBRIRDDriver.java
```java
public class CUBRIDDriver implements Driver {
	// version
	public static final String version_string = "@JDBC_DRIVER_VERSION_STRING@";
	public static final int major_version;
	public static final int minor_version;
	public static final int patch_version;
	static {
		StringTokenizer st = new StringTokenizer(version_string, ".");
		if (st.countTokens() != 4) {
			throw new RuntimeException("Could not parse version_string: "
					+ version_string);
		}
		major_version = Integer.parseInt(st.nextToken());
		minor_version = Integer.parseInt(st.nextToken());
		patch_version = Integer.parseInt(st.nextToken());
	}
... (생략)
	static {
		try {
			DriverManager.registerDriver(new CUBRIDDriver());
		} catch (SQLException e) {
		}
	}
```

여기서 눈여겨 볼 곳은 static 블록이다.

static 블록이 왜 중요하냐고? 클래스가 로딩 되면 static 관련 애들은 JVM의 method 영역으로 올라가게 된다.
따라서 static 이 붙은 애들은 클래스가 로딩되는 시점부터 사용할 준비가 되는 것이다.

처음 static 블록은 그냥 버전 정보를 저장할 뿐이니깐 패스하고

두 번째 static 블록을 보면 실질적으로 CUBRIDDriver 오브젝트를 등록하는 코드가 보인다.
*(이부분은 MySQL, Oracle 등 다 같을 것이다. 표준이니깐)*

그럼 여기서 registerDriver(java.sql.Driver)는 뭘 하는 애일까? 그냥 등록만 하면 끝인가? 궁금하다!

###  registerDriver(java.sql.Driver)
```javadoc
    /**
     * Registers the given driver with the {@code DriverManager}.
     * A newly-loaded driver class should call
     * the method {@code registerDriver} to make itself
     * known to the {@code DriverManager}. If the driver is currently
     * registered, no action is taken.
     *
     * @param driver the new JDBC Driver that is to be registered with the
     *               {@code DriverManager}
     * @exception SQLException if a database access error occurs
     * @exception NullPointerException if {@code driver} is null
     */
    public static synchronized void registerDriver(java.sql.Driver driver)
        throws SQLException {

        registerDriver(driver, null);
    }
```

앗, `registerDriver(driver, null);` 을 호출한다. 이걸 찾아가보자.

### registerDriver(java.sql.Driver, java.sql.DriverAction)
```java
    /**
     * Registers the given driver with the {@code DriverManager}.
     * A newly-loaded driver class should call
     * the method {@code registerDriver} to make itself
     * known to the {@code DriverManager}. If the driver is currently
     * registered, no action is taken.
     *
     * @param driver the new JDBC Driver that is to be registered with the
     *               {@code DriverManager}
     * @param da     the {@code DriverAction} implementation to be used when
     *               {@code DriverManager#deregisterDriver} is called
     * @exception SQLException if a database access error occurs
     * @exception NullPointerException if {@code driver} is null
     * @since 1.8
     */
    public static synchronized void registerDriver(java.sql.Driver driver,
            DriverAction da)
        throws SQLException {

        /* Register the driver if it has not already been added to our list */
        if(driver != null) {
            registeredDrivers.addIfAbsent(new DriverInfo(driver, da));
        } else {
            // This is for compatibility with the original DriverManager
            throw new NullPointerException();
        }

        println("registerDriver: " + driver);

    }
```

자, 보니깐 registerdDrivers 라는 변수에 새로운 드라이버 정보가 저장되는걸 볼 수 있다. *(이 변수는 thread-safe ArrayList이고, DriverInfo가 없을 경우만 add 한다)*

그럼 registerdDrivers에 저장된 변수는 언제 어떻게 사용이 될까??

곰곰히 생각해보자. Class.forName() 을 호출하고 나서 뭘 했지?!

바로 `DriverManager.getConnection("Specific vender's url");` 이런 코드를 입력했을 것이다. 바로, Connection 오브젝트를 생성하는 단계이다.

이때 과연 무슨 일이 일어날까?? *(소스... 소스를 보자!)*

### DriverManager.getConnection
```java
    //  Worker method called by the public getConnection() methods.
    private static Connection getConnection(
        String url, java.util.Properties info, Class<?> caller) throws SQLException {
        /*
         * When callerCl is null, we should check the application's
         * (which is invoking this class indirectly)
         * classloader, so that the JDBC driver class outside rt.jar
         * can be loaded from here.
         */
        ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;
        synchronized(DriverManager.class) {
            // synchronize loading of the correct classloader.
            if (callerCL == null) {
                callerCL = Thread.currentThread().getContextClassLoader();
            }
        }

        if(url == null) {
            throw new SQLException("The url cannot be null", "08001");
        }

        println("DriverManager.getConnection(\"" + url + "\")");

        // Walk through the loaded registeredDrivers attempting to make a connection.
        // Remember the first exception that gets raised so we can reraise it.
        SQLException reason = null;

        for(DriverInfo aDriver : registeredDrivers) {
            // If the caller does not have permission to load the driver then
            // skip it.
            if(isDriverAllowed(aDriver.driver, callerCL)) {
                try {
                    println("    trying " + aDriver.driver.getClass().getName());
                    Connection con = aDriver.driver.connect(url, info);
                    if (con != null) {
                        // Success!
                        println("getConnection returning " + aDriver.driver.getClass().getName());
                        return (con);
                    }
                } catch (SQLException ex) {
                    if (reason == null) {
                        reason = ex;
                    }
                }

            } else {
                println("    skipping: " + aDriver.getClass().getName());
            }

        }

        // if we got here nobody could connect.
        if (reason != null)    {
            println("getConnection failed: " + reason);
            throw reason;
        }

        println("getConnection: no suitable driver found for "+ url);
        throw new SQLException("No suitable driver found for "+ url, "08001");
    }
```

와 길다~ getConnection 메소드는 여러개로 오버로딩 되어 있으므로 가장 중요한 getConnection만 보면 될 것 같다.

중간에 잘 찾아보면 registerDriver 메소드에서 봤던 registeredDrivers 변수가 보인다.

for-loop을 돌면서 driver 오브젝트를 이용해 connect 메소드를 호출해 실질적인 Connection 오브젝트를 생성한다.

이후부터는 생성된 Connection 오브젝트를 가지고 Statement를 만들고 ResultSet도 만들고 하면서 DB에 왔다갔다 하면 된다. ㅋ
*(JDBC 4.0 스펙부터는 굳이 Class.forName()으로 클래스 로딩 안해도 된다고 한다...)*


## 결론

신기술을 배우자. 그리고 적용하자.

