

# 主要技术栈

![image-20220412153038573](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412153038573.png)

# 整体模块-微服务项目

![image-20220412153051473](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412153051473.png)

ORM： 对象关系映射（Object Relational Mapping，简称ORM）模式是一种为了解决面向对象与关系数据库存在的互不匹配的现象的技术。简单的说，ORM是通过使用描述对象和数据库之间映射的元数据，将程序中的对象自动持久化到关系数据库中。

## MVC开发流程

![image-20220412153102451](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412153102451.png)

## 秒杀系统整体业务流程

![image-20220412153110261](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412153110261.png)

## 数据库设计

**核心就是前三张表，随机数的表主要用于测试高并发下生成订单编号怎么保持唯一**

![image-20220412153126103](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412153126103.png)

# 各模块设计

## 商品列表展示

![image-20220412153159275](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412153159275.png)

Cankill字段是为了防止用户在自己电脑上更改时间，则可以提前进行秒杀(使用CASE WHEN)

### Controller:

```java
@Controller
public class ItemController {
private static final Logger log = LoggerFactory.getLogger(ItemController.class);
private static final String prefix = "item";
@Autowired
private IItemService itemService;
/**
 * 获取商品列表
 */
@RequestMapping(value = {"/","/index",prefix+"/list",prefix+"/index.html"},method = RequestMethod.GET)
public String list(ModelMap modelMap){
    try {
        //获取待秒杀商品列表
        List<ItemKill> list=itemService.getKillItems();
        modelMap.put("list",list);

        log.info("获取待秒杀商品列表-数据：{}",list);
    }catch (Exception e){
        log.error("获取待秒杀商品列表-发生异常：",e.fillInStackTrace());
        return "redirect:/base/error";
    }
    return "list";
 }
}
```

### Service:

```java
@Service
public class ItemService implements IItemService {
    private static final Logger log = LoggerFactory.getLogger(ItemService.class);
    @Autowired
    private ItemKillMapper itemKillMapper;

    /**
     * 获取待秒杀商品列表
     * @return
     * @throws Exception
     */
    @Override
    public List<ItemKill> getKillItems() throws Exception {
        return itemKillMapper.selectAll();
    }
}
```

### Mapper(DAO)：

```java
public interface ItemKillMapper {
    List<ItemKill> selectAll();

    ItemKill selectById(@Param("id") Integer id);

    int updateKillItem(@Param("killId") Integer killId);



    ItemKill selectByIdV2(@Param("id") Integer id);

    int updateKillItemV2(@Param("killId") Integer killId);
}
```

对应的xml文件：

```xml
<!--查询待秒杀的活动商品列表-->
<select id="selectAll" resultType="com.kobe.kill.model.entity.ItemKill">
  SELECT
    a.*,
    b.name AS itemName,
    (
      CASE WHEN (now() BETWEEN a.start_time AND a.end_time AND a.total > 0)
        THEN 1
      ELSE 0
      END
    )      AS canKill
  FROM item_kill AS a LEFT JOIN item AS b ON b.id = a.item_id
  WHERE a.is_active = 1
</select>
```

### 出现错误与解决方式：

<img src="C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412154133651.png" alt="image-20220412154133651" style="zoom:150%;" />

出现ErrorPageFilter的错误

 原因：

 （1）检查程序中response响应的对象写的位置是否正确，会引起这个异常。

 （2）用户在请求某个资源或链接时，由于用户网速真心感人，可能用户会提前断掉请求（关闭页面，刷新也是有可能的）

解决：先把error页面删除

## 秒杀商品详情展示：

![image-20220412154439212](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412154439212.png)

### Controller

```java
@Controller
public class ItemController {
    private static final Logger log = LoggerFactory.getLogger(ItemController.class);

    private static final String prefix = "item";

    @Autowired
    private IItemService itemService;
    /* *
     * 获取待秒杀商品的详情
     * @return
     * */

    @RequestMapping(value = prefix+"/detail/{id}",method = RequestMethod.GET)
    public String detail(@PathVariable Integer id, ModelMap modelMap){
        if(id==null||id<0){
            return "redirect:/base/error";
        }
        try {
            ItemKill detail=itemService.getKillDetail(id);
            modelMap.put("detail",detail);
        }catch (Exception e){
            log.error("获取待秒杀商品的详情-发生异常：id={}",id,e.fillInStackTrace());
            return "redirect:/base/error";
        }
        return "info";
    }
}
```

### Service

```java
@Service
public class ItemService implements IItemService {
    private static final Logger log = LoggerFactory.getLogger(ItemService.class);
    @Autowired
    private ItemKillMapper itemKillMapper;
    @Override 
    public ItemKill getKillDetail(Integer id) throws Exception {
        ItemKill entity=itemKillMapper.selectById(id);
        if (entity==null){
            throw new Exception("获取秒杀详情-待秒杀商品记录不存在");
        }
        return entity;
    }
}
```

### Mapper(Dao)

```java
public interface ItemKillMapper {
    List<ItemKill> selectAll();

    ItemKill selectById(@Param("id") Integer id);

    int updateKillItem(@Param("killId") Integer killId);



    ItemKill selectByIdV2(@Param("id") Integer id);

    int updateKillItemV2(@Param("killId") Integer killId);
}
```

```xml
<!--获取秒杀详情-->
<select id="selectById" resultType="com.kobe.kill.model.entity.ItemKill">
  SELECT
    a.*,
    b.name AS itemName,
    (
      CASE WHEN (now() BETWEEN a.start_time AND a.end_time AND a.total > 0)
        THEN 1
      ELSE 0
      END
    )      AS canKill
  FROM item_kill AS a LEFT JOIN item AS b ON b.id = a.item_id
  WHERE a.is_active = 1 AND a.id= #{id}
</select>
```

## 商品秒杀实战

### 流程

![image-20220412155146754](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412155146754.png)

#### Controller

```java
/***
     * 商品秒杀核心业务逻辑
     * @param dto
     * @param result
     * @return
     */
    @RequestMapping(value = prefix + "/execute", method = RequestMethod.POST, consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
    @ResponseBody
    public BaseResponse execute(@RequestBody @Validated KillDto dto, BindingResult result, HttpSession session) {
        if (result.hasErrors() || dto.getKillId() <= 0) {
            return new BaseResponse(StatusCode.InvalidParams);
        }
//        Object uId = session.getAttribute("uid");
//        if (uId == null) {
//            return new BaseResponse(StatusCode.UserNotLogin);
//        }
        Integer userId=dto.getUserId();
//        Integer userId = (Integer) uId;

        BaseResponse response = new BaseResponse(StatusCode.Success);
        try {
            Boolean res = killService.killItem(dto.getKillId(), userId);
            if (!res) {
                return new BaseResponse(StatusCode.Fail.getCode(), "哈哈~商品已抢购完毕或者不在抢购时间段哦!");
            }
        } catch (Exception e) {
            response = new BaseResponse(StatusCode.Fail.getCode(), e.getMessage());
        }
        return response;
    }
```

#### Service

```java
@Override
public Boolean killItem(Integer killId, Integer userId) throws Exception {
    Boolean result=false;

    //TODO:判断当前用户是否已经抢购过当前商品
    if (itemKillSuccessMapper.countByKillUserId(killId,userId) <= 0){
        //TODO:查询待秒杀商品详情
        ItemKill itemKill=itemKillMapper.selectById(killId);

        //TODO:判断是否可以被秒杀canKill=1?
        if (itemKill!=null && 1==itemKill.getCanKill() ){
            //TODO:扣减库存-减一
            int res=itemKillMapper.updateKillItem(killId);
            System.out.println("kobekobekobekobekobe"+res);
            //TODO:扣减是否成功?是-生成秒杀成功的订单，同时通知用户秒杀成功的消息
            if (res>0){
                commonRecordKillSuccessInfo(itemKill,userId);
                result=true;
            }
        }
    }else{
        throw new Exception("您已经抢购过该商品了!");
    }
    return result;
}
```

```java
/**
     * 通用的方法-记录用户秒杀成功后生成的订单-并进行异步邮件消息的通知
     * @param kill
     * @param userId
     * @throws Exception
     */
    private void commonRecordKillSuccessInfo(ItemKill kill,Integer userId) throws Exception{
        //TODO:记录抢购成功后生成的秒杀订单记录
        ItemKillSuccess entity = new ItemKillSuccess();
        //传统随机数方法
//        entity.setCode(RandomUtil.generateOrderCode());
        //雪花算法
        String orderNo=String.valueOf(snowFlake.nextId());
        entity.setCode(orderNo);
        entity.setItemId(kill.getItemId());
        entity.setKillId(kill.getId());
        entity.setUserId(userId.toString());
        entity.setStatus(SysConstant.OrderStatus.SuccessNotPayed.getCode().byteValue());
        entity.setCreateTime(DateTime.now().toDate());
        int res = itemKillSuccessMapper.insert(entity);
        if(res>0){
            //TODO：进行异步邮件消息的通知=rabbitmq+mail
            rabbitSenderService.sendKillSuccessEmailMsg(orderNo);
            //TODO：入死信队列，用于"失效"超过指定的TTL时间时仍然未支付的订单
            rabbitSenderService.sendKillSuccessOrderExpireMsg(orderNo);
        }

    }
```

#### Mapper(Dao)

##### ItemKillMapper

```java
public interface ItemKillMapper {
    List<ItemKill> selectAll();

    ItemKill selectById(@Param("id") Integer id);

    int updateKillItem(@Param("killId") Integer killId);



    ItemKill selectByIdV2(@Param("id") Integer id);

    int updateKillItemV2(@Param("killId") Integer killId);
}
```

```xml
<!--抢购商品，剩余数量减一-->
<update id="updateKillItem">
  UPDATE item_kill
  SET total = total - 1
  WHERE
      id = #{killId}
</update>
```

##### ItemKillSuccessMapper

```java
public interface ItemKillSuccessMapper {
    int deleteByPrimaryKey(String code);

    int insert(ItemKillSuccess record);

    int insertSelective(ItemKillSuccess record);

    ItemKillSuccess selectByPrimaryKey(String code);

    int updateByPrimaryKeySelective(ItemKillSuccess record);

    int updateByPrimaryKey(ItemKillSuccess record);

    int countByKillUserId(@Param("killId") Integer killId, @Param("userId") Integer userId);

    KillSuccessUserInfo selectByCode(@Param("code") String code);

    int expireOrder(@Param("code") String code);

    List<ItemKillSuccess> selectExpireOrders();
}
```

```xml
<!--根据秒杀活动跟用户Id查询用户的抢购数量-->
<select id="countByKillUserId" resultType="java.lang.Integer">
  SELECT
      COUNT(1) AS total
  FROM
      item_kill_success
  WHERE
      user_id = #{userId}
  AND kill_id = #{killId}
  <!-- AND `status` IN (-1, 0) -->
  AND `status` IN (0)
</select>
```

### 订单编号的生成方式

一般情况，实现全局唯一ID，有三种方案，分别是通过中间件方式、UUID、雪花算法。

　　方案一，通过中间件方式，可以是把数据库或者redis缓存作为媒介，从中间件获取ID。这种呢，优点是可以体现全局的递增趋势（优点只能想到这个），缺点呢，倒是一大堆，比如，依赖中间件，假如中间件挂了，就不能提供服务了；依赖中间件的写入和事务，会影响效率；数据量大了的话，你还得考虑部署集群，考虑走代理。这样的话，感觉问题复杂化了

　　方案二，通过UUID的方式，java.util.UUID就提供了获取UUID的方法，使用UUID来实现全局唯一ID，优点是操作简单，也能实现全局唯一的效果，缺点呢，就是不能体现全局视野的递增趋势；太长了，UUID是32位，有点浪费；最重要的，是插入的效率低，因为呢，我们使用mysql的话，一般都是B+tree的结构来存储索引，假如是数据库自带的那种主键自增，节点满了，会裂变出新的节点，新节点满了，再去裂变新的节点，这样利用率和效率都很高。而UUID是无序的，会造成中间节点的分裂，也会造成不饱和的节点，插入的效率自然就比较低下了。

　　方案三，基于redis生成全局id策略，因为Redis是单线的天生保证原子性，可以使用原子性操作INCR和INCRBY来实现，注意在Redis集群情况下，同MySQL一样需要设置不同的增长步长，同时key一定要设置有效期，可以使用Redis集群来获取更高的吞吐量

　　方案四，通过snowflake算法

![image-20220412155210479](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412155210479.png)

#### 时间戳+N位流水号：

使用JDK自带的SimpleDateFormat

```java
public class RandomUtil {

    private static final SimpleDateFormat dateFormatOne=new SimpleDateFormat("yyyyMMddHHmmssSS");

    private static final ThreadLocalRandom random=ThreadLocalRandom.current();

    /**
     * 生成订单编号-方式一
     * @return
     */
    public static String generateOrderCode(){
        //TODO:时间戳+N为随机数流水号
        return dateFormatOne.format(DateTime.now().toDate()) + generateNumber(4);
    }

    //N为随机数流水号
    public static String generateNumber(final int num){
        StringBuffer sb = new StringBuffer();
        for(int i=1;i<=num;i++){
            sb.append(random.nextInt(9));
        }
        return sb.toString();
    }
}
```

#### 雪花算法

　SnowFlake算法生成id的结果是一个64bit大小的整数，它的结构如下图

![image-20220412163138453](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412163138453.png)

- `1位`，不用。二进制中最高位为1的都是负数，但是我们生成的id一般都使用整数，所以这个最高位固定是0
- `41位`，用来记录时间戳（毫秒）。
  - 41位可以表示$2^{41}-1$个数字，
  - 如果只用来表示正整数（计算机中正数包含0），可以表示的数值范围是：0 至 $2^{41}-1$，减1是因为可表示的数值范围是从0开始算的，而不是1。
  - 也就是说41位可以表示$2^{41}-1$个毫秒的值，转化成单位年则是$(2^{41}-1) / (1000 * 60 * 60 * 24 * 365) = 69$年
- `10位`，用来记录工作机器id。
  - 可以部署在$2^{10} = 1024$个节点，包括`5位datacenterId`和`5位workerId`
  - `5位（bit）`可以表示的最大正整数是$2^{5}-1 = 31$，即可以用0、1、2、3、....31这32个数字，来表示不同的datecenterId或workerId
- `12位`，序列号，用来记录同毫秒内产生的不同id。
  - `12位（bit）`可以表示的最大正整数是$2^{12}-1 = 4095$，即可以用0、1、2、3、....4094这4095个数字，来表示同一机器同一时间截（毫秒)内产生的4095个ID序号

　　由于在Java中64bit的整数是long类型，所以在Java中SnowFlake算法生成的id就是long来存储的。

　　SnowFlake可以保证：

- 所有生成的id按时间趋势递增
- 整个分布式系统内不会产生重复id（因为有datacenterId和workerId来做区分）

```java
public class SnowFlake {
    /**
     * 起始的时间戳
     */
    private final static long START_STAMP = 1480166465631L;

    /**
     * 每一部分占用的位数
     */
    private final static long SEQUENCE_BIT = 12; //序列号占用的位数
    private final static long MACHINE_BIT = 5;   //机器标识占用的位数
    private final static long DATA_CENTER_BIT = 5;//数据中心占用的位数

    /**
     * 每一部分的最大值
     */
    private final static long MAX_DATA_CENTER_NUM = -1L ^ (-1L << DATA_CENTER_BIT);
    private final static long MAX_MACHINE_NUM = -1L ^ (-1L << MACHINE_BIT);
    private final static long MAX_SEQUENCE = -1L ^ (-1L << SEQUENCE_BIT);

    /**
     * 每一部分向左的位移
     */
    private final static long MACHINE_LEFT = SEQUENCE_BIT;
    private final static long DATA_CENTER_LEFT = SEQUENCE_BIT + MACHINE_BIT;
    private final static long TIMESTAMP_LEFT = DATA_CENTER_LEFT + DATA_CENTER_BIT;

    private long dataCenterId;  //数据中心
    private long machineId;     //机器标识
    private long sequence = 0L; //序列号
    private long lastStamp = -1L;//上一次时间戳

    public SnowFlake(long dataCenterId, long machineId) {
        if (dataCenterId > MAX_DATA_CENTER_NUM || dataCenterId < 0) {
            throw new IllegalArgumentException("dataCenterId can't be greater than MAX_DATA_CENTER_NUM or less than 0");
        }
        if (machineId > MAX_MACHINE_NUM || machineId < 0) {
            throw new IllegalArgumentException("machineId can't be greater than MAX_MACHINE_NUM or less than 0");
        }
        this.dataCenterId = dataCenterId;
        this.machineId = machineId;
    }

    /**
     * 产生下一个ID
     *
     * @return
     */
    public synchronized long nextId() {
        long currStamp = getNewStamp();
        if (currStamp < lastStamp) {
            throw new RuntimeException("Clock moved backwards.  Refusing to generate id");
        }

        if (currStamp == lastStamp) {
            //相同毫秒内，序列号自增
            sequence = (sequence + 1) & MAX_SEQUENCE;
            //同一毫秒的序列数已经达到最大
            if (sequence == 0L) {
                currStamp = getNextMill();
            }
        } else {
            //不同毫秒内，序列号置为0
            sequence = 0L;
        }

        lastStamp = currStamp;

        return (currStamp - START_STAMP) << TIMESTAMP_LEFT //时间戳部分
                | dataCenterId << DATA_CENTER_LEFT       //数据中心部分
                | machineId << MACHINE_LEFT             //机器标识部分
                | sequence;                             //序列号部分
    }

    private long getNextMill() {
        long mill = getNewStamp();
        while (mill <= lastStamp) {
            mill = getNewStamp();
        }
        return mill;
    }

    private long getNewStamp() {
        return System.currentTimeMillis();
    }
}
```

### Rabbitmq实现消息异步

![image-20220412164031067](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412164031067.png)

死信队列消息模型

![image-20220412170853730](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412170853730.png)

#### 配置信息

```java
#rabbitmq
spring.rabbitmq.virtual-host=/
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
#消费者实例数量 
spring.rabbitmq.listener.simple.concurrency=5
spring.rabbitmq.listener.simple.max-concurrency=15
#消费者拉取消息数量
spring.rabbitmq.listener.simple.prefetch=10
    
mq.env=test

#秒杀成功异步发送邮件的消息模型
mq.kill.item.success.email.queue=${mq.env}.kill.item.success.email.queue
mq.kill.item.success.email.exchange=${mq.env}.kill.item.success.email.exchange
mq.kill.item.success.email.routing.key=${mq.env}.kill.item.success.email.routing.key

#订单超时未支付自动失效-死信队列消息模型
mq.kill.item.success.kill.dead.queue=${mq.env}.kill.item.success.kill.dead.queue
mq.kill.item.success.kill.dead.exchange=${mq.env}.kill.item.success.kill.dead.exchange
mq.kill.item.success.kill.dead.routing.key=${mq.env}.kill.item.success.kill.dead.routing.key

mq.kill.item.success.kill.dead.real.queue=${mq.env}.kill.item.success.kill.dead.real.queue
mq.kill.item.success.kill.dead.prod.exchange=${mq.env}.kill.item.success.kill.dead.prod.exchange
mq.kill.item.success.kill.dead.prod.routing.key=${mq.env}.kill.item.success.kill.dead.prod.routing.key

#单位为ms
#mq.kill.item.success.kill.expire=10000
mq.kill.item.success.kill.expire=1800000
scheduler.expire.orders.time=30
```

#### 配置类

```java
/**
 * 通用化 Rabbitmq 配置
 */
@Configuration
public class RabbitmqConfig {
    private final static Logger log = LoggerFactory.getLogger(RabbitmqConfig.class);

    @Autowired
    private Environment env;

    //链接工厂，建立一个通道连接
    @Autowired
    private CachingConnectionFactory connectionFactory;

    //消费者实例，用来配置高并发下的消费者
    @Autowired
    private SimpleRabbitListenerContainerFactoryConfigurer factoryConfigurer;

    /**
     * 单一消费者
     * @return
     */
    @Bean(name = "singleListenerContainer")
    public SimpleRabbitListenerContainerFactory listenerContainer(){
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        //用Json传输
        factory.setMessageConverter(new Jackson2JsonMessageConverter());
        factory.setConcurrentConsumers(1);
        factory.setMaxConcurrentConsumers(1);
        factory.setPrefetchCount(1);
        factory.setTxSize(1);
        return factory;
    }

    /**
     * 多个消费者
     * @return
     */
    @Bean(name = "multiListenerContainer")
    public SimpleRabbitListenerContainerFactory multiListenerContainer(){
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factoryConfigurer.configure(factory,connectionFactory);
        factory.setMessageConverter(new Jackson2JsonMessageConverter());
        //确认消费模式-NONE
        factory.setAcknowledgeMode(AcknowledgeMode.NONE);
        //设置高并发下用多少个消费者实例去消费消息，提高吞吐量
        factory.setConcurrentConsumers(env.getProperty("spring.rabbitmq.listener.simple.concurrency",int.class));
        factory.setMaxConcurrentConsumers(env.getProperty("spring.rabbitmq.listener.simple.max-concurrency",int.class));
        factory.setPrefetchCount(env.getProperty("spring.rabbitmq.listener.simple.prefetch",int.class));
        return factory;
    }

    /**
     * 核心组件
     * @return
     */
    @Bean
    public RabbitTemplate rabbitTemplate(){
        connectionFactory.setPublisherConfirms(true);
        connectionFactory.setPublisherReturns(true);
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        rabbitTemplate.setMandatory(true);
        rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            @Override
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                log.info("消息发送成功:correlationData({}),ack({}),cause({})",correlationData,ack,cause);
            }
        });
        rabbitTemplate.setReturnCallback(new RabbitTemplate.ReturnCallback() {
            @Override
            public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
                log.warn("消息丢失:exchange({}),route({}),replyCode({}),replyText({}),message:{}",exchange,routingKey,replyCode,replyText,message);
            }
        });
        return rabbitTemplate;
    }


    //构建异步发送邮箱通知的消息模型
    @Bean
    public Queue successEmailQueue(){
        return new Queue(env.getProperty("mq.kill.item.success.email.queue"),true);
    }

    @Bean
    public TopicExchange successEmailExchange(){
        return new TopicExchange(env.getProperty("mq.kill.item.success.email.exchange"),true,false);
    }

    @Bean
    public Binding successEmailBinding(){
        return BindingBuilder.bind(successEmailQueue()).to(successEmailExchange()).with(env.getProperty("mq.kill.item.success.email.routing.key"));
    }


    //构建秒杀成功之后-订单超时未支付的死信队列消息模型

    @Bean
    public Queue successKillDeadQueue(){
        Map<String, Object> argsMap= Maps.newHashMap();
        argsMap.put("x-dead-letter-exchange",env.getProperty("mq.kill.item.success.kill.dead.exchange"));
        argsMap.put("x-dead-letter-routing-key",env.getProperty("mq.kill.item.success.kill.dead.routing.key"));
        return new Queue(env.getProperty("mq.kill.item.success.kill.dead.queue"),true,false,false,argsMap);
    }

    //基本交换机
    @Bean
    public TopicExchange successKillDeadProdExchange(){
        return new TopicExchange(env.getProperty("mq.kill.item.success.kill.dead.prod.exchange"),true,false);
    }

    //创建基本交换机+基本路由 -> 死信队列 的绑定
    @Bean
    public Binding successKillDeadProdBinding(){
        return BindingBuilder.bind(successKillDeadQueue()).to(successKillDeadProdExchange()).with(env.getProperty("mq.kill.item.success.kill.dead.prod.routing.key"));
    }

    //真正的队列
    @Bean
    public Queue successKillRealQueue(){
        return new Queue(env.getProperty("mq.kill.item.success.kill.dead.real.queue"),true);
    }

    //死信交换机
    @Bean
    public TopicExchange successKillDeadExchange(){
        return new TopicExchange(env.getProperty("mq.kill.item.success.kill.dead.exchange"),true,false);
    }

    //死信交换机+死信路由->真正队列 的绑定
    @Bean
    public Binding successKillDeadBinding(){
        return BindingBuilder.bind(successKillRealQueue()).to(successKillDeadExchange()).with(env.getProperty("mq.kill.item.success.kill.dead.routing.key"));
    }
}
```

#### 发送类

##### 邮件发送

```java
/**
 * 秒杀成功异步发送邮件通知消息
 */
public void sendKillSuccessEmailMsg(String orderNo){
    log.info("秒杀成功异步发送邮件通知消息-准备发送消息：{}",orderNo);
    try {
        if (StringUtils.isNotBlank(orderNo)){
            KillSuccessUserInfo info=itemKillSuccessMapper.selectByCode(orderNo);
            if (info!=null){
                //TODO:rabbitmq发送消息的逻辑
                rabbitTemplate.setMessageConverter(new Jackson2JsonMessageConverter());
                rabbitTemplate.setExchange(env.getProperty("mq.kill.item.success.email.exchange"));
                rabbitTemplate.setRoutingKey(env.getProperty("mq.kill.item.success.email.routing.key"));
                //TODO：将info充当消息发送至队列
                rabbitTemplate.convertAndSend(info, new MessagePostProcessor() {
                    @Override
                    public Message postProcessMessage(Message message) throws AmqpException {
                        MessageProperties messageProperties=message.getMessageProperties();
                        messageProperties.setDeliveryMode(MessageDeliveryMode.PERSISTENT);
                       messageProperties.setHeader(AbstractJavaTypeMapper.DEFAULT_CONTENT_CLASSID_FIELD_NAME,KillSuccessUserInfo.class);
                        return message;
                    }
                });
            }
        }
    }catch (Exception e){
        log.error("秒杀成功异步发送邮件通知消息-发生异常，消息为：{}",orderNo,e.fillInStackTrace());
    }
}
```

##### 发送信息进入死信队列

```java
/**
 * 秒杀成功后生成抢购订单-发送信息入死信队列，等待着一定时间失效超时未支付的订单
 * @param orderCode
 */
public void sendKillSuccessOrderExpireMsg(final String orderCode){
    try {
        if (StringUtils.isNotBlank(orderCode)){
            KillSuccessUserInfo info=itemKillSuccessMapper.selectByCode(orderCode);
            if (info!=null){
                rabbitTemplate.setMessageConverter(new Jackson2JsonMessageConverter());
                rabbitTemplate.setExchange(env.getProperty("mq.kill.item.success.kill.dead.prod.exchange"));
                rabbitTemplate.setRoutingKey(env.getProperty("mq.kill.item.success.kill.dead.prod.routing.key"));
                rabbitTemplate.convertAndSend(info, new MessagePostProcessor() {
                    @Override
                    public Message postProcessMessage(Message message) throws AmqpException {
                        MessageProperties mp=message.getMessageProperties();
                        mp.setDeliveryMode(MessageDeliveryMode.PERSISTENT);
                        mp.setHeader(AbstractJavaTypeMapper.DEFAULT_CONTENT_CLASSID_FIELD_NAME,KillSuccessUserInfo.class);
                        //TODO：动态设置TTL(为了测试方便，暂且设置10s)
                        mp.setExpiration(env.getProperty("mq.kill.item.success.kill.expire"));
                        return message;
                    }
                });
            }
        }
    }catch (Exception e){
        log.error("秒杀成功后生成抢购订单-发送信息入死信队列，等待着一定时间失效超时未支付的订单-发生异常，消息为：{}",orderCode,e.fillInStackTrace());
    }
}
```

#### 接收类

##### 邮件接收

```java
    /**
     * 秒杀异步邮件通知-接收消息
     */
    @RabbitListener(queues = {"${mq.kill.item.success.email.queue}"},containerFactory = "singleListenerContainer")
    public void consumeEmailMsg(KillSuccessUserInfo info){
        try {
            log.info("秒杀异步邮件通知-接收消息:{}",info);
            //TODO:真正的发送邮件....
            //发送简单文件
/*            MailDto dto=new MailDto(env.getProperty("mail.kill.item.success.subject"),"这是测试内容",new String[]{info.getEmail()});
            mailService.sendSimpleEmail(dto);*/
            //发生花式文件
            final String content=String.format(env.getProperty("mail.kill.item.success.content"),info.getItemName(),info.getCode());
            MailDto dto=new MailDto(env.getProperty("mail.kill.item.success.subject"),content,new String[]{info.getEmail()});
            mailService.sendHTMLMail(dto);
        }catch (Exception e){
            log.error("秒杀异步邮件通知-接收消息-发生异常：",e.fillInStackTrace());
        }
    }
```

##### 订单过期监听器

```java
/**
 * 用户秒杀成功后超时未支付-监听者
 * @param info
 */
@RabbitListener(queues = {"${mq.kill.item.success.kill.dead.real.queue}"},containerFactory = "singleListenerContainer")
public void consumeExpireOrder(KillSuccessUserInfo info){
    try {
        log.info("用户秒杀成功后超时未支付-监听者-接收消息:{}",info);
        if (info!=null){
            ItemKillSuccess entity=itemKillSuccessMapper.selectByPrimaryKey(info.getCode());
            if (entity!=null && entity.getStatus().intValue()==0){
                itemKillSuccessMapper.expireOrder(info.getCode());
            }
        }
    }catch (Exception e){
        log.error("用户秒杀成功后超时未支付-监听者-发生异常：",e.fillInStackTrace());
    }
}
```

#### 发送错误以及解决方式

![image-20220412170516202](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412170516202.png)

没有绑定到对应的队列上，愿意是没有加上@bean注解

### 邮件发送

![image-20220412170749772](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412170749772.png)

```java
@Data
@ToString
@AllArgsConstructor
@NoArgsConstructor
public class MailDto {
    //邮件主题
    private String subject;
    //邮件内容
    private String content;
    //接收人
    private String[] tos;
}
```

```java
@Service
@EnableAsync
public class MailService {
    private static final Logger log= LoggerFactory.getLogger(MailService.class);
    @Autowired
    private JavaMailSender mailSender;
    @Autowired
    private Environment env;
    /**
     * 发送简单文本文件
     */
    @Async
    public void sendSimpleEmail(final MailDto dto){
        try {
            SimpleMailMessage message=new SimpleMailMessage();
            message.setFrom(env.getProperty("mail.send.from"));
            message.setFrom(env.getProperty("mail.send.from"));
            message.setTo(dto.getTos());
            message.setSubject(dto.getSubject());
            message.setText(dto.getContent());
            mailSender.send(message);
            log.info("发送简单文本文件-发送成功!");
        }catch (Exception e){
            log.error("发送简单文本文件-发生异常： ",e.fillInStackTrace());
        }
    }
    /**
     * 发送花哨邮件
     * @param dto
     */
    @Async
    public void sendHTMLMail(final MailDto dto){
        try {
            MimeMessage message=mailSender.createMimeMessage();
            MimeMessageHelper messageHelper=new MimeMessageHelper(message,true,"utf-8");
            messageHelper.setFrom(env.getProperty("mail.send.from"));
            messageHelper.setTo(dto.getTos());
            messageHelper.setSubject(dto.getSubject());
            messageHelper.setText(dto.getContent(),true);
            mailSender.send(message);
            log.info("发送花哨邮件-发送成功!");
        }catch (Exception e){
            log.error("发送花哨邮件-发生异常： ",e.fillInStackTrace());
        }
    }
}
```

#### 邮件配置

```java
#发送邮件配置
spring.mail.host=smtp.qq.com
spring.mail.username=kobe1978mvp
spring.mail.password=esacqndqfdckbbgh
    
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
spring.mail.properties.mail.smtp.starttls.required=true
mail.send.from=kobe1978mvp@qq.com

mail.kill.item.success.subject=商品抢购成功
mail.kill.item.success.content=您好，您已成功抢购到商品: <strong style="color: red">%s</strong> ，复制该链接并在浏览器采用新的页面打开，即可查看抢购详情：${system.domain.url}/kill/record/detail/%s，并请您在1个小时内完成订单的支付，超时将失效该订单哦！祝你生活愉快！
```

#### 发送错误及解决方式

<img src="C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412170127363.png" alt="image-20220412170127363" style="zoom:200%;" />

解决方式 将 spring.mail.username=kobe1978mvp 后缀的@qq.com去掉，因为已经在spring.mail.host上配置了qq.com

## 订单详情展示

### Comtroller

```java
/**
 * 查看订单详情
 * @return
 */
@RequestMapping(value = prefix+"/record/detail/{orderNo}",method = RequestMethod.GET)
public String killRecordDetail(@PathVariable String orderNo, ModelMap modelMap){
    if (StringUtils.isBlank(orderNo)){
        return "error";
    }
    KillSuccessUserInfo info=itemKillSuccessMapper.selectByCode(orderNo);
    if (info==null){
        return "error";
    }
    modelMap.put("info",info);
    return "killRecord";
}
```

### Service  没写

### Mapper(Dao)

```java
<!--根据秒杀成功后的订单编码查询-->
<select id="selectByCode" resultType="com.kobe.kill.model.dto.KillSuccessUserInfo">
  SELECT
    a.*,
    b.user_name,
    b.phone,
    b.email,
    c.name AS itemName
  FROM item_kill_success AS a
    LEFT JOIN user b ON b.id = a.user_id
    LEFT JOIN item c ON c.id = a.item_id
  WHERE a.code = #{code}
        AND b.is_active = 1
</select>
```

## 用户登录（Shiro)

![image-20220413102100280](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220413102100280.png)

配置类

```java
@Configuration
public class ShiroConfig {
    @Bean
    public CustomRealm customRealm(){
        return new CustomRealm();
    }
    @Bean
    public SecurityManager securityManager(){
        DefaultWebSecurityManager securityManager=new DefaultWebSecurityManager();
        securityManager.setRealm(customRealm());
        return securityManager;
    }
    @Bean
    public ShiroFilterFactoryBean shiroFilterFactoryBean(){
        ShiroFilterFactoryBean bean=new ShiroFilterFactoryBean();
        bean.setSecurityManager(securityManager());
        bean.setLoginUrl("/to/login");
        bean.setUnauthorizedUrl("/unauth");
        //过滤器链
        Map<String, String> filterChainDefinitionMap=new HashMap<>();
        filterChainDefinitionMap.put("/to/login","anon");
        filterChainDefinitionMap.put("/**","anon");
        filterChainDefinitionMap.put("/kill/execute/*","authc");
        filterChainDefinitionMap.put("/item/detail/*","authc");
        bean.setFilterChainDefinitionMap(filterChainDefinitionMap);
        return bean;
    }
}
```

Controller

对数据库中的用户的密码进行加盐处理  Token中存储了用户名和用户密码 

```java
@Controller
public class UserController {
    private static final Logger log= LoggerFactory.getLogger(UserController.class);
    @Autowired
    private Environment env;
    /**
     * 跳到登录页
     * @return
     */
    @RequestMapping(value = {"/to/login","/unauth"})
    public String toLogin(){
        return "login";
    }
    /**
     * 登录认证
     * @param userName
     * @param password
     * @param modelMap
     * @return
     */
    @RequestMapping(value = "/login",method = RequestMethod.POST)
    public String login(@RequestParam String userName, @RequestParam String password, ModelMap modelMap){
        String errorMsg="";
        try {
            if (!SecurityUtils.getSubject().isAuthenticated()){
                String newPsd=new Md5Hash(password,env.getProperty("shiro.encrypt.password.salt")).toString();
                UsernamePasswordToken token=new UsernamePasswordToken(userName,newPsd);
                SecurityUtils.getSubject().login(token);
            }
        }catch (UnknownAccountException e){
            errorMsg=e.getMessage();
            modelMap.addAttribute("userName",userName);
        }catch (DisabledAccountException e){
            errorMsg=e.getMessage();
            modelMap.addAttribute("userName",userName);
        }catch (IncorrectCredentialsException e){
            errorMsg=e.getMessage();
            modelMap.addAttribute("userName",userName);
        }catch (Exception e){
            errorMsg="用户登录异常，请联系管理员!";
            e.printStackTrace();
        }
        if (StringUtils.isBlank(errorMsg)){
            return "redirect:/index";
        }else{
            modelMap.addAttribute("errorMsg",errorMsg);
            return "login";
        }
    }
    /**
     * 退出登录
     * @return
     */
    @RequestMapping(value = "/logout")
    public String logout(){
        SecurityUtils.getSubject().logout();
        return "login";
    }
}
```

![image-20220413110838915](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220413110838915.png)

Service

```java
/**
 * 用户自定义的realm-用于shiro的认证、授权
 **/
public class CustomRealm extends AuthorizingRealm {
    private static final Logger log= LoggerFactory.getLogger(CustomRealm.class);
    private static final Long sessionKeyTimeOut=3600_000L;
    @Autowired
    private UserMapper userMapper;
    /**
     * 授权
     * @param principalCollection
     * @return
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        return null;
    }
    /**
     * 认证-登录
     * @param authenticationToken
     * @return
     * @throws AuthenticationException
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
        UsernamePasswordToken token= (UsernamePasswordToken) authenticationToken;
        String userName=token.getUsername();
        String password=String.valueOf(token.getPassword());
        log.info("当前登录的用户名={} 密码={} ",userName,password);
        User user=userMapper.selectByUserName(userName);
        if (user==null){
            throw new UnknownAccountException("用户名不存在!");
        }
        if (!Objects.equals(1,user.getIsActive().intValue())){
            throw new DisabledAccountException("当前用户已被禁用!");
        }
        if (!user.getPassword().equals(password)){
            throw new IncorrectCredentialsException("用户名密码不匹配!");
        }
        SimpleAuthenticationInfo info=new SimpleAuthenticationInfo(user.getUserName(),password,getName());
        setSession("uid",user.getId());
        return info;
    }
    /**
     * 将key与对应的value塞入shiro的session中-最终交给HttpSession进行管理(如果是分布式session配置，那么就是交给redis管理)
     * @param key
     * @param value
     */
    private void setSession(String key,Object value){
        Session session= SecurityUtils.getSubject().getSession();
        if (session!=null){
            session.setAttribute(key,value);
            session.setTimeout(sessionKeyTimeOut);
        }
    }
}
```

# Jmeter压力测试

配置Http请求，CSV数据文件设置 HTTP信息投管理器，查看结果树

![image-20220412172152320](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412172152320.png)

![image-20220412172256588](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412172256588.png)

## 测试结果

在线程数开启为1000的情况下，商品数量为50，用户id数为5，出现了超卖的现象：

![image-20220412172457334](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412172457334.png)

多个用户购买了同一个商品

![image-20220412172629608](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412172629608.png)

有些请求出现了数据格式的错误

![image-20220412172655004](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412172655004.png)

或者是出现已经抢购过的提示

![image-20220412172715527](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412172715527.png)

## 分析

在多线程情况下，sql语句不严禁多个线程进行减1操作

```mysql
<!--抢购商品，剩余数量减一-->
<update id="updateKillItem">
  UPDATE item_kill
  SET total = total - 1
  WHERE
      id = #{killId}
</update>
```

# 系统优化

在生产环境中，如果既要控制并发的安全，又要qps不高，可以选择zookeeper，如果既要控制并发安全，也要保证快速处理，可以使用redis和redisson

![image-20220412171701610](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412171701610.png)

## 定时任务失效超时未支付的订单

![image-20220412171041269](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412171041269.png)

使用cron表达式实现30分钟的定时任务

```java
/**
 * 定时获取status=0的订单并判断是否超过TTL，然后进行失效
 */
//@Scheduled(cron = "0/10 * * * * ?")
@Scheduled(cron = "0 0/30 * * * ?")
public void schedulerExpireOrders(){
    try {
        List<ItemKillSuccess> list=itemKillSuccessMapper.selectExpireOrders();
        if (list!=null && !list.isEmpty()){
            //java8的写法
            list.stream().forEach(i -> {
                if (i!=null && i.getDiffTime() > env.getProperty("scheduler.expire.orders.time",Integer.class)){
                    itemKillSuccessMapper.expireOrder(i.getCode());
                }
            });
        }
    }catch (Exception e){
        log.error("定时获取status=0的订单并判断是否超过TTL，然后进行失效-发生异常：",e.fillInStackTrace());
    }
}
```

```xml
<!--批量获取待处理的已保存订单记录-->
<select id="selectExpireOrders" resultType="com.kobe.kill.model.entity.ItemKillSuccess">
  SELECT
      a.*,TIMESTAMPDIFF(MINUTE,a.create_time,NOW()) AS diffTime
  FROM
      item_kill_success AS a
  WHERE
      a.`status` = 0
</select>
```

```xml
<!--失效更新订单信息-->
<update id="expireOrder">
  UPDATE item_kill_success
  SET status = -1
  WHERE code = #{code} AND status = 0
</update>
```

## 数据库Mysql层面优化

![image-20220412195725778](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412195725778.png)

service

```java
/**
 * 商品秒杀核心业务逻辑的处理-mysql的优化
 * @param killId
 * @param userId
 * @return
 * @throws Exception
 */
@Override
public Boolean killItemV2(Integer killId, Integer userId) throws Exception {
    Boolean result=false;
    //TODO:判断当前用户是否已经抢购过当前商品
    if (itemKillSuccessMapper.countByKillUserId(killId,userId) <= 0){
        //TODO:A.查询待秒杀商品详情
        ItemKill itemKill=itemKillMapper.selectByIdV2(killId);
        //TODO:判断是否可以被秒杀canKill=1?
        if (itemKill!=null && 1==itemKill.getCanKill() && itemKill.getTotal()>0){
            //TODO:B.扣减库存-减一
            int res=itemKillMapper.updateKillItemV2(killId);
            //TODO:扣减是否成功?是-生成秒杀成功的订单，同时通知用户秒杀成功的消息
            if (res>0){
                commonRecordKillSuccessInfo(itemKill,userId);
                result=true;
            }
        }
    }else{
        throw new Exception("您已经抢购过该商品了!");
    }
    return result;
}
```

xml

```mysql
<!--获取秒杀详情V2-->
<select id="selectByIdV2" resultType="com.kobe.kill.model.entity.ItemKill">
  SELECT
    a.*,
    b.name AS itemName,
    (CASE WHEN (now() BETWEEN a.start_time AND a.end_time)
      THEN 1
     ELSE 0
     END)  AS canKill
  FROM item_kill AS a LEFT JOIN item AS b ON b.id = a.item_id
  WHERE a.is_active = 1 AND a.id =#{id} AND a.total>0
</select>
```

```mysql
<!--抢购商品，剩余数量减一-->
<update id="updateKillItemV2">
  UPDATE item_kill
  SET total = total - 1
  WHERE id = #{killId} AND total>0
</update>
```

### 测试分析

在线程数开启为5000的情况下进行测试

商品数结果为0

![image-20220412201145313](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412201145313.png)

但是仍然出现同一用户抢购了多个秒杀商品

![image-20220412201221860](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412201221860.png)

### 双重校验锁再度优化

类似于单例模式的双重校验

![image-20220412203313465](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412203313465.png)

![image-20220412203227542](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412203227542.png)

测试分析  

在一定程度上进行了优化，但是还是会出现商品扣减多于用户数的情况，因为写的时间如果慢于查的时间的话，还是会出现问题

在线程数为1000的情况下 ，有一定的效果，但是还是出现了扣减商品为6个 实际订单数为5个情况

![image-20220412204003916](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412204003916.png)

![image-20220412204055082](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412204055082.png)

在线程数为5000的情况 扣减商品为10个，实际订单为6个

![image-20220412204244554](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412204244554.png)

![image-20220412204258398](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412204258398.png)

## 基于Redis的分布式锁优化

**一般秒杀优化方案**：一般的系统优化秒杀是这样做的，做一个原子的计数器，这个原子计数器可以通过一些redis/NoSQL来实现，这个原子计数器就是这个商品的库存，当一个用户进来，执行秒杀的时候，他会去减库存，那么就是减这个原子计数器，当我减原子计数器成功之后，会去记录这个行为，作为一个消息去放到一个分布式的MQ当中，然后后端的服务会去消费消息并落地,一般落地到MySQL当中。比如说他会记录谁购买了这个商品，并记录。

​			这个架构的优势在于能够抗住非常高的并发

痛点在于：不够稳定，工程师要对这些组件非常熟悉，熟悉他们的数据一致性模型，以及了解自己的逻辑应该怎么处理回滚，比如说当我们的减库存失败了，MQ访问超时了。还有就是幂等性难保证：重复秒杀问题，就是当它减库存的时候，它不知道用户之前有没有减库存，这时候他就会发一个消息，然后告诉它这个用户又去发送一个秒杀请求。同时也是一个不适合新手的架构。

利用Redis的原子操作

![image-20220412205343348](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412205343348.png)

Redis Setnx（ **SET** if **N**ot e**X**ists ）命令在指定的 key 不存在时，为 key 设置指定的值，这种情况下等同 [SET](https://www.redis.com.cn/commands/set.html) 命令。当 `key`存在时，什么也不做。

![image-20220412211332182](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412211332182.png)

redis实现分布式锁可以采用ValueOperations.setIfAbsent(key, value)或RedisConnection.setNX(key, value)方法

　　ValueOperations.setIfAbsent(key, value)封装了RedisConnection.setNX(key, value)

　　该方法用意：

　　　　并发或非并发都只允许一个插入成功。

　　　　如果key存在，插入值失败，返回false，可以循环执行，直至成功或超时。

　　　　如果key不存，插入值成功，返回true，业务逻辑处理结束后，将key删除。

```java
//TODO:借助Redis的原子操作实现分布式锁-对共享操作-资源进行控制
ValueOperations valueOperations=stringRedisTemplate.opsForValue();
final String key=new StringBuffer().append(killId).append(userId).append("-RedisLock").toString();
final String value=RandomUtil.generateOrderCode();
Boolean cacheRes=valueOperations.setIfAbsent(key,value); //luna脚本提供“分布式锁服务”，就可以写在一起
```

### 配置文件

```mysql
#redis
spring.redis.host=127.0.0.1
spring.redis.port=6379
#spring.redis.password=
redis.config.host=redis://127.0.0.1:6379
```

```java
@Configuration
public class RedisConfig {
    @Autowired
    private RedisConnectionFactory redisConnectionFactory;
    @Bean
    public RedisTemplate<String,Object> redisTemplate(){
        RedisTemplate<String,Object> redisTemplate=new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        //TODO:指定Key、Value的序列化策略
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new JdkSerializationRedisSerializer());
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        return redisTemplate;
    }
    @Bean
    public StringRedisTemplate stringRedisTemplate(){
        StringRedisTemplate stringRedisTemplate=new StringRedisTemplate();
        stringRedisTemplate.setConnectionFactory(redisConnectionFactory);
        return stringRedisTemplate;
    }
}
```

### Service

```java
/**
 * 商品秒杀核心业务逻辑的处理-redis的分布式锁
 * @param killId
 * @param userId
 * @return
 * @throws Exception
 */
@Override
public Boolean killItemV3(Integer killId, Integer userId) throws Exception {
    Boolean result=false;
    if (itemKillSuccessMapper.countByKillUserId(killId,userId) <= 0){
        //TODO:借助Redis的原子操作实现分布式锁-对共享操作-资源进行控制
        ValueOperations valueOperations=stringRedisTemplate.opsForValue();
        //key通过killID+userID+后缀来构造
        final String key=new StringBuffer().append(killId).append(userId).append("-RedisLock").toString();
        //使用随机数来生产Value
        final String value=RandomUtil.generateOrderCode();
        Boolean cacheRes=valueOperations.setIfAbsent(key,value); //luna脚本提供“分布式锁服务”，就可以写在一起
        //TOOD:redis部署节点宕机了
        if (cacheRes){
            //30秒后进行释放
            stringRedisTemplate.expire(key,30, TimeUnit.SECONDS);
            try {
                ItemKill itemKill=itemKillMapper.selectByIdV2(killId);
                if (itemKill!=null && 1==itemKill.getCanKill() && itemKill.getTotal()>0){
                    int res=itemKillMapper.updateKillItemV2(killId);
                    if (res>0){
                        commonRecordKillSuccessInfo(itemKill,userId);
                        result=true;
                    }
                }
            }catch (Exception e){
                throw new Exception("还没到抢购日期、已过了抢购时间或已被抢购完毕！");
            }finally {
                //释放锁
                if (value.equals(valueOperations.get(key).toString())){
                    stringRedisTemplate.delete(key);
                }
            }
        }
    }else{
        throw new Exception("Redis-您已经抢购过该商品了!");
    }
    return result;
}
```

### 测试结果

成功保证了一致性

![image-20220412212203948](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412212203948.png)

![image-20220412212228294](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412212228294.png)

### 结果分析

当 Boolean cacheRes=valueOperations.setIfAbsent(key,value)这条语句执行完后，redis部署节点宕机，

会出现  stringRedisTemplate.expire(key,30, TimeUnit.SECONDS)执行不了的情况，相应的stringRedisTemplate.delete（key)操作也执行不了

cacheRes会一直出现在Redis数据库里面，等待再次开启的时候就会返回false,这就会出现key锁死的情况

## 基于Redisson的分布式锁

为了避免key锁死的情况发生，使用Redison进行优化

![image-20220412215253024](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412215253024.png)

### 配置类

```java
@Configuration
public class RedissonConfig {
    @Autowired
    private Environment env;

    @Bean
    public RedissonClient redissonClient(){
        Config config=new Config();
        config.useSingleServer()
                .setAddress(env.getProperty("redis.config.host"))
                .setPassword(env.getProperty("spring.redis.password"));
        RedissonClient client= Redisson.create(config);
        return client;
    }
}
```

### Service

```java
 @Override
    public Boolean killItemV4(Integer killId, Integer userId) throws Exception {
        Boolean result=false;
        //设置锁
        final String lockKey=new StringBuffer().append(killId).append(userId).append("-RedissonLock").toString();
        RLock lock=redissonClient.getLock(lockKey);
        try {
            //可重入锁，最多等待30秒，上锁以后10秒自动解锁
            Boolean cacheRes=lock.tryLock(30,10,TimeUnit.SECONDS);
            if (cacheRes){
                //TODO:核心业务逻辑的处理
                if (itemKillSuccessMapper.countByKillUserId(killId,userId) <= 0){
                    ItemKill itemKill=itemKillMapper.selectByIdV2(killId);
                    if (itemKill!=null && 1==itemKill.getCanKill() && itemKill.getTotal()>0){
                        int res=itemKillMapper.updateKillItemV2(killId);
                        if (res>0){
                            commonRecordKillSuccessInfo(itemKill,userId);
                            result=true;
                        }
                    }
                }else{
                    throw new Exception("redisson-您已经抢购过该商品了!");
                }
            }
        }finally {
            lock.unlock();
            //lock.forceUnlock();
        }
        return result;
    }
```

### 出现问题和解决

Unable to connect to Redis server: 127.0.0.1/127.0.0.1:6379 

找到问题了，我的redis没有密码，把在配置文件里面注释掉Password

```java
#spring.redis.password=
```

## 基于Zookeeper的分布式锁

​       临时有序节点，在同一时刻只会创建一个序号最小的节点，由于每一个线程会创建一个节点，所以它的速度没有redis和redisson那么快

![image-20220412220501835](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220412220501835.png)

配置类

```java
@Configuration
public class ZookeeperConfig {
    @Autowired
    private Environment env;
    /**
     * 自定义注入ZooKeeper客户端操作实例
     * @return
     */
    @Bean
    public CuratorFramework curatorFramework(){
        CuratorFramework curatorFramework= CuratorFrameworkFactory.builder()
                .connectString(env.getProperty("zk.host"))
                .namespace(env.getProperty("zk.namespace"))
                //重试策略
                .retryPolicy(new RetryNTimes(5,1000))
                .build();
        curatorFramework.start();
        return curatorFramework;
    }
}

#zookeeper
zk.host=127.0.0.1:2181
zk.namespace=kill
```

Service

```java
  @Autowired
    private CuratorFramework curatorFramework;
    private static final String pathPrefix="/kill/zkLock/";
/**
 * 商品秒杀核心业务逻辑的处理-基于ZooKeeper的分布式锁
 * @param killId
 * @param userId
 * @return
 * @throws Exception
 */
@Override
public Boolean killItemV5(Integer killId, Integer userId) throws Exception {
    Boolean result=false;
    InterProcessMutex mutex=new InterProcessMutex(curatorFramework,pathPrefix+killId+userId+"-lock");
    try {
        if (mutex.acquire(10L,TimeUnit.SECONDS)){
            //TODO:核心业务逻辑
            if (itemKillSuccessMapper.countByKillUserId(killId,userId) <= 0){
                ItemKill itemKill=itemKillMapper.selectByIdV2(killId);
                if (itemKill!=null && 1==itemKill.getCanKill() && itemKill.getTotal()>0){
                    int res=itemKillMapper.updateKillItemV2(killId);
                    if (res>0){
                        commonRecordKillSuccessInfo(itemKill,userId);
                        result=true;
                    }
                }
            }else{
                throw new Exception("zookeeper-您已经抢购过该商品了!");
            }
        }
    }catch (Exception e){
        throw new Exception("还没到抢购日期、已过了抢购时间或已被抢购完毕！");
    }finally {
        if (mutex!=null){
            mutex.release();
        }
    }
    return result;
}
```

## 其他优化

![image-20220413100151591](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220413100151591.png)

![image-20220413114029025](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220413114029025.png)