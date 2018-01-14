---
categories: Spring 
tags: Spring
---
<!-- TOC -->

- [问题](#问题)
- [方案设计](#方案设计)
- [实现](#实现)
    - [Handler](#handler)
    - [Parser](#parser)
    - [BeanPostProcessor](#beanpostprocessor)
    - [核心类](#核心类)
- [使用方法](#使用方法)
- [其他](#其他)

<!-- /TOC -->


## 问题
近期项目中使用到了spring的定时任务，只需要使用 **@Scheduled** 并且简单的配置就可以启动定时任务，十分方便。
但是我们在用的时候由于对多个任务进行了抽象，希望得到的效果是由抽象类作为调度的入口，各个子实现类负责具体的任务执行，同时又希望多个任务的调度时间可以不同。举个例子，
```java
package spring;
@Component
public abstract class AbstractSchedule {

    abstract void doTask();

    @Scheduled(cron = "${cron}" )
    public void task() {
        System.out.println(this.getClass());
        doTask();
    }
}

package spring;
@Component
public class Schedule1 extends AbstractSchedule{
    @Override
    void doTask() {
        System.out.println("schedule1");
    }
}

package spring;
@Component
public class Schedule2 extends AbstractSchedule {
    @Override
    void doTask() {
        System.out.println("schedule2");
    }
}
```
我们希望上面的task作为入口，同时希望  **@Scheduled** 注解中的 **${cron}** 能够针对不同的子类起到不同的效果。

但是目前的 **@Scheduled** 注解无法满足我们的要求，按照上面的代码每个任务的时间都是相同的。如果每个子类单独设置 **@Scheduled** 注解则破坏了我们使用抽象类的初衷。

## 方案设计

为了实现上面想要达到的效果，在网上找了许多很多资料，没有发现现成的方法，无奈只得自己想办法。回想下上面提到的需求:
> 通过统一入口调度，并且时间可以区分设置。

我们知道 **@Scheduled** 的解析是在当前的task对应的bean初始化完成后调用BeanPostProcessor进行处理的，具体到源码中对应的就是**ScheduledAnnotationBeanostProcessor**。在这个processor中会读取 **@Scheduled** 的cron，如果是cron是 **${}** 这种形式的则会调用StringResolver进行解析，也就是说我们能否在这个processor处理之前对cron的值做出修改，修改成每个子类有自己的值呢？当然是可以的只需要我们在 **ScheduledAnnotationBeanostProcessor** 之前插入另一个Processor即可，并且新插入的Processor优先级要高于 **ScheduledAnnotationBeanostProcessor** 。但是这样做又会有一个问题，因为 **@Scheduled** 是加在抽象类上的，一旦修改就会影响其他类，更何况抽象类加载后不允许修改 **@Scheduled** 。这个思路似乎行不通，但是不要着急，我们换个角度想，如果抽象类不能修改我们能否修改子类呢？这时候你可能会想子类没有 **@Scheduled** 注解，并且子类即使有 **@Scheduled**，也是不允许修改的。

这时候问题变成了下面的两个子问题，
1. 子类没有@Scheduled
2. 子类即使有@Scheduled，也是不允许修改的。

针对上面的问题，我们采用如下的方法解决，
1. 将子类作为父类，运行时动态创建新的子类
2. 在新生成的子类中添加一个方法，方法名与 **@Scheduled** 所在方法一致，并且方法的作用就是调用 **@Scheduled** 所在方法。
3. 为新生成的方法添加 **@Scheduled** 注解，并且修改对应的cron为类单独的。

通过上面的方法可以达到的效果是，针对每个task生成了一个新的类，并且都带有自己的 **@Scheduled** 方法。

## 实现

### Handler

```
public class ScheduleNamespaceHandler extends NamespaceHandlerSupport {

    @Override
    public void init() {
        registerBeanDefinitionParser("cron-attach-class", new CronAttachClassParser());
    }
}
```

### Parser
```
public class CronAttachClassParser implements BeanDefinitionParser {

    public static final String CRON_ATTACH_CLASS_PROCESSOR_BEAN_NAME = "com.lpmoon.spring.schedule.cronAttachClassBeanPostProcessor";

    @Override
    public BeanDefinition parse(Element element, ParserContext parserContext) {
        Object source = parserContext.extractSource(element);
        BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(
                "com.lpmoon.spring.schedule.CronAttachClassBeanPostProcessor");
        builder.getRawBeanDefinition().setSource(source);

        registerPostProcessor(parserContext, builder, CRON_ATTACH_CLASS_PROCESSOR_BEAN_NAME);

        return null;
    }

    private static void registerPostProcessor(
            ParserContext parserContext, BeanDefinitionBuilder builder, String beanName) {

        builder.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        parserContext.getRegistry().registerBeanDefinition(beanName, builder.getBeanDefinition());
        BeanDefinitionHolder holder = new BeanDefinitionHolder(builder.getBeanDefinition(), beanName);
        parserContext.registerComponent(new BeanComponentDefinition(holder));
    }
}
```
Parser主要的功能就是注册BeanPostProcessor，用于实现我们上面的设想。

### BeanPostProcessor
```
public class CronAttachClassBeanPostProcessor implements DestructionAwareBeanPostProcessor,
        Ordered, EmbeddedValueResolverAware, BeanFactoryAware, ApplicationContextAware,
        SmartInitializingSingleton, ApplicationListener<ContextRefreshedEvent>, DisposableBean {

    private final Set<Class<?>> nonAnnotatedClasses =
            Collections.newSetFromMap(new ConcurrentHashMap<Class<?>, Boolean>(64));

    private StringValueResolver embeddedValueResolver;

    private BeanFactory beanFactory;

    private ApplicationContext applicationContext;

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }

    @Override
    public void destroy() throws Exception {

    }

    @Override
    public void afterSingletonsInstantiated() {

    }

    @Override
    public void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException {

    }

    @Override
    public boolean requiresDestruction(Object bean) {
        return false;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {

        Class<?> targetClass = AopUtils.getTargetClass(bean);

        if (!this.nonAnnotatedClasses.contains(targetClass)) {
            Map<Method, Set<Scheduled>> annotatedMethods = MethodIntrospector.selectMethods(targetClass,
                    new MethodIntrospector.MetadataLookup<Set<Scheduled>>() {
                        @Override
                        public Set<Scheduled> inspect(Method method) {
                            Set<Scheduled> scheduledMethods = AnnotatedElementUtils.getMergedRepeatableAnnotations(
                                    method, Scheduled.class, Schedules.class);
                            return (!scheduledMethods.isEmpty() ? scheduledMethods : null);
                        }
                    });
            if (annotatedMethods.isEmpty()) {
                this.nonAnnotatedClasses.add(targetClass);
            }
            else {
                //
                try {
                    return ExtentScheduledTaskFactory.generate(bean);
                } catch (NotFoundException e) {
                    e.printStackTrace();
                } catch (CannotCompileException e) {
                    e.printStackTrace();
                } catch (ClassNotFoundException e) {
                    e.printStackTrace();
                } catch (IOException e) {
                    e.printStackTrace();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
        return bean;

    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {

    }

    @Override
    public void setEmbeddedValueResolver(StringValueResolver resolver) {
        this.embeddedValueResolver = embeddedValueResolver;
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}
```

BeanPostProcessor会针对每一个带有 **@Scheduled** 方法进行一系列操作，包括生成新类，添加方法，添加注解，并且实例化新类用于替换老的对象。需要注意的是为了保证这个Processor在 **ScheduledAnnotationBeanostProcessor** 之前执行，我们需要设置其优先级高于 **ScheduledAnnotationBeanostProcessor** 。所以我们在这里设置其优先级为 **Ordered.HIGHEST_PRECEDENCE** 。

### 核心类
```
public class ExtentScheduledTaskFactory {

    public static Object generate(Object source) throws Exception {

        // generate new class extends source class
        ClassPool pool = ClassPool.getDefault();
        CtClass parent = pool.get(source.getClass().getName());
        CtClass child = pool.makeClass(source.getClass().getSimpleName() + "Child");
        child.setSuperclass(parent);
        child.defrost();

        // scan all methods with @Schedule
        ClassFile childClassFile = child .getClassFile();
        ConstPool childConstPool = childClassFile.getConstPool();

        CtMethod[] ctMethods = child.getMethods();
        for (CtMethod ctMethod : ctMethods) {
            Object scheduleAnnotation = ctMethod.getAnnotation(Scheduled.class);
            if (scheduleAnnotation != null) {
                String cron = ((Scheduled) scheduleAnnotation).cron();
                if (StringUtils.isEmpty(cron) || !cron.startsWith("$")) {
                    continue;
                }

                // generate new pattern of @Schedule.cron
                String newPattern = generatPatternWithClassName(parent, cron);

                // generate new annotation
                AnnotationsAttribute annotationsAttribute = generateScheduleAnnotation(childConstPool, newPattern);

                // generate override method for child class, and set new annotation
                CtMethod overrideMethod = generateOverrideMethod(child, ctMethod, annotationsAttribute);

                // set body
                setOverrideMethodBody(ctMethod, overrideMethod);

                // add override method
                child.addMethod(overrideMethod);
            }
        }

        Class childClass = child.toClass();
        return ObjectUtil.copy(source, childClass);
    }

    private static void setOverrideMethodBody(CtMethod ctMethod, CtMethod overrideMethod) throws NotFoundException, CannotCompileException {
        // generate parameter string for calling super method
        int paramCount = ctMethod.getParameterTypes().length;
        StringBuilder paramNameStr = new StringBuilder();
        if (paramCount >= 1) {
            paramNameStr.append("$1");
        }
        for (int i = 1; i < paramCount; i++) {
            paramNameStr.append(", $" + (i + 1));
        }

        // set method body
        boolean voidReturn = ctMethod.getReturnType() == null;
        StringBuilder body = new StringBuilder();
        body.append("{")
                .append("System.out.println(\"execute in father class; \");")
                .append((voidReturn ? "" : "return ") + "super." + ctMethod.getName() + "(" + paramNameStr.toString() + ");") //
            .append("}");

        overrideMethod.setBody(body.toString());
    }

    private static CtMethod generateOverrideMethod(CtClass child, CtMethod ctMethod, AnnotationsAttribute annotationsAttribute) throws CannotCompileException {
        CtMethod overrideMethod = CtNewMethod.copy(ctMethod, child, null);
        for (Object attribute : ctMethod.getMethodInfo().getAttributes()) {
            if (!(attribute instanceof CodeAttribute)) {
                overrideMethod.getMethodInfo().addAttribute((AttributeInfo) attribute);
            }
        }
        overrideMethod.getMethodInfo().addAttribute(annotationsAttribute);
        return overrideMethod;
    }

    private static AnnotationsAttribute generateScheduleAnnotation(ConstPool childConstPool, String newPattern) {
        AnnotationsAttribute annotationsAttribute = new AnnotationsAttribute(childConstPool, AnnotationsAttribute.visibleTag);
        Annotation annotation = new Annotation(Scheduled.class.getName(), childConstPool);
        annotation.addMemberValue("cron", new StringMemberValue("${" + newPattern + "}", childConstPool));
        annotationsAttribute.addAnnotation(annotation);
        return annotationsAttribute;
    }

    private static String generatPatternWithClassName(CtClass parent, String cron) {
        int patternStart = cron.indexOf("{");
        int patternEnd = cron.indexOf("}");
        String pattern = cron.substring(patternStart + 1, patternEnd);
        return parent.getName() + "." + pattern;
    }
}
```

上面的代码比较长，可以大致分为下面几步，
1. 使用javassit生成新的类
2. 为新的类添加与带有 **@Scheduled** 方法同名的方法
3. 为这些方法添加 **@Scheduled** 注解，注解的cron带有当前类名作为前缀

## 使用方法

在xml中配置
```
    <context:property-placeholder location="classpath:application.properties"/>
    <schedule:cron-attach-class/>
```

在配置文件中配置
```
spring.Schedule1.cron=0/5 * *  * * ? 
spring.Schedule2.cron=0/15 * *  * * ? 
```

运行在文中最开头提到的代码，可以得到如下的输出，
```
schedule2
execute in father class; 
class Schedule1Child
schedule1
execute in father class; 
class Schedule1Child
schedule1
execute in father class; 
class Schedule1Child
schedule1
execute in father class; 
class Schedule2Child
schedule2
execute in father class; 
class Schedule1Child
```

schedule2每执行一次，schedule1都已经执行了3次。和我们配置所要达到的预期一致。

## 其他
上面的方案在实现上忽略了很多的细节，还有许多待完善的地方，但是其核心目的已经达到了:

> 在不改动原有代码的基础上，在运行时注入，实现我们动态区分子任务调度时间的目的。

github地址:
https://github.com/lpmoon/spring-extension
