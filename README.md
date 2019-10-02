## 1.尝试加载xposed的类,如果能加载则表示已经安装了。

XposedHelpers类中存在fieldCache methodCache constructorCache 这三个静态成员，都是hashmap类型，凡是需要被hook的且已经被找到的对象都会被缓存到这三个map里面。
我们通过便利这三个map来找到相关hook信息。
备注:方法a是检测xposed到底改了什么东西存放到a中。抖音似乎会收集相关信息并上报。

[Java] 纯文本查看 复制代码

```java
    public void b()
	{
		try
		{
			Object localObject = ClassLoader.getSystemClassLoader()
                .loadClass("de.robv.android.xposed.XposedHelpers").newInstance();
			// 如果加载类失败 则表示当前环境没有xposed 
			if (localObject != null)
			{
				a(localObject, "fieldCache");
				a(localObject, "methodCache");
				a(localObject, "constructorCache");
			}
			return;
		}
		catch (Throwable localThrowable)
		{}
	}

	private void a(Object arg5, String arg6)
	{
		try
		{
			// 从XposedHelpers中读取相关的hook信息
			Field v0_1 = arg5.getClass().getDeclaredField(arg6);
			v0_1.setAccessible(true);
			Set v0_2 = v0_1.get(arg5).keySet();
			if (v0_2 == null)
			{
				return;
			}
			if (v0_2.isEmpty())
			{
				return;
			}
			Iterator v1 = v0_2.iterator();
			// 排除无关紧要的类
			while (v1.hasNext())
			{
				Object v0_3 = v1.next();
				if (v0_3 == null)
				{
					continue;
				}
				if (((String)v0_3).length() <= 0)
				{
					continue;
				}
				if (((String)v0_3).toLowerCase().startsWith("android.support"))
				{
					continue;
				}
				if (((String)v0_3).toLowerCase().startsWith("javax."))
				{
					continue;
				}
				if (((String)v0_3).toLowerCase().startsWith("android.webkit"))
				{
					continue;
				}
				if (((String)v0_3).toLowerCase().startsWith("java.util"))
				{
					continue;
				}
				if (((String)v0_3).toLowerCase().startsWith("android.widget"))
				{
					continue;
				}
				if (((String)v0_3).toLowerCase().startsWith("sun."))
				{
					continue;
				}
				this.a.add(v0_3);
			}
		}
		catch (Throwable v0)
		{
			v0.printStackTrace();
		}
	}
```
	
## 2.检测xposed相关文件

检测XposedBridge.jar这个文件和de.robv.android.xposed.XposedBridge
XposedBridge.jar存放在framework里面，de.robv.android.xposed.XposedBridge是开发xposed框架使用的主要接口。
在这个方法里读取了maps这个文件，在linux内核中，这个文件存储了进程映射了的内存区域和访问权限。在这里我们可以看到这个进程加载了那些文件。
如果进程加载了xposed相关的so库或者jar则表示xposed框架已注入。

[Java] 纯文本查看 复制代码

```java
    public static boolean a(String paramString)
	{
		try
		{
			Object localObject = new HashSet();
			// 读取maps文件信息
			BufferedReader localBufferedReader = 
                new BufferedReader(new FileReader("/proc/" + Process.myPid() + "/maps"));
			// 遍历查询关键词 反编译出来的代码可能不太准确
			for (;;)
			{
				String str = localBufferedReader.readLine();
				if (str == null)
				{
					break;
				}
				if ((str.endsWith(".so")) || (str.endsWith(".jar")))
				{
					((Set)localObject).add(str.substring(str.lastIndexOf(" ") + 1));
				}
			}
			localBufferedReader.close();
			localObject = ((Set)localObject).iterator();
			while (((Iterator)localObject).hasNext())
			{
				boolean bool = ((String)((Iterator)localObject).next()).contains(paramString);
				if (bool)
				{
					return true;
				}
			}
		}
		catch (Exception paramString)
		{}
		return false;
	}
```
	  
## 3.检测方法的调用栈

如果你的手机安装了xposed框架，那么你在查看app崩溃时候的堆栈信息，一定能在顶层找到xposed的类信息，
既然xposed想改你的信息，那么在调用栈里面肯定是有它的身影的。比如出现de.robv.android.xposed.XposedBridge这个类，方法被hook的时候调用栈里会出现de.robv.android.xposed.XposedBridge.handleHookedMethod 和de.robv.android.xposed.XposedBridge.invokeOriginalMethodNative这些xposed的方法，在dalvik.system.NativeStart.main方法后出现de.robv.android.xposed.XposedBridge.main调用

[Java] 纯文本查看 复制代码

```java
    try {
        throw new Exception("");
    } catch (Exception localException) {
        StackTraceElement[] arrayOfStackTraceElement = localException.getStackTrace();
        int m = arrayOfStackTraceElement.length;
        int i = 0;
        int j;
        // 遍历整个堆栈查询xposed相关信息
        for (int k = 0; i < m; k = j)
		{
            StackTraceElement localStackTraceElement = arrayOfStackTraceElement[i];
            j = k;
            if (localStackTraceElement.getClassName()
				.equals("com.android.internal.os.ZygoteInit"))
			{
                k += 1;
                j = k;
                if (k == 2)
				{
                    return true;
                }
            }
            if (localStackTraceElement.getClassName().equals(e))
			{
                return true;
            }
            i += 1;
        }
    }
    return false;
	}

	try
	{
		throw new Exception("");
	}
	catch(
	Exception localException)
	{
		arrayOfStackTraceElement = localException.getStackTrace();
		j = arrayOfStackTraceElement.length;
		i = 0;
	}
	for(;;)
	{
		boolean bool1 = bool2;
		if (i < j)
		{
			if (arrayOfStackTraceElement[i].getClassName()
                .equals("de.robv.android.xposed.XposedBridge"))
			{
				bool1 = true;
			}
		}
		else
		{
			return bool1;
		}
		i += 1;
	}
```
		
## 4.修改hook方法的查询结果

数组c包含了抖音里比较重要的及各类， this.a指的是检测出的被修改的方法，字段。他执行了两者的匹配，将结果存放到了一个JSONObject里面，里面包含了是否使用模拟器，系统的详细信息，是否是双开xpp，是否适用了xp，hook了那些方法等信息上报到https://xlog.snssdk.com/do/y?ver=0.4&ts=

[ HTML] 纯文本查看 复制代码

```java
    private String[] c = {"android.os.Build#SERIAL", 
        "android.os.Build#PRODUCT",
        "android.os.Build#DEVICE", 
        "android.os.Build#FINGERPRINT", 
        "android.os.Build#MODEL", 
        "android.os.Build#BOARD", 
        "android.os.Build#BRAND", 
        "android.os.Build.BOOTLOADER", 
        "android.os.Build#HARDWARE", 
        "android.os.SystemProperties#get(java.lang.String,java.lang.String)", 
        "android.os.SystemProperties#get(java.lang.String)", 
        "java.lang.System#getProperty(java.lang.String)", 
        "android.telephony.TelephonyManager#getDeviceId()", 
        "android.telephony.TelephonyManager#getSubscriberId()", 
        "android.net.wifi.WifiInfo#getMacAddress()", 
        "android.os.Debug#isDebuggerConnected()", 
        "android.app.activitymanager#isUserAMonkey()", 
        "com.ss."};

	public ArrayList<String> c()
	{
		ArrayList localArrayList = new ArrayList();
		Iterator localIterator = this.a.iterator();
		while (localIterator.hasNext())
		{
			String str1 = (String) localIterator.next();
			String[] arrayOfString = this.c;
			int j = arrayOfString.length;
			int i = 0;
			while (i < j)
			{
				if (true == str1.startsWith(arrayOfString[i]))
				{
					String str2 = str1.replace(")#exact", ")").replace(")#bestmatch", ")").replace("#", ".");
					if (str2.length() > 0)
					{
						localArrayList.add(str2);
					}
				}
				i += 1;
			}
		}
		return localArrayList;
	}
```

## 5.检测方法是否被篡改

Modifier.isNative方法判断是否是native修饰的方法，xposedhook方法的时候，会把方法转位native方法，我们检测native方法中不该为native的方法来达到检测的目的，比如getDeviceId,getMacAddress这些方法如果是native则表示已经被篡改了。

[Java] 纯文本查看 复制代码

```java
    public static boolean a(String paramString1, String paramString2, Class... paramVarArgs)
	{
		try
		{
			// 判断方法是不是用native修饰的
			boolean bool = Modifier.isNative(Class.forName(paramString1)
											 .getDeclaredMethod(paramString2, paramVarArgs).getModifiers());
			if (bool)
			{
				return true;
			}
		}
		catch (Exception paramString1)
		{
			paramString1.printStackTrace();
		}
		return false;
	}
```
	  
## 6.检测包名

这个是最简单也是最不靠谱的方法，因为只要禁止app读取应用列表就没办法了。

[Java] 纯文本查看 复制代码

```java
    if (((ApplicationInfo)localObject).packageName.equals("de.robv.android.xposed.installer"))
	{
		Log.wtf("HookDetection", "Xposed found on the system.");
	}
```
	  
## 反制xposed
	
1.通过反射重写xposed设置，直接禁用

这个方法是酷安里中抄来的。重写application，在onCreate中写入

[Java] 纯文本查看 复制代码

```java
    try {
		Field v0_1 = ClassLoader.getSystemClassLoader()
			.loadClass("de.robv.android.xposed.XposedBridge")
			.getDeclaredField("disableHooks");
		v0_1.setAccessible(true);
		v0_1.set(null, Boolean.valueOf(true));
	}
    catch(Throwable v0) {
	}
```
	  
## 未完待续。。。

博客发布页

https://blog.coderstory.cn/about-xposed/
