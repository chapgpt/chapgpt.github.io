---
layout: post
title: rabbitmq的使用
date: 2020-06-29 10:32:20 +0300
description: You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. # Add post description (optional)
img: i-rest.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [Holidays, Hawaii]


普通生产者消费者模型
连接类：
public class MqConnectionUtil {
	
	 public static Connection getConnection() throws Exception {
	        //定义连接工厂
	        ConnectionFactory factory = new ConnectionFactory();
	        //设置服务地址
	        factory.setHost("192.168.217.128");
	        //端口
	        factory.setPort(5672);
	        //设置账号信息，用户名、密码、vhost
	        factory.setVirtualHost("admintest");
	        factory.setUsername("admin");
	        factory.setPassword("admin");
	        // 通过工程获取连接
	        Connection connection = factory.newConnection();
	        return connection;
	    }

}
消费者：
public class Consumer {
	private final static String QUEUE_NAME = "mq_test_01";

    public static void main(String[] argv) throws Exception {

        // 获取到连接以及mq通道
        Connection connection = MqConnectionUtil.getConnection();
        // 从连接中创建通道
        Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 定义队列的消费者
        QueueingConsumer consumer = new QueueingConsumer(channel);
        // 监听队列
        channel.basicConsume(QUEUE_NAME, true, consumer);

        // 获取消息
        while (true) {
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            String message = new String(delivery.getBody());
            System.out.println(" [x] Received '" + message + "'");
        }
    }

}

生产者：
public class Producer {
	
	private final static String QUEUE_NAME = "mq_test_01";

    public static void main(String[] argv) throws Exception {
        // 获取到连接以及mq通道
        Connection connection = MqConnectionUtil.getConnection();
        // 从连接中创建通道
        Channel channel = connection.createChannel();
        
        // 声明（创建）队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 消息内容
        String message = "Hello World!";
        channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");
        //关闭通道和连接
        channel.close();
        connection.close();
    }
}

Work Queues工作队列模型
一个生产发送消息到队列，允许有多个消费者接收消息，但是一条消息只会被一个消费者获取。
生产者：
public class Producers {
	private final static String QUEUE_NAME = "mq_test_01";

    public static void main(String[] argv) throws Exception {
        // 获取到连接以及mq通道
        Connection connection = MqConnectionUtil.getConnection();
        // 从连接中创建通道
        Channel channel = connection.createChannel();
        
        // 声明（创建）队列  
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 消息内容
        for (int i = 1; i <= 20; i++) {
            // 消息内容
            String message = "task .. " + i;
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            System.out.println("生产者发送消息：" + message);
            Thread.sleep(500);
        }
        //关闭通道和连接
        channel.close();
        connection.close();
    }
}

消费者1：
public class Consumers1 {
	private final static String QUEUE_NAME = "mq_test_01";

    public static void consume() throws Exception {

        // 获取到连接以及mq通道
        Connection connection = MqConnectionUtil.getConnection();
        // 从连接中创建通道
        Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 定义队列的消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
        	//消费队列事件监听
        	@Override
        	public  void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                    byte[] body) throws IOException {
        		// body 即消息体
        		String msg = new String(body);
                System.out.println("消费者1接收到消息：" + msg);
                
                try {
					Thread.sleep(50);
				} catch (Exception e) {
				}
                
        	}
        	
        };
        channel.basicConsume(QUEUE_NAME, true,consumer);
    }
}
消费者2：

public class Consumers2 {
	private final static String QUEUE_NAME = "mq_test_01";

    public  static void consume() throws Exception {

        // 获取到连接以及mq通道
        Connection connection = MqConnectionUtil.getConnection();
        // 从连接中创建通道
        Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 定义队列的消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
        	//消费队列事件监听
        	@Override
        	public  void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                    byte[] body) throws IOException {
        		// body 即消息体
        		String msg = new String(body);
                System.out.println("消费者2接收到消息：" + msg);
                
                try {
					Thread.sleep(50);
				} catch (Exception e) {
				}
                
//                channel.basicAck(envelope.getDeliveryTag(), false);
        	}
        	
        };
        channel.basicConsume(QUEUE_NAME, true,consumer);
    }
}
订阅模型： fanout模型中，生产者发布消息，所有消费者都可以获取所有消息
1、1个生产者，多个消费者
2、每一个消费者都有自己的一个队列
3、生产者没有将消息直接发送到队列，而是发送到了交换机
4、每个队列都要绑定到交换机
5、生产者发送的消息，经过交换机到达队列，实现一个消息被多个消费者获取的目的

X（exchange）交换机的类型有以下几种：
Fanout：广播，交换机将消息发送到所有与之绑定的队列中去

Direct：定向，交换机按照指定的Routing Key发送到匹配的队列中去

Topics：通配符，与Direct大致相同，不同在于Routing Key可以根据通配符进行匹配
订阅模型之Fanout
可以有多个消费者，每个消费者都有自己的队列
每个队列都要与exchange绑定
生产者发送消息到exchange
exchange将消息把消息发送到所有绑定的队列中去
消费者从各自的队列中获取消息
生产者：
public class Producers {
	private final static String EXCHANGE_NAME  = "fanout_exchange";

    public static void main(String[] argv) throws Exception {
        // 获取到连接以及mq通道
        Connection connection = MqConnectionUtil.getConnection();
        // 从连接中创建通道
        Channel channel = connection.createChannel();
        
        // 声明交换机 ，指定类型为fanout
        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");

        // 消息内容
        for (int i = 1; i <= 20; i++) {
            // 消息内容
            String message = "task .. " + i;
            channel.basicPublish(EXCHANGE_NAME, "",null, message.getBytes());
            System.out.println("生产者发送消息：" + message);
            Thread.sleep(500);
        }
        //关闭通道和连接
        channel.close();
        connection.close();
    }
}
消费者1：
public class ConsumersF1 {
	
	private static final String QUEUE_NAME = "fanout_queue_1";
	private static final String EXCHANGE_NAME = "fanout_exchange";

    public static void consume() throws Exception {

        // 获取到连接以及mq通道
        Connection connection = MqConnectionUtil.getConnection();
        // 从连接中创建通道
        Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        
        // 声明exchange，指定类型为fanout
        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
        
        //消费者将队列与交换机进行绑定
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "");
        
        // 定义队列的消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
        	//消费队列事件监听
        	@Override
        	public  void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                    byte[] body) throws IOException {
        		// body 即消息体
        		String msg = new String(body);
                System.out.println("消费者1接收到消息：" + msg);
                
                try {
					Thread.sleep(50);
				} catch (Exception e) {
				}
                
        	}
        	
        };
        channel.basicConsume(QUEUE_NAME, true,consumer);
    }
}
消费者2：
public class ConsumersF2 {
	
	private static final String QUEUE_NAME = "fanout_queue_2";
	private static final String EXCHANGE_NAME = "fanout_exchange";

    public static void consume() throws Exception {

        // 获取到连接以及mq通道
        Connection connection = MqConnectionUtil.getConnection();
        // 从连接中创建通道
        Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        
        // 声明exchange，指定类型为fanout
        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
        
        //消费者将队列与交换机进行绑定
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "");
        
        // 定义队列的消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
        	//消费队列事件监听
        	@Override
        	public  void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                    byte[] body) throws IOException {
        		// body 即消息体
        		String msg = new String(body);
                System.out.println("消费者2接收到消息：" + msg);
                
                try {
					Thread.sleep(50);
				} catch (Exception e) {
				}
                
        	}
        	
        };
        channel.basicConsume(QUEUE_NAME, true,consumer);
    }
}
订阅模型之Direct
消费者C1的队列与交换机绑定时设置的Routing Key是“error”， 而C2的队列与交换机绑定时设置的Routing Key包括三个：“info”，“error”，“warning”，假如生产者发送一条消息到交换机，并设置消息的Routing Key为“info”，那么交换机只会将消息发送给C2的队列。


生产者：
public class Producers {
	private final static String EXCHANGE_NAME  = "direct_exchange";

    public static void main(String[] argv) throws Exception {
        // 获取到连接以及mq通道
        Connection connection = MqConnectionUtil.getConnection();
        // 从连接中创建通道
        Channel channel = connection.createChannel();
        
        // 声明交换机 ，指定类型为direct
        channel.exchangeDeclare(EXCHANGE_NAME, "direct");
       
        // 消息内容
        for (int i = 1; i <= 20; i++) {
            // 消息内容
            String message = "task .. " + i;
            //生产者发送消息时，设置消息的Routing Key:"info"
            channel.basicPublish(EXCHANGE_NAME, "info",null, message.getBytes());
            System.out.println("生产者发送消息：" + message);
            Thread.sleep(500);
        }
        //关闭通道和连接
        channel.close();
        connection.close();
    }
}
消费者：
public class ConsumersD1 {
	
	private static final String QUEUE_NAME = "direct_queue_1";
	private static final String EXCHANGE_NAME = "direct_exchange";

    public static void consume() throws Exception {

        // 获取到连接以及mq通道
        Connection connection = MqConnectionUtil.getConnection();
        // 从连接中创建通道
        Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        
        // 声明exchange，指定类型为direct
        channel.exchangeDeclare(EXCHANGE_NAME, "direct");
        
        //消费者将队列与交换机进行绑定
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "info");
        
        // 定义队列的消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
        	//消费队列事件监听
        	@Override
        	public  void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                    byte[] body) throws IOException {
        		// body 即消息体
        		String msg = new String(body);
                System.out.println("消费者1接收到消息：" + msg);
                
                try {
					Thread.sleep(50);
				} catch (Exception e) {
				}
                
        	}
        	
        };
        channel.basicConsume(QUEUE_NAME, true,consumer);
    }
}
public class ConsumersD2 {
	
	private static final String QUEUE_NAME = "direct_queue_2";
	private static final String EXCHANGE_NAME = "direct_exchange";

    public static void consume() throws Exception {

        // 获取到连接以及mq通道
        Connection connection = MqConnectionUtil.getConnection();
        // 从连接中创建通道
        Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        
        // 声明exchange，指定类型为direct
        channel.exchangeDeclare(EXCHANGE_NAME, "direct");
        
        //消费者将队列与交换机进行绑定
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "error");
        
        // 定义队列的消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
        	//消费队列事件监听
        	@Override
        	public  void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                    byte[] body) throws IOException {
        		// body 即消息体
        		String msg = new String(body);
                System.out.println("消费者2接收到消息：" + msg);
                
                try {
					Thread.sleep(50);
				} catch (Exception e) {
				}
                
        	}
        	
        };
        channel.basicConsume(QUEUE_NAME, true,consumer);
    }
}
发布订阅之Topics
Topic类型的Exchange与Direct相比，都是可以根据RoutingKey把消息路由到不同的队列。只不过Topic类型Exchange可以让队列在绑定Routing key 的时候使用通配符
Routingkey 一般都是有一个或多个单词组成，多个单词之间以”.”分割，例如： item.insert
通配符规则：
     #：匹配一个或多个词

     *：匹配不多不少恰好1个词

举例：
     audit.#：能够匹配audit.irs.corporate 或者 audit.irs

     audit.*：只能匹配audit.irs

Topics生产者代码与Direct大致相同，只不过子声明交换机时，将类型设为topic，
消费者代码也与Direct大致相同，也是在声明交换机时设置类型为topic




