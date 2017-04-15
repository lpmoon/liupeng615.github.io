 java8中引入了stream，使得在某些情况下代码变得更加简洁以及优雅。举一个例子，假如我们需要在众多单词中筛选出特定字符串开始的单词的总数。在java8之前我们大概会用以下的方式来实现，
```
List<String> words = Arrays.asList("orange", "apple", "banana");
int count = 0;
for (String word : words) {
    if (word.startsWith("b")) {
        count++;
    }
}

System.out.println(count);
```
而在java8之后我们可以使用如下的方式来实现，
```
List<String> words = Arrays.asList("orange", "apple", "banana");
int count = (int) words.stream().filter((word) -> word.startsWith("b")).count();
System.out.println(count);
```
从代码结构来说，采用java8实现的代码更加简洁优雅，读者在阅读的时候很容易就可以理解这段代码的作用:
* 采用filter进行过滤
* 采用count进行计数

这里举得例子比较简单，在一些功能更加复杂的代码中，stream的优势更加明显。 上面说了stream带来的优势，很多人会好奇java8是如何实现stream的。stream实现正如其英文的含义一样，采用了流式的实现方式，数据经过一个处理器，如果满足条件则进入下一个处理器，直到最后生成结果，这与我们平常经常提到的责任链模式有点相似。
![http://ww1.sinaimg.cn/mw1024/87f5e2f6gw1fa2g9cm0a8j21hc0u0n07.jpg](http://ww1.sinaimg.cn/mw1024/87f5e2f6gw1fa2g9cm0a8j21hc0u0n07.jpg)
以上面的例子为例，我们来分析下stream的具体实现。

## words.stream()
在使用stream的时候，我们首先要把需要操作的对象转换为对应的stream。 例子中调用的stream()方法位于Collection接口中，该方法用default关键字进行修饰。default的作用是为了提供接口中方法的默认实现，在这里添加该方法是为了保证使用版本低于8的jdk编译的源码能够在java8环境中正常使用。
```
    default Stream<E> stream() {
        return StreamSupport.stream(spliterator(), false);
    }
```
而构造stream的任务交给了StreamSupport类，
```
    public static <T> Stream<T> stream(Spliterator<T> spliterator, boolean parallel) {
        Objects.requireNonNull(spliterator);
        return new ReferencePipeline.Head<>(spliterator,
                                            StreamOpFlag.fromCharacteristics(spliterator),
                                            parallel);
    }
```
构造stream需要两个参数，
* spliterator
* parallel
第二个参数很好理解，如果parallel设置为了true，那么stream中的数据在操作的时候会采用多线程的方式，这样子可以最大限度的加速stream的处理。 而第一个参数就显得有些抽象了，我们先来看一下javadoc对该参数的解释，
> An object for traversing and partitioning elements of a source. The source of elements covered by a Spliterator could be, for example, an array, a Collection, an IO channel, or a generator function.
A Spliterator may traverse elements individually (tryAdvance()) or sequentially in bulk (forEachRemaining()).
A Spliterator may also partition off some of its elements (using trySplit) as another Spliterator, to be used in possibly-parallel operations. Operations using a Spliterator that cannot split, or does so in a highly imbalanced or inefficient manner, are unlikely to benefit from parallelism. Traversal and splitting exhaust elements; each Spliterator is useful for only a single bulk computation.

spliterator可以作为遍历所有元素的迭代器，也可以作为划分元素的分类器。 在我们的例子中spliterator的构造方式如下，
```
    @Override
    public Spliterator<E> spliterator() {
        return new ArrayListSpliterator<>(this, 0, -1, 0);
    }
```
从spliterator的命名来看，可以看出当我们的原始对象是Array的时候，spliterator属于上文提到的元素迭代器。 回到刚才stream的初始化的地方，StreamSupport方法返回的是ReferencePipeline.Head的实例，而Head是抽象类ReferencePipeline的具体实现，而ReferencePipeline又是AbstractPipeline的实现。 这里需要先解释下AbstractPipeline中的几个属性，这样方便后续代码的理解，
```
/**
     * Backlink to the head of the pipeline chain (self if this is the source
     * stage).
     * 最初始的pipeline，也就是从普通对象通过stream()方法而得到的第一个stream()。该pipeline会随着后续
     * pipeline的生成而一直传递下去。
     */
    @SuppressWarnings("rawtypes")
    private final AbstractPipeline sourceStage;

    /**
     * The "upstream" pipeline, or null if this is the source stage.
     * 当前pipeline的上游pipeline。
     */
    @SuppressWarnings("rawtypes")
    private final AbstractPipeline previousStage;

    /**
     * The next stage in the pipeline, or null if this is the last stage.
     * Effectively final at the point of linking to the next pipeline.
     * 当前pipeline的下游pipeline。 理论上消息经过当前pipeline处理后会传递给该pipeline。 如果当前pipeline是最后
     * 阶段的pipeline，那么该属性应该是null。
     */
    @SuppressWarnings("rawtypes")
    private AbstractPipeline nextStage;

    /**
     * The source spliterator. Only valid for the head pipeline.
     * Before the pipeline is consumed if non-null then {@code sourceSupplier}
     * must be null. After the pipeline is consumed if non-null then is set to
     * null.
     * 构造初始pipeline的时候传入的spliterator。
     */
    private Spliterator<?> sourceSpliterator;

    /**
     * The source supplier. Only valid for the head pipeline. Before the
     * pipeline is consumed if non-null then {@code sourceSpliterator} must be
     * null. After the pipeline is consumed if non-null then is set to null.
     */
    private Supplier<? extends Spliterator<?>> sourceSupplier;

    /**
     * True if pipeline is parallel, otherwise the pipeline is sequential; only
     * valid for the source stage.
     */
    private boolean parallel;
```

至此，我们最原始的stream已经生成了。 让我们来看一下内存中的数据，
![http://ww3.sinaimg.cn/mw690/87f5e2f6jw1fa2iu8x2eij20gp08rmyo.jpg](http://ww3.sinaimg.cn/mw690/87f5e2f6jw1fa2iu8x2eij20gp08rmyo.jpg)

当前pipeline的pre和next pipeline都是null。

## filter((word) -> word.startsWith("b"))
filter用于返回一个带有过滤功能的stream，其具体实现如下所示，
```
    @Override
    public final Stream<P_OUT> filter(Predicate<? super P_OUT> predicate) {
        Objects.requireNonNull(predicate);
        return new StatelessOp<P_OUT, P_OUT>(this, StreamShape.REFERENCE,
                                     StreamOpFlag.NOT_SIZED) {
            @Override
            Sink<P_OUT> opWrapSink(int flags, Sink<P_OUT> sink) {
                return new Sink.ChainedReference<P_OUT, P_OUT>(sink) {
                    @Override
                    public void begin(long size) {
                        downstream.begin(-1);
                    }

                    @Override
                    public void accept(P_OUT u) {
                        if (predicate.test(u))
                            downstream.accept(u);
                    }
                };
            }
        };
    }
```
在构造stream的时候，传入了一个参数StreamShape.REFERENCE, 该参数用于表示上游pipeline的类型。目前默认的有四种类型，
```
enum StreamShape {
    /**
     * The shape specialization corresponding to {@code Stream} and elements
     * that are object references.
     * stream中的类型是object引用
     */
    REFERENCE,
    /**
     * The shape specialization corresponding to {@code IntStream} and elements
     * that are {@code int} values.
     * stream中的元素是int类型
     */
    INT_VALUE,
    /**
     * The shape specialization corresponding to {@code LongStream} and elements
     * that are {@code long} values.
     * stream中的元素是long类型
     */
    LONG_VALUE,
    /**
     * The shape specialization corresponding to {@code DoubleStream} and
     * elements that are {@code double} values.
     * stream中的类型是double类型
     */
    DOUBLE_VALUE
}
```

filter方法最终调用的构造函数是，
```
    /**
     * Constructor for appending an intermediate operation stage onto an
     * existing pipeline.
     *
     * @param previousStage the upstream pipeline stage
     * @param opFlags the operation flags for the new stage, described in
     * {@link StreamOpFlag}
     */
    AbstractPipeline(AbstractPipeline<?, E_IN, ?> previousStage, int opFlags) {
        if (previousStage.linkedOrConsumed)
            throw new IllegalStateException(MSG_STREAM_LINKED);
        previousStage.linkedOrConsumed = true;
        previousStage.nextStage = this; // @0

        this.previousStage = previousStage; // @1
        this.sourceOrOpFlags = opFlags & StreamOpFlag.OP_MASK;
        this.combinedFlags = StreamOpFlag.combineOpFlags(opFlags, previousStage.combinedFlags);
        this.sourceStage = previousStage.sourceStage; // @2
        if (opIsStateful())
            sourceStage.sourceAnyStateful = true;
        this.depth = previousStage.depth + 1; // @3
    }
```
这与初始化最初的stream的时候是不同的。 这里需要指定previous pipeline。 代码中的@0处，将当前pipeline的next pipeline设置为新生成的pipeline，代码中的@1处，将新生成的pipeline的previous pipeline设置为当前的pipeline。 与filter类似的其他方法调用也会产生同样的效果，多次通过类似方法调用构造函数后，会形成一个类似双向链表的责任链。 代码中的@2处会将原始的pipeline不停的传递下去。 新生成的stream都实现了onWrapSink方法，该方法最终会将整条调用链路串联起来，这个后面会提到。 

到这里调用filter方法生成的stream就构造完成了。我们来看一下内存中的数据是什么样子，
![http://ww2.sinaimg.cn/mw690/87f5e2f6gw1fa2ixzc6z1j20gt0dbwgw.jpg](http://ww2.sinaimg.cn/mw690/87f5e2f6gw1fa2ixzc6z1j20gt0dbwgw.jpg)

当前的stream已经更新，当前pipeline的previous pipeline正是刚才生成的初始的pipeline。而初始的pipeline的next pipeline则是当前pipeline。

## count()
细心的同学会发现到这一步之前，代码一直在生成新的stream，并没有开始实际的结果生成，大家可以尝试打一下断点印证一下。 而count方法才是最终返回结果的函数，类似count函数的还有min, max等几个函数。 这些函出现在调用链的最后端，用于结束调用链，同时生成最终我们需要的结果。
这里看一下count的实现，
```
    public final long count() {
        return mapToLong(e -> 1L).sum();
    }
```
从代码中可以看出count是由两步构成的:
* 将过滤后的元素映射为1
* 映射后的元素求和

继续看一下mapToLang究竟做了什么，
```
    @Override
    public final LongStream mapToLong(ToLongFunction<? super P_OUT> mapper) {
        Objects.requireNonNull(mapper);
        return new LongPipeline.StatelessOp<P_OUT>(this, StreamShape.REFERENCE,
                                      StreamOpFlag.NOT_SORTED | StreamOpFlag.NOT_DISTINCT) {
            @Override
            Sink<P_OUT> opWrapSink(int flags, Sink<Long> sink) {
                return new Sink.ChainedReference<P_OUT, Long>(sink) {
                    @Override
                    public void accept(P_OUT u) {
                        downstream.accept(mapper.applyAsLong(u));
                    }
                };
            }
        };
    }
```
mapToLong的实现和filter十分类似，一个不同点的是mapToLong返回的是LongStream，通过前面的StreamShape可以知道LongStream中的所有元素都是long。 另一个不同点是mapToLong传入的参数是mapper用于映射，而filter传入的是predicate用于过滤。 我们再来看一下经过这一步后的内存数据，
![http://ww3.sinaimg.cn/mw690/87f5e2f6jw1fa2jk53tsij20gs0lcwif.jpg](http://ww3.sinaimg.cn/mw690/87f5e2f6jw1fa2jk53tsij20gs0lcwif.jpg)

从最外层开始依次是mappter, filter, initial pipeline。

接下来看一下sum方法都做了什么，
```
    @Override
    public final long sum() {
        // use better algorithm to compensate for intermediate overflow?
        return reduce(0, Long::sum);
    }
```
sum调用reduce函数进行了聚合操作，reduce第一个参数传入的是初始值，而Long::sum其实是下面的代码的简写，用于对LongStream中的元素进行累加操作。
```
new LongBinaryOperator() {
     @Overried
     long applyAsLong(long left, long right) {
        return Long.sum(left, right);
     }
}
```
reduce方法如下所示，
```
    @Override
    public final long reduce(long identity, LongBinaryOperator op) {
        return evaluate(ReduceOps.makeLong(identity, op));
    }
```
而最终的执行在evaluate中，
```
    final <R> R evaluate(TerminalOp<E_OUT, R> terminalOp) {
        assert getOutputShape() == terminalOp.inputShape();
        if (linkedOrConsumed)
            throw new IllegalStateException(MSG_STREAM_LINKED);
        linkedOrConsumed = true;

        return isParallel()
               ? terminalOp.evaluateParallel(this, sourceSpliterator(terminalOp.getOpFlags()))
               : terminalOp.evaluateSequential(this, sourceSpliterator(terminalOp.getOpFlags()));
    }
```
这里会调用ReduceOp的evaluateSequential方法进行执行。 之所以会调用evaluateSequential是因为我们在最开始生成的stream传入的参数parallel=false。
```
        @Override
        public <P_IN> R evaluateSequential(PipelineHelper<T> helper,
                                           Spliterator<P_IN> spliterator) {
            return helper.wrapAndCopyInto(makeSink(), spliterator).get();
        }
```
makeSink()用于生成最终的sink，也就是将long累加的sink。 这个方法的定义在ReduceOps.makeLong中，
```
    public static TerminalOp<Long, Long>
    makeLong(long identity, LongBinaryOperator operator) {
        Objects.requireNonNull(operator);
        class ReducingSink
                implements AccumulatingSink<Long, Long, ReducingSink>, Sink.OfLong {
            private long state;

            @Override
            public void begin(long size) {
                state = identity;
            }

            @Override
            public void accept(long t) {
                state = operator.applyAsLong(state, t); // 这里接收初始值和累加器生成最终的结果。
            }

            @Override
            public Long get() {
                return state;
            }

            @Override
            public void combine(ReducingSink other) {
                accept(other.state);
            }
        }
        return new ReduceOp<Long, Long, ReducingSink>(StreamShape.LONG_VALUE) {
            @Override
            public ReducingSink makeSink() {
                return new ReducingSink();
            }
        };
    }
```
wrapAndCopyInfo实现如下，
```
    @Override
    final <P_IN, S extends Sink<E_OUT>> S wrapAndCopyInto(S sink, Spliterator<P_IN> spliterator) {
        copyInto(wrapSink(Objects.requireNonNull(sink)), spliterator);
        return sink;
    }
```
wrapsink用于将当前所有的pipeline串联起来，
```
    @Override
    @SuppressWarnings("unchecked")
    final <P_IN> Sink<P_IN> wrapSink(Sink<E_OUT> sink) {
        Objects.requireNonNull(sink);

        for ( @SuppressWarnings("rawtypes") AbstractPipeline p=AbstractPipeline.this; p.depth > 0; p=p.previousStage) {
            sink = p.opWrapSink(p.previousStage.combinedFlags, sink);
        }
        return (Sink<P_IN>) sink;
    }
```
从当前pipeline往前遍历，调用opWrapSink方法，不断生成新的sink。 该方法的初始化p是ReferencePipeline，同时操作类型是mapToLong。

```
    @Override
    public final LongStream mapToLong(ToLongFunction<? super P_OUT> mapper) {
        Objects.requireNonNull(mapper);
        return new LongPipeline.StatelessOp<P_OUT>(this, StreamShape.REFERENCE,
                                      StreamOpFlag.NOT_SORTED | StreamOpFlag.NOT_DISTINCT) {
            @Override
            Sink<P_OUT> opWrapSink(int flags, Sink<Long> sink) {
                return new Sink.ChainedReference<P_OUT, Long>(sink) {
                    @Override
                    public void accept(P_OUT u) {
                        downstream.accept(mapper.applyAsLong(u));
                    }
                };
            }
        };
    }
```
new LongPipeline.StatelessOp(...)会生成新的sink，该sink的作用是接收一个参数调用mapper将其映射为long，同时调用下游的sink进行处理。 该sink在初始化的时候会指定下游的sink为当前传入的sink。
```
    static abstract class ChainedReference<T, E_OUT> implements Sink<T> {
        protected final Sink<? super E_OUT> downstream;

        public ChainedReference(Sink<? super E_OUT> downstream) {
            this.downstream = Objects.requireNonNull(downstream); // @1
        }

        @Override
        public void begin(long size) {
            downstream.begin(size);
        }

        @Override
        public void end() {
            downstream.end();
        }

        @Override
        public boolean cancellationRequested() {
            return downstream.cancellationRequested();
        }
    }
```
在代码@1指定了downstream。 这样就形成了如下的调用链，
<font color="red">**mapToLong sink -> ReducingSink(累加)**</font>
最终会形成如下的调用链，
<font color="red">**filter sink -> mapToLong sink -> ReducingSink(累加)**</font>

回到刚才的copyAndWrapInto方法，copyInto方法如下所示，
```
    @Override
    final <P_IN> void copyInto(Sink<P_IN> wrappedSink, Spliterator<P_IN> spliterator) {
        Objects.requireNonNull(wrappedSink);

        if (!StreamOpFlag.SHORT_CIRCUIT.isKnown(getStreamAndOpFlags())) {
            wrappedSink.begin(spliterator.getExactSizeIfKnown());
            spliterator.forEachRemaining(wrappedSink);
            wrappedSink.end();
        }
        else {
            copyIntoWithCancel(wrappedSink, spliterator);
        }
    }
```
这里最主要的功能集中在forEachRemaining方法，该方法负责遍历最初始的对象中的所有数据然后将数据依次传入到上面生成的sink链中。
```
        @SuppressWarnings("unchecked")
        @Override
        public void forEachRemaining(Consumer<? super T> action) {
            Object[] a; int i, hi; // hoist accesses and checks from loop
            if (action == null)
                throw new NullPointerException();
            if ((a = array).length >= (hi = fence) &&
                (i = index) >= 0 && i < (index = hi)) {
                do { action.accept((T)a[i]); } while (++i < hi);
            }
        }
```

<font color="red">**传入"orange"后ReducingSink的值是0，"ornage"在filter sink阶段被过滤。
传入"apple"后ReducingSink的值是0，"apple"在filter sink阶段被过滤。
传入"banana"后ReducingSink的值是1, "banana"在filter sink阶段被接收，进入到mapToLong sink被映射为1，然后在ReducingSink阶段和0相加得到1.**</font>

最终调用ReducingSink的get方法就可以获得我们需要的值。

到此我们的例子的执行过程就分析完了。 无论我们的stream按照何种方式进行组合都会在最终生成sink链，然后依次将数据传入进行处理得到最终结果。 流程如下图所示，
![http://ww3.sinaimg.cn/mw690/87f5e2f6jw1fa2ml1gskcj21hc0u0wip.jpg](http://ww3.sinaimg.cn/mw690/87f5e2f6jw1fa2ml1gskcj21hc0u0wip.jpg)


这里分析的例子比较简单，还有许多细节没有提到，比如如果我们采用的方式是parallel的，执行的过程是怎样的呢？ 这里就不展开了。。。大家有兴趣可以自己去看一看。 总的来说stream的实现是非常巧妙的，大家在平常编程的时候也可以借鉴其思想。。。
