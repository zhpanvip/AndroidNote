LayoutInflater是一个布局渲染工具，作用是根据xml布局文件构建View树。使用方式如下：



```java
View itemView = LayoutInflater.from(context).inflate(R.layout.item_view,container,false);
```

通过LayoutInflater.from静态方法获取LayoutInflater实例。源码如下：

```java
public static LayoutInflater from(Context context) {
    LayoutInflater LayoutInflater =
            (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    if (LayoutInflater == null) {
        throw new AssertionError("LayoutInflater not found.");
    }
    return LayoutInflater;
}
```

可以看到这里通过context.getSystemService去获取一个类型为LAYOUT_INFLATER_SERVICE的服务。这个服务是什么呢？

## LayoutInflater 服务是什么？

上文中获取的这个LayoutInflater服务与AMS、WMS等服务不同，它是APP端子机虚拟的一个服务。其主要作用是，在本地调用创建PhoneLayoutInflater工具对象。在SystemServiceRegistry的静态代码块中有如下代码：

```java
final class SystemServiceRegistry {
      static {
            registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
                new CachedServiceFetcher<LayoutInflater>() {
            @Override
            public LayoutInflater createService(ContextImpl ctx) {
                return new PhoneLayoutInflater(ctx.getOuterContext());
            }});
      }
}
```

也就是说，这里获取服务实际上是一个PhoneLayoutInflater对象。

## LayoutInflater 构建View的过程

LayoutInflater 构建 View 是从 inflate 方法开始的，在 LayoutInflater 中有多个 inflate 的重载方法，最终都会调用下面这个方法：

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();

    // ...

    // 开启预编译，这里不讨论这种情况
    View view = tryInflatePrecompiled(resource, res, root, attachToRoot);
    if (view != null) {
        return view;
    }
    // 通过 Resource 得到布局对应的XmlResourceParser
    XmlResourceParser parser = res.getLayout(resource);
    try {
        // 继续调用重载的 inflate 方法
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
```

这里的 XmlResourceParser 是一个包含了 xml 文件信息的对象，Resource 的 getLayout 方法会通过 AssetManager 的一个 native 方法 loadResourceValue 去加载 xml 文件信息。最终这个xml文件会通过getResTable 从APK包的resources.arsc中取出，并最终封装成XmlResourceParser。这个过程不是重点内容，不去深究。

接下来调用的 inflate 方法会根据 XmlResourceParser 来创建 View。

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
				// ... 省略了 Debug 相关代码

        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        View result = root;

        try {
            advanceToRootNode(parser);
            final String name = parser.getName();

						// 处理 merge 标签
            if (TAG_MERGE.equals(name)) {
     						// ...
                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // 通过 createViewFromTag 生成 xml 的 Root View
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ViewGroup.LayoutParams params = null;

                if (root != null) {
                    // 如果提供了 root ,则创建跟随 root 的 layout params
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        // 如果不 attachToRoot 的话，则为这个View设置 LayoutParams
                        temp.setLayoutParams(params);
                    }
                }

                // 生成子 View
                rInflateChildren(parser, temp, attrs, true);

                // root 不为null，并且 attachToRoot 为 true，则将 temp 添加到 root
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }

        }
        // ... 省略 catch 相关代码

        return result;
    }
}
```

代码中注释写的比较详细，这里需要注意一下的是 createViewFromTag 这个方法。这个方法在后边代码中还会出现，所以这里先不做解读。接着看 rInflateChildren 方法的代码：

```java
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
        boolean finishInflate) throws XmlPullParserException, IOException {
    rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
}
```

只是调用了 rInflate 方法，rInflate源码如下：

```java
void rInflate(XmlPullParser parser, View parent, Context context,
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

    final int depth = parser.getDepth();
    int type;
    boolean pendingRequestFocus = false;

    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

        if (type != XmlPullParser.START_TAG) {
            continue;
        }

        final String name = parser.getName();

        if (TAG_REQUEST_FOCUS.equals(name)) {
            pendingRequestFocus = true;
            consumeChildElements(parser);
        } else if (TAG_TAG.equals(name)) {
            parseViewTag(parser, parent, attrs);
        } else if (TAG_INCLUDE.equals(name)) { // 处理 include 标签
            if (parser.getDepth() == 0) {
                throw new InflateException("<include /> cannot be the root element");
            }
            parseInclude(parser, context, parent, attrs);
        } else if (TAG_MERGE.equals(name)) { // 处理 merge 标签
            throw new InflateException("<merge /> must be the root element");
        } else { // 解析 View
            final View view = createViewFromTag(parent, name, context, attrs);
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
            rInflateChildren(parser, view, attrs, true);
            viewGroup.addView(view, params);
        }
    }

    if (pendingRequestFocus) {
        parent.restoreDefaultFocus();
    }

    if (finishInflate) {
        parent.onFinishInflate();
    }
}
```

可以看到在这个 rInflate 方法中会对 include 标签、merge 标签等进行处理。重点关注的还是 else 中的逻辑，可以看到又是通过 createViewFromTag 来得到一个View，接着为这个 View 设置 LayoutParams,然后又调用了 rInflateChildren。显然这里就是一个递归调用，来不断解析xml文件中的View。

到这里重点就应该看一下 View 到底是如何生成的了，看下 createViewFromTag 方法。

```java
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {

   // ...

    try {
        View view = tryCreateView(parent, name, context, attrs);

        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
                if (-1 == name.indexOf('.')) {
                    view = onCreateView(context, parent, name, attrs);
                } else {
                    view = createView(context, name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }

        return view;
    } catch (InflateException e) {
        throw e;

    } catch (ClassNotFoundException e) {
        final InflateException ie = new InflateException(
                getParserStateDescription(context, attrs)
                + ": Error inflating class " + name, e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;

    } catch (Exception e) {
        final InflateException ie = new InflateException(
                getParserStateDescription(context, attrs)
                + ": Error inflating class " + name, e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;
    }
}
```

首先通过 tryCreateView 来尝试创建 View，源码如下：

```java
public final View tryCreateView(@Nullable View parent, @NonNull String name,
    @NonNull Context context,
    @NonNull AttributeSet attrs) {
    // 如果是一个 blink 标签，则创建一个 BlinkLayout，这个布局会没间隔 500ms 闪烁一次。
    if (name.equals(TAG_1995)) {
        // Let's party like it's 1995!
        return new BlinkLayout(context, attrs);
    }

    View view;
    if (mFactory2 != null) {
       // 可以看到，这里如果 Factory2 不为空则使用 Factory2 的 Factory 来创建View
        view = mFactory2.onCreateView(parent, name, context, attrs);
    } else if (mFactory != null) {
        // 如果 mFactory2 为null，但是 mFactory 不为 null，则通过Factory的Factory创建View
        view = mFactory.onCreateView(name, context, attrs);
    } else {
        view = null;
    }

    if (view == null && mPrivateFactory != null) {
       // 使用私有的mPrivateFactory创建View
        view = mPrivateFactory.onCreateView(parent, name, context, attrs);
    }

    return view;
}
```

但是，这里在没有设置 Factory2 或 Factory 的时候这两个值都是空，所有这里先不讨论这一情况。接着看 createViewFromTag 这个方法，可以看到，接下来调用了 onCreateView 来创建 View，而 onCreateView 最终调用了 createView，代码如下：



```java 
public final View createView(@NonNull Context viewContext, @NonNull String name,
        @Nullable String prefix, @Nullable AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
    // ...

    // 从 sConstructorMap 缓存中查找 name 对应的构造方法
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    if (constructor != null && !verifyClassLoader(constructor)) {
        constructor = null;
        sConstructorMap.remove(name);
    }
    Class<? extends View> clazz = null;

    try {
        if (constructor == null) {
            // 缓存中没有找到 name 对应的构造方法，则尝试获取 name 对应的Class对象
            clazz = Class.forName(prefix != null ? (prefix + name) : name, false,
                    mContext.getClassLoader()).asSubclass(View.class);

            if (mFilter != null && clazz != null) {
                boolean allowed = mFilter.onLoadClass(clazz);
                if (!allowed) {
                    failNotAllowed(name, prefix, viewContext, attrs);
                }
            }
            // 获取构造方法
            constructor = clazz.getConstructor(mConstructorSignature);
            constructor.setAccessible(true);
            // 将构造方法放入缓存
            sConstructorMap.put(name, constructor);
        } else {
           // ...
        }

       // ...

        try {
            // 通过构造方法实例化 View 对象
            final View view = constructor.newInstance(args);
            if (view instanceof ViewStub) {
                // Use the same context when inflating ViewStub later.
                final ViewStub viewStub = (ViewStub) view;
                viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
            }
            return view;
        } finally {
            mConstructorArgs[0] = lastContext;
        }
    }
    // ... 省略 catch 相关代码
}
```

可以看到，在createView方法中，首先会从 sConstructorMap 中查找 name对应的构造方法，如果没有找到，则获取 name对应的Class类对象，然后获取构造方法，最终，通过构造方法反射实例化出了View对象。

## Factory 与 Factory2

上一章提到的 tryCreateView 方法中首先判断了Factory2是否为空，不为空则使用 Factory2 的 onCreateView方法来创建View，如果 Factory2 为空，接着又尝试使用 Factory 来创建View。如果经过以上代码 View 依然为空，才会使用反射创建View。那么，Factory 与 Factory2 是什么？它们又是在什么时候赋值的？

在LayoutInflater源码中可以看到Factory是一个接口，且其内部只有一个onCreateView方法，源码如下：

```java
public interface Factory {
    /**
     * Hook you can supply that is called when inflating from a LayoutInflater.
     * You can use this to customize the tag names available in your XML
     * layout files.
     *
     * <p>
     * Note that it is good practice to prefix these custom names with your
     * package (i.e., com.coolcompany.apps) to avoid conflicts with system
     * names.
     *
     * @param name Tag name to be inflated.
     * @param context The context the view is being created in.
     * @param attrs Inflation attributes as specified in XML file.
     *
     * @return View Newly created view. Return null for the default
     *         behavior.
     */
    @Nullable
    View onCreateView(@NonNull String name, @NonNull Context context,
            @NonNull AttributeSet attrs);
}
```

从onCreateView的注释中可以看出来，它是一个在LayoutInflater inflate View时的一个hook点，可以使用它来自定义xml中的标签。如果不理解这句话，可以一下[BackgroundLibrary](https://github.com/JavaNoober/BackgroundLibrary)这个库。

而Factory2 则是一个继承了Factory的接口，它的代码如下：

```java
public interface Factory2 extends Factory {
    /**
     * Version of {@link #onCreateView(String, Context, AttributeSet)}
     * that also supplies the parent that the view created view will be
     * placed in.
     *
     * @param parent The parent that the created view will be placed
     * in; <em>note that this may be null</em>.
     * @param name Tag name to be inflated.
     * @param context The context the view is being created in.
     * @param attrs Inflation attributes as specified in XML file.
     *
     * @return View Newly created view. Return null for the default
     *         behavior.
     */
    @Nullable
    View onCreateView(@Nullable View parent, @NonNull String name,
            @NonNull Context context, @NonNull AttributeSet attrs);
}
```

Factory2 继承了 Factory，且 Factory2 中新增了一个 onCreateView 方法，这个方法与Factory 的 onCreateView相比多了一个parent 的参数。因此，如果需要使用parent的话，可以使用Factory2这个接口。

Factory 与 Factory2 是在哪里被赋值的？翻下LayoutInflater 的代码，可以看到如下：

```java 
public void setFactory(Factory factory) {
    if (mFactorySet) {
        throw new IllegalStateException("A factory has already been set on this LayoutInflater");
    }
    if (factory == null) {
        throw new NullPointerException("Given factory can not be null");
    }
    mFactorySet = true;
    if (mFactory == null) {
        mFactory = factory;
    } else {
        mFactory = new FactoryMerger(factory, null, mFactory, mFactory2);
    }
}

/**
 * Like {@link #setFactory}, but allows you to set a {@link Factory2}
 * interface.
 */
public void setFactory2(Factory2 factory) {
    if (mFactorySet) {
        throw new IllegalStateException("A factory has already been set on this LayoutInflater");
    }
    if (factory == null) {
        throw new NullPointerException("Given factory can not be null");
    }
    mFactorySet = true;
    if (mFactory == null) {
        mFactory = mFactory2 = factory;
    } else {
        mFactory = mFactory2 = new FactoryMerger(factory, factory, mFactory, mFactory2);
    }
}
```

LayoutInflater 中提供了设置 Factory 与Factory2 的方法。

那么这两个接口除了上边提到的用它来自定义xml中的标签外，还有其他用途吗？答案是确定的。我们可以使用Factory2来实现XML文件中的View替换，还可以使用Factory2实现换肤框架。

其实，在AppCompatActivity中已经默认为LayoutInflater设置了Factory2，来看AppCompatActivity的onCreate方法：

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    final AppCompatDelegate delegate = getDelegate();
    delegate.installViewFactory();
    delegate.onCreate(savedInstanceState);
    super.onCreate(savedInstanceState);
}
```

 这里先获取了一个AppCompatDelegate的实例，然后调用了它的 installViewFactory方法，这个方法的实现是在AppCompatDelegateImpl中，AppCompatDelegateImpl实现了Factory2接口：

```java
class AppCompatDelegateImpl extends AppCompatDelegate
        implements MenuBuilder.Callback, LayoutInflater.Factory2 {
 // ...
  public void installViewFactory() {
      LayoutInflater layoutInflater = LayoutInflater.from(mContext);
      if (layoutInflater.getFactory() == null) {
          LayoutInflaterCompat.setFactory2(layoutInflater, this);
      } else {
          if (!(layoutInflater.getFactory2() instanceof AppCompatDelegateImpl)) {
              Log.i(TAG, "The Activity's LayoutInflater already has a Factory installed"
                      + " so we can not install AppCompat's");
          }
      }
  }
}
```

在 installViewFactory 方法中通过LayoutInflaterCompat的 setFactory2方法为LayoutInflater设置了Factory2，即将AppCompatDelegateImpl设置到了LayoutInflater中。



```java 
class AppCompatDelegateImpl extends AppCompatDelegate
        implements MenuBuilder.Callback, LayoutInflater.Factory2 {

  // ...

  @Override
  public final View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
      return createView(parent, name, context, attrs);
  }

   @Override
      public View createView(View parent, final String name, @NonNull Context context,
              @NonNull AttributeSet attrs) {
          if (mAppCompatViewInflater == null) {
              TypedArray a = mContext.obtainStyledAttributes(R.styleable.AppCompatTheme);
              String viewInflaterClassName =
                      a.getString(R.styleable.AppCompatTheme_viewInflaterClass);
              if ((viewInflaterClassName == null)
                      || AppCompatViewInflater.class.getName().equals(viewInflaterClassName)) {
                  // 初始化AppCompatViewInflater
                  mAppCompatViewInflater = new AppCompatViewInflater();
              } else {
                  try {
                      // 初始化AppCompatViewInflater
                      Class<?> viewInflaterClass = Class.forName(viewInflaterClassName);
                      mAppCompatViewInflater =
                              (AppCompatViewInflater) viewInflaterClass.getDeclaredConstructor()
                                      .newInstance();
                  } catch (Throwable t) {
                       // 初始化AppCompatViewInflater
                      mAppCompatViewInflater = new AppCompatViewInflater();
                  }
              }
          }

          boolean inheritContext = false;
					// ...

					// 调用AppCompatViewInflater 的 createView
          return mAppCompatViewInflater.createView(parent, name, context, attrs, inheritContext,
                  IS_PRE_LOLLIPOP, /* Only read android:theme pre-L (L+ handles this anyway) */
                  true, /* Read read app:theme as a fallback at all times for legacy reasons */
                  VectorEnabledTintResources.shouldBeUsed() /* Only tint wrap the context if enabled */
          );
      }
}
```

AppCompatDelegateImpl 中重写了Factory2 的onCreateView方法，并调用了createView方法。而createView方法中的核心就是实例化了AppCompatViewInflater，接着调用了AppCompatViewInflater的createView方法。

AppCompatViewInflater的createView做了什么？

```java
final View createView(View parent, final String name, @NonNull Context context,
        @NonNull AttributeSet attrs, boolean inheritContext,
        boolean readAndroidTheme, boolean readAppTheme, boolean wrapContext) {
    final Context originalContext = context;

    // We can emulate Lollipop's android:theme attribute propagating down the view hierarchy
    // by using the parent's context
    if (inheritContext && parent != null) {
        context = parent.getContext();
    }
    if (readAndroidTheme || readAppTheme) {
        // We then apply the theme on the context, if specified
        context = themifyContext(context, attrs, readAndroidTheme, readAppTheme);
    }
    if (wrapContext) {
        context = TintContextWrapper.wrap(context);
    }

    View view = null;

    // We need to 'inject' our tint aware Views in place of the standard framework versions
    switch (name) {
        case "TextView":
            view = createTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "ImageView":
            view = createImageView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "Button":
            view = createButton(context, attrs);
            verifyNotNull(view, name);
            break;
        case "EditText":
            view = createEditText(context, attrs);
            verifyNotNull(view, name);
            break;
        case "Spinner":
            view = createSpinner(context, attrs);
            verifyNotNull(view, name);
            break;
        case "ImageButton":
            view = createImageButton(context, attrs);
            verifyNotNull(view, name);
            break;
        case "CheckBox":
            view = createCheckBox(context, attrs);
            verifyNotNull(view, name);
            break;
        case "RadioButton":
            view = createRadioButton(context, attrs);
            verifyNotNull(view, name);
            break;
        case "CheckedTextView":
            view = createCheckedTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "AutoCompleteTextView":
            view = createAutoCompleteTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "MultiAutoCompleteTextView":
            view = createMultiAutoCompleteTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "RatingBar":
            view = createRatingBar(context, attrs);
            verifyNotNull(view, name);
            break;
        case "SeekBar":
            view = createSeekBar(context, attrs);
            verifyNotNull(view, name);
            break;
        case "ToggleButton":
            view = createToggleButton(context, attrs);
            verifyNotNull(view, name);
            break;
        default:
            // The fallback that allows extending class to take over view inflation
            // for other tags. Note that we don't check that the result is not-null.
            // That allows the custom inflater path to fall back on the default one
            // later in this method.
            view = createView(context, name, attrs);
    }

    if (view == null && originalContext != context) {
        // If the original context does not equal our themed context, then we need to manually
        // inflate it using the name so that android:theme takes effect.
        view = createViewFromTag(context, name, attrs);
    }

    if (view != null) {
        // If we have created a view, check its android:onClick
        checkOnClickListener(view, attrs);
    }

    return view;
}

protected AppCompatTextView createTextView(Context context, AttributeSet attrs) {
    return new AppCompatTextView(context, attrs);
}

@NonNull
protected AppCompatImageView createImageView(Context context, AttributeSet attrs) {
    return new AppCompatImageView(context, attrs);
}

// ...
```



可以看到，如果我们在xml中声明的是ImageView/TextView,则会被替换成AppCompatImageView/AppCompatTextView。也就是说AppCompactActivity通过Factory2接口帮我们实现了兼容版本的View。




参考：

https://www.jianshu.com/p/36f6b83fd6de

https://juejin.cn/post/6844903697353277447