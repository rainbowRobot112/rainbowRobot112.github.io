# Netty channel 独有资源管理

DefaultAttributeMap 机制是 netty 提供对 channel 独有资源管理的手段。逻辑 channel 都是DefaultAttributeMap 的子类（逻辑 channel 是 netty 提供的接口实现类比如 NioSocketChannel，作为 attachment 绑定在 jdk 底层 channel 对象上），能在并发条件下管理和绑定 Attribute（以 AttributeKey 作为唯一标识 ），而 Attribute 则缓存 channel 生命周期中数据。

netty 对连接的独有资源管理手段是值得学习的一点，本文目标是解析整个 DefaultAttributeMap 机制的原理。

选用 netty 版本为 **4.1.86.Final**。

# 使用案例

Attribute 一般使用在继承 ChannelInboundHandlerAdapter 类的 hanler 中，比如记录连接的启动时间和结束时间

```java
@ChannelHandler.Sharable
public class ConnectHandler extends ChannelInboundHandlerAdapter {

    private static final String CHANNEL_START_TIMESTAMP = "s_t";
    private static final String CHANNEL_INFORMATION = "channel_information";

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        Attribute<Map<String, Object>> attribute = ctx.channel().attr(AttributeKey.valueOf(CHANNEL_INFORMATION));
        Map<String, Object> map = attribute.get();
        if (map == null) {
            map = new HashMap<>();
            attribute.set(map);
        }
        map.put(CHANNEL_START_TIMESTAMP, System.currentTimeMillis());
        super.channelActive(ctx);
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        super.channelInactive(ctx);
        Attribute<Map<String, Object>> attribute = ctx.channel().attr(AttributeKey.valueOf(CHANNEL_INFORMATION));
        long startTime = (long) attribute.getAndSet(null).getOrDefault(CHANNEL_START_TIMESTAMP, 0L);
        System.out.println("Channel end. begin time: " + startTime + ", end time: " + System.currentTimeMillis());
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        Attribute<Map<String, Object>> attribute = ctx.channel().attr(AttributeKey.valueOf(CHANNEL_INFORMATION));
        long startTime = (long) attribute.getAndSet(null).getOrDefault(CHANNEL_START_TIMESTAMP, 0L);
        System.out.println("Channel failed. begin time: " + startTime + ", end time: " + System.currentTimeMillis());
    }
}
```

每个 channel 与 各自的 AttributeKey 绑定，适用于被 @ChannelHandler.Sharable 注解的单例 hanlder 中保证单个 channel 持有独立资源。

# 源码分析

首先是 DefaultAttributeMap，下面为 NioSocketChannel 开始的继承关系图，红框中表示 NioSocketChannel 继承自 DefaultAttributeMap。所有 netty channel 都继承自 AbstractChannel，从继承关系来 channel 都继承自 DefaultAttributeMap 具备绑定 Attribute 的能力。

![img](https://cdn.nlark.com/yuque/0/2023/png/1945901/1701092971365-f5d1a529-b19b-4bfb-9fca-91f93d8c55f3.png)

Attribute 继承关系图，继承 AtomicReference 具备并发修改属性值的能力，负责管理 AttributeKey

![img](https://cdn.nlark.com/yuque/0/2023/png/1945901/1701095784082-f0ecd07b-faf8-4269-9f97-8b87c78104d6.png)

AttributeKey 继承关系图，存储唯一 key 关键字以及给每个 key 设置的唯一 id

![img](https://cdn.nlark.com/yuque/0/2023/png/1945901/1701179965471-e2f3f46c-c117-43d2-a082-df83cb877acf.png)

DefaultAttributeMap 与 AttributeKey 类图关系，可以清晰看到 AttributeMap 和 Attribute 的功能

![yuque_diagram](yuque_diagram.png)

## AttributeKey 源码

```java
public final class AttributeKey<T> extends AbstractConstant<AttributeKey<T>> {

    // 全局 AttributeKey 对象管理池，保证并发环境下 AttributeKey 的唯一性
    private static final ConstantPool<AttributeKey<Object>> pool = new ConstantPool<AttributeKey<Object>>() {
        @Override
        protected AttributeKey<Object> newConstant(int id, String name) {
            return new AttributeKey<Object>(id, name);
        }
    };

    // 从对象池中获取 key 为 name 的 AttributeKey
    @SuppressWarnings("unchecked")
    public static <T> AttributeKey<T> valueOf(String name) {
        return (AttributeKey<T>) pool.valueOf(name);
    }

    // 判断对象池中是否存在 key 为 name 的 AttributeKey
    public static boolean exists(String name) {
        return pool.exists(name);
    }

    // 在对象池中实例化 key 为 name 的 AttributeKey
    @SuppressWarnings("unchecked")
    public static <T> AttributeKey<T> newInstance(String name) {
        return (AttributeKey<T>) pool.newInstance(name);
    }

    //valueOf(firstNameComponent.getName() + "#" + secondNameComponent) 获取 AttributeKey
    @SuppressWarnings("unchecked")
    public static <T> AttributeKey<T> valueOf(Class<?> firstNameComponent, String secondNameComponent) {
        return (AttributeKey<T>) pool.valueOf(firstNameComponent, secondNameComponent);
    }

    // 禁止外部调用构造方法
    private AttributeKey(int id, String name) {
        super(id, name);
    }
}
```

AttributeKey 类中有个很重要的类 ConstantPool，管理 AttributeKey 的对象池，负责 AttributeKey 的创建和管理。ConstantPool 是抽象类，使用时需实现 newConstant 方法处理对象的创建

```java
public abstract class ConstantPool<T extends Constant<T>> {

    // netty 优化实现的 ConcurrentMap
    private final ConcurrentMap<String, T> constants = PlatformDependent.newConcurrentHashMap();

    // id 标识生成
    private final AtomicInteger nextId = new AtomicInteger(1);

    public T valueOf(Class<?> firstNameComponent, String secondNameComponent) {
        return valueOf(
                checkNotNull(firstNameComponent, "firstNameComponent").getName() +
                '#' +
                checkNotNull(secondNameComponent, "secondNameComponent"));
    }

    public T valueOf(String name) {
        return getOrCreate(checkNonEmpty(name, "name"));
    }

    // 池中存在对象则返回，否则创建对象。线程安全基于 ConcurrentMap
    private T getOrCreate(String name) {
        T constant = constants.get(name);
        if (constant == null) {
            final T tempConstant = newConstant(nextId(), name);
            constant = constants.putIfAbsent(name, tempConstant);
            if (constant == null) {
                return tempConstant;
            }
        }

        return constant;
    }

    public boolean exists(String name) {
        return constants.containsKey(checkNonEmpty(name, "name"));
    }

    public T newInstance(String name) {
        return createOrThrow(checkNonEmpty(name, "name"));
    }

    // 创建对象，如果创建后对象不为 null 则抛出异常。线程安全基于 ConcurrentMap
    private T createOrThrow(String name) {
        T constant = constants.get(name);
        if (constant == null) {
            final T tempConstant = newConstant(nextId(), name);
            constant = constants.putIfAbsent(name, tempConstant);
            if (constant == null) {
                return tempConstant;
            }
        }

        throw new IllegalArgumentException(String.format("'%s' is already in use", name));
    }

    // 需要继承实现的对象创建方法
    protected abstract T newConstant(int id, String name);

    @Deprecated
    public final int nextId() {
        return nextId.getAndIncrement();
    }
}
```

getOrCreate 和 createOrThrow 这里说是线程安全，其实应该考虑场景，pool 管理的是 AttributeKey，AttributeKey 生成时间一般在连接创建的时候比如上面的案例，保证并发创建 AttributeKey 时资源争抢不出现问题，即使出现创建新的对象覆盖已有对象的情况，作为 key 的 name 也不会变。如果对创建记录 key 的对象要求较高可以使用 createOrThrow，创建对象时如果对象已存在会抛出异常，然后再使用 getOrCreate 获取对象。

## Attribute 源码

DefaultAttribute 是 DefaultAttributeMap 的内部类，也是 Attribute 实现类，修饰符是 private，限制在 DefaultAttributeMap 类中使用。

```java
    private static final class DefaultAttribute<T> extends AtomicReference<T> implements Attribute<T> {

        //提供 attributeMap 属性的原子性操作
        private static final AtomicReferenceFieldUpdater<DefaultAttribute, DefaultAttributeMap> MAP_UPDATER =
                AtomicReferenceFieldUpdater.newUpdater(DefaultAttribute.class,
                                                       DefaultAttributeMap.class, "attributeMap");
        private static final long serialVersionUID = -2661411462200283011L;

        //当前 attribute 所在 attributeMap
        private volatile DefaultAttributeMap attributeMap;
        //DefaultAttribute 标识的 key
        private final AttributeKey<T> key;

        DefaultAttribute(DefaultAttributeMap attributeMap, AttributeKey<T> key) {
            this.attributeMap = attributeMap;
            this.key = key;
        }

        @Override
        public AttributeKey<T> key() {
            return key;
        }

        private boolean isRemoved() {
            return attributeMap == null;
        }

        @Override
        public T setIfAbsent(T value) {
            while (!compareAndSet(null, value)) {
                T old = get();
                if (old != null) {
                    return old;
                }
            }
            return null;
        }

        // 线程安全的方式删除 attributeMap 并返回当前值
        @Override
        public T getAndRemove() {
            final DefaultAttributeMap attributeMap = this.attributeMap;
            final boolean removed = attributeMap != null && MAP_UPDATER.compareAndSet(this, attributeMap, null);
            T oldValue = getAndSet(null);
            if (removed) {
                attributeMap.removeAttributeIfMatch(key, this);
            }
            return oldValue;
        }

        // 线程安全的方式删除 attributeMap
        @Override
        public void remove() {
            final DefaultAttributeMap attributeMap = this.attributeMap;
            final boolean removed = attributeMap != null && MAP_UPDATER.compareAndSet(this, attributeMap, null);
            set(null);
            if (removed) {
                attributeMap.removeAttributeIfMatch(key, this);
            }
        }
    }
```

## DefaultAttributeMap 源码

DefaultAttributeMap 负责管理 attribute，

```java
public class DefaultAttributeMap implements AttributeMap {

    // 提供 attributes 的原子性操作
    private static final AtomicReferenceFieldUpdater<DefaultAttributeMap, DefaultAttribute[]> ATTRIBUTES_UPDATER =
            AtomicReferenceFieldUpdater.newUpdater(DefaultAttributeMap.class, DefaultAttribute[].class, "attributes");
    // 默认空 DefaultAttribute[]
    private static final DefaultAttribute[] EMPTY_ATTRIBUTES = new DefaultAttribute[0];

    // 按照 AttributeKey 的 id 进行二分查找，找到 AttributeKey 对应的 attribute 的 index
    // 如果不存在则返回负数
    private static int searchAttributeByKey(DefaultAttribute[] sortedAttributes, AttributeKey<?> key) {
        int low = 0;
        int high = sortedAttributes.length - 1;

        while (low <= high) {
            int mid = low + high >>> 1;
            DefaultAttribute midVal = sortedAttributes[mid];
            AttributeKey midValKey = midVal.key;
            if (midValKey == key) {
                return mid;
            }
            int midValKeyId = midValKey.id();
            int keyId = key.id();
            assert midValKeyId != keyId;
            boolean searchRight = midValKeyId < keyId;
            if (searchRight) {
                low = mid + 1;
            } else {
                high = mid - 1;
            }
        }

        return -(low + 1);
    }

    // 新 Attribute 插入到数组中并保证id的有序
    private static void orderedCopyOnInsert(DefaultAttribute[] sortedSrc, int srcLength, DefaultAttribute[] copy,
                                            DefaultAttribute toInsert) {
        // let's walk backward, because as a rule of thumb, toInsert.key.id() tends to be higher for new keys
        final int id = toInsert.key.id();
        int i;
        for (i = srcLength - 1; i >= 0; i--) {
            DefaultAttribute attribute = sortedSrc[i];
            assert attribute.key.id() != id;
            // 设置的id号大于attributes中最大的id则直接跳出循环
            if (attribute.key.id() < id) {
                break;
            }
            // 否则将最大id的attribute放入队尾
            copy[i + 1] = sortedSrc[i];
        }
        copy[i + 1] = toInsert;
        final int toCopy = i + 1;
        if (toCopy > 0) {
            //将剩下attribute放入队列中
            System.arraycopy(sortedSrc, 0, copy, 0, toCopy);
        }
    }

    // 初始 attributes 为空数组
    private volatile DefaultAttribute[] attributes = EMPTY_ATTRIBUTES;

    // 通过 AttributeKey 获取返回值 Attribute, 如果不存在则创建新的 Attribute 并返回
    @SuppressWarnings("unchecked")
    @Override
    public <T> Attribute<T> attr(AttributeKey<T> key) {
        ObjectUtil.checkNotNull(key, "key");
        DefaultAttribute newAttribute = null;
        // 通过循环与CAS的方式避免使用互斥锁
        for (;;) {
            final DefaultAttribute[] attributes = this.attributes;
            // 通过 AttributeKey 查询index是否存在，如果不存在返回负值
            final int index = searchAttributeByKey(attributes, key);
            final DefaultAttribute[] newAttributes;
            if (index >= 0) {
                final DefaultAttribute attribute = attributes[index];
                assert attribute.key() == key;
                if (!attribute.isRemoved()) {
                    return attribute;
                }
                // 替换被删除的 AttributeKey
                if (newAttribute == null) {
                    newAttribute = new DefaultAttribute<T>(this, key);
                }
                final int count = attributes.length;
                newAttributes = Arrays.copyOf(attributes, count);
                newAttributes[index] = newAttribute;
            } else {
                if (newAttribute == null) {
                    newAttribute = new DefaultAttribute<T>(this, key);
                }
                final int count = attributes.length;
                newAttributes = new DefaultAttribute[count + 1];
                orderedCopyOnInsert(attributes, count, newAttributes, newAttribute);
            }
            if (ATTRIBUTES_UPDATER.compareAndSet(this, attributes, newAttributes)) {
                return newAttribute;
            }
        }
    }

    @Override
    public <T> boolean hasAttr(AttributeKey<T> key) {
        ObjectUtil.checkNotNull(key, "key");
        return searchAttributeByKey(attributes, key) >= 0;
    }

    // 删除 AttributeKey 对应的 attribute
    private <T> void removeAttributeIfMatch(AttributeKey<T> key, DefaultAttribute<T> value) {
        // 循环和CAS操作来避免使用互斥锁
        for (;;) {
            final DefaultAttribute[] attributes = this.attributes;
            final int index = searchAttributeByKey(attributes, key);
            // 负值表示不存在 attribute，直接返回
            if (index < 0) {
                return;
            }
            final DefaultAttribute attribute = attributes[index];
            assert attribute.key() == key;
            if (attribute != value) {
                return;
            }
            // 只一个 attribute 元素时将 attributes 设置为 EMPTY_ATTRIBUTES
            final int count = attributes.length;
            final int newCount = count - 1;
            final DefaultAttribute[] newAttributes =
                    newCount == 0? EMPTY_ATTRIBUTES : new DefaultAttribute[newCount];
            // 重新生成新的 attributes
            // perform 2 bulk copies
            System.arraycopy(attributes, 0, newAttributes, 0, index);
            final int remaining = count - index - 1;
            if (remaining > 0) {
                System.arraycopy(attributes, index + 1, newAttributes, index, remaining);
            }
            if (ATTRIBUTES_UPDATER.compareAndSet(this, attributes, newAttributes)) {
                return;
            }
        }
    }
    ...(DefaultAttribute)
}
```

# 总结

DefaultAttributeMap 的设计提升了读资源的速度，增加了写资源的成本，适合并发下读操作多的场景。并发性能上，DefaultAttributeMap 机制在并发上做了很多优化，使用了内存友好型的方式编程，减少了缓存随机读的问题。使用CAS乐观锁替换了性能影响较大的互斥锁，不再堵塞线程，提高了并发的性能。