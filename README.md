# Android-Plugin-Framework


此项目是Android插件框架完整源码以及实例。用来开发Android插件APK，并通过动态加载的方式在宿主程序中运行。

若插件APK是完全独立的APK，那么插件apk也可独立安装运行。
若插件APK不是完全独立的apk，比如和插件宿主程序共用一些依赖库，那么插件apk只能在宿主程序中运行。不可独立运行。
因为此时插件apk的代码是不完整的。

目录结构说明：

  PluginCore工程是插件库核心工程。用于提供对插件功能的支持。

  PluginMain是用来测试的插件宿主程序Demo工程。

  PluginShareLib是用来测试的插件宿主程序的依赖库Demo工程

  PluginTest是用来测试的插件Demo工程。此工程下有用来编译插件的ant脚本。

宿主程序工程可以通过ant编译或者导入eclipse后直接点击Run菜单进行安装。

插件Demo工程需要通过插件ant脚本编译。编译命令为 “ant clean debug” 原因是Demo中引用了宿主程序的依赖库。需要在编译时对共享库进行排除。
插件编译出来以后，可以将插件复制到sdcard，然后在宿主程序中调用PluginLoader.installPlugin("插件apk绝对路径")进行安装

还有一种简易的安装方式，是使用编译命令为 “ant clean debug install” 直接将插件apk安装到系统中，PluginMain工程会监听系统的应用安装广播，监听到插件apk安装广播后，
再自动调用PluginLoader.installPlugin("/data/app/插件apk文件.apk")进行插件安装。免去复制到sdcard的过程。


（虽然没有用过apkplug、以及另外一个插件框架作者singwhatiwanna写的DL框架，但是看过他们的一些介绍文档，感觉自己的这份实现应该是更简单易用更完善。。。哈哈，是不是有王婆卖瓜的嫌疑。）


# 已支持的功能：
  1、插件apk无需安装，由宿主程序动态加载运行。
  
  2、插件形式支持fragment和activity代理。
     这两种形式是插件开发中的两种主要形式。
     
  3、插件支持activity非代理模式，真正实现Activity无需注册，既不用反射，也不用代理，实现Activity完整生命周期。
     
  4、插件库apk可无任何特殊代码。如插件中的fragment，activity等无需继承任何特定基类或接口。完全和普通app代码没有区别.
  
  5、支持插件共用宿主程序依赖库提供的自定义控件
  
  6、支持插件中使用自定义控件
  
  7、支持插件资源和宿主资源混合使用。意即支持如下场景:
  
     插件程序和宿主程序共用依赖库时插件中的布局文件中使用了宿主程序中的自定义控件，而宿主程序中的自定义控件中又使用
     了宿主程序中的布局文件。
     插件代码中无需做任何特殊处理，即可支持这种布局文件。
     
  8、插件中的各种资源、布局、R、以及宿主程序中的各种资源、布局、R等可随意使用、也无任何特殊代码。
  
  10、支持插件使用宿主程序主题和系统主题
  
  11、解决资源id冲突问题。尝试做过插件开发的同学应该都遇到，插件资源id和宿主程序资源id可能相同，导致获取的资源不是想要的资源。
     此问题其实在android提供的编译器aapt中早已提供了支持。
  
  12、需要关注PluginTest工程的ant.properties文件和project.properties文件以及custom_rules.xml文件，插件使用宿主程序共享库，以及共享库R引用，和编译时排除的功能，都在这3个配置文件中体现
  
  13、支持使用插件中定义的主题以及style(控件style暂不支持5.x)。
  
# 暂不支持的功能：

  1、插件中定义的控件的style 暂不支持5.x
  
  2、支持在插件中通过R文件使用宿主程序中的资源，暂不支持插件资源文件中直接使用宿主程序中的资源。但是支持间接使用。
     例如在上述“已支持的功能”6中描述的,实际就是间接使用。
  
# 后续需要解决的问题：
  1、解决暂不支持的两点问题
  
  2、使支持插件资源文件中直接引用依赖库中的资源。目测可能需要重写android自带的aapt程序。
  

# 实现原理简介：
  1、插件apk的class
  
     通过构造插件apk的Dexclassloader来加载插件apk中的类。DexClassLoader的parent设置为宿主程序的classloader，即可将主程序和插件程序的class贯通
  
  2、插件apk的资源
  
     通过构造插件apk的AssetManager和Resouce类即可。
     
     本项目最关键一点功能、也是和网上其他插件项目不同的地方之一，在于
     通过addAssetsPath方法添加资源的时候，同时添加了插件程序的资源文件和宿主程序的资源。这样就     可以做到插件资源合并。很多资源文件都迎刃而解。
  
  3、插件apk中的资源id
  
    完成上述第二点以后，还有需要解决的难题，是宿主程序资源id和插件程序id重复的问题。
    这个问题解决办法也很简单
    我们知道，资源id是在编译时生成的，其生成的规则是0xPPTTNNNN
    PP段，是用来标记apk的，默认情况下系统资源PP是01，应用程序的PP是07
    TT段，是用来标记资源类型的，比如图标、布局等，相同的类型TT值相同，但是同一个TT值不代表同一种资源，例如这次编译的时候可能使用03作为layout的TT，那下次编译的时候可能会使用06作为TT的值，具体使用那个值，实际上和当前APP使用的资源类型的个数是相关联的。
    NNNN则是某种资源类型的资源id，默认从1开始，依次累加。
    
    那么我们要解决资源id问题，就可从TT的值开始入手，只要将每次编译时的TT值固定，即可是资源id达到分组的效果，从而避免重复。
    例如将宿主程序的layout资源的TT固定为03，将插件程序资源的layout的TT值固定为23,即可解决资源id重复的问题了。
    
    固定资源idTT值的版本也非常简单，提供一份public.xml，在public.xml中指定什么资源类型以什么TT值开头即可。
    具体public.xml如何编写，可参考PluginMain/res/values/public.xml 以及 PluginTest/res/values/public.xml俩个文件，它们是分别用来固定宿主程序和插件程序资源id的范围的。
    
    
    
  4、插件apk的Context
  
    构造一个Context即可，具体的Context实现请参考PluginCore/src/com/plugin/core/PluginContextTheme.java
    关键是要重写几个获取资源、主题的方法，以及重写getClassLoader方法

  5、插件中的LayoutInfalter
  
     通过第4步构造出来的Context获取LayoutInfater即可
     
     
  6、如何实现插件代码不依赖任何特殊代码，如继承特定的基类、接口等。
  
    要做到这一点，最主要的是实现上述第4步，构造出插件的Context后，所有问题都可以得到解决。
    
  7、插件中Activity无需注册，拥有完整生命周期是如何实现的。
    
    众所周知、Activity是系统组件，必须在manifest中注册才能被系统唤起并拥有完整生命周期，通过Activity代理和放射的方式实现的
    实际是伪生命周期。并非完整生命周期。那么如果才能实现activity免注册，而且拥有完整的生命周期呢，这要从activity的启动流程中
    入手。
    
    App安装时，系统会扫描app的Manifest并缓存到一个xml中，activity启动时，系统会现在查找缓存的xml，如果查到了，再通过classLoad去load这个class，并构造一个activity实例。那么我们只需要将classload加载这个class的时候做一个简单的映射，让系统以为加载的是A class，而实际上加载的是B class，达到挂羊头买狗肉的效果，即可将预注册的Aclass替换为未注册的activity，从而实现插件中的Activity
    完全被系统接管，而拥有完整生命周期。
    
    在PluginMain和PluginTest已经添加了这种实现方式的测试实例。
    
    
  8、通过activity代理方式实现加载插件中的activity是如何实现的
  
     要实现这一点，同样是基于上述第4点，构造出插件的Context后，通过attachBaseContext的方式，替换代理Activiyt的context即可。
     另外还需要在获得插件Activity对象后，通过反射给Activity的attach()方法中attach的成员变量赋值。
     这样可解决另外一个插件框架作者singwhatiwanna实现的代码中所谓this和that的问题。也是可以使插件Activity不需要继承任何特定基类或者接口的原因。
     
     activity代理方式不建议使用，完全可以用上述第7点替代。因为代理方式可能会有兼容问题
  
  9、activity代理实现后，其他组件，如service等，如法炮制即可。
  
  
  10、插件编译问题。
  
     如果插件和宿主共享依赖库，那边编译插件的时候不可将共享库编译到插件当中，包括共享库的代码以及R文件，但是需要在编译时添加到classpath中，且插件中如果要使用共享依赖库中的资源，需要使用共享库的R文件来进行引用。这几点在PluginTest示例工程中有体现。
     
  
  11、插件开发模式
    插件UI可通过fragment或者activity来实现
    如果是activity实现的插件，前面介绍了，分两种模式。自由模式和代理模式
    
    如果是fragment实现的插件，又分为两种
    1种是fragment运行在任意支持fragment的activity中，这种方式，在开发fragment的时候，fragmeng中凡是要使用context的地方，都需要使用通过PluginLoader.getPluginContext()获取的context，那么这种fragment对其运行容器没有特殊要求
    
    还有1种是，fragment运行在PluginCore提供的PluginSpecDisplayer中，这种方式，由于其运行容器PluginSpecDisplayer的Context已经被PluginLoader.getPluginContext获取的context替换，因此这种fragment的代码和普通非插件开发时开发的fragment的代码没有任何区别。
    
    
  12、插件主题
    轻松支持插件主题。实现原理基于上述第2点。
  
# 需要注意的问题
  项目插件开发后，特别需要注意的是宿主程序混淆问题。宿主程序混淆后，可能会导致插件程序运行时出现classnotfound
     
联系作者：
  QQ：15871365851 添加时请注明插件开发。
