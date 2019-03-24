
1: 链表的反转  

	public ListNode ReverseList(ListNode head) {
	
        ListNode preHead = null;
        while(head!=null){
            ListNode next = head.next;
            head.next = preHead;
            preHead = head;
            head = next;
        }
        return preHead;
    }



2: 网页输入 www.taobao.com的整个过程，包括了nginx，和dns
 ## 在浏览器中输入网址后，会先去寻找对应的ip。所以会先看缓存中是否有相应的dns信息，如果没有，则看hosts文件，最后像dns服务器查询
 ## 然后向对应的服务器发送tcp请求，三次握手，得到响应后，四次挥手
 
3: Object的方法，哪些属于本地方法，为什么要有本地方法，int[] 数组是不是继承了Object的本地方法，hashcode什么用，有没有看过字节码，foreach的字节码是怎么实现的

4: 对象头里有什么
	
5: 锁的膨胀

6: java 8的一些语法以及原理，stream，lambda

7: mysql是否命中索引，索引优化

8: linux命令对文件的第二列排序

9: top,jmap,jstack

10: full gc, old gc区别

11: CMS和G1的差异

12: io各种 

13: mysql 索引

14: 乐观锁和悲观锁

15: Spring

```
1、说下你对 Spring 生态的了解？

2、说下你对 Spring AOP 和 IOC 的理解？看过实现原理吗？

3、说下 Bean 在 Spring 中的生命周期？

4、讲下你知道的 Spring 注解有哪些？该什么场景使用？

5、Spring 事务知道吗？有了解过吗？

6、说下你刚才说的 SpringBoot 吧，你觉得 SpringBoot 有什么优点？

7、SpringBoot 自动化配置是怎么做的？有看过实现源码吗？

8、Spring Boot 中最核心的注解 SpringBootApplication 有看过源码分析过吗？

9、你的项目中 SpringBoot 用到了哪些和其他技术栈整合的？

10、使用 Spring 或者 SpringBoot 有遇到过什么印象深刻的问题吗？当时是怎么解决的？
```
