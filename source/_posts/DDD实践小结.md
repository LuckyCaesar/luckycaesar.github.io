---
title: DDD实践小结
tags: [领域驱动, 微服务]
index_img: /img/code-wallpaper-2.jpeg
date: 2020-11-18 21:00:00
---



## 闲话

这几个月公司对系统进行重构，我们的[**架构师**](https://github.com/yangyang0507)决定尝试下目前比较流行的设计思想 - DDD，领域驱动设计。这种设计理念很早就出现了，但是迄今为止除了大厂外并没有比较普适性的实践落地方案，也是由于许多中小型系统还达不到相当程度的复杂度来让DDD充分发挥其优势，或者由于部分开发人员的基本素质还达不到相关的要求，导致很难推行。

但就我所了解或者参与的一些项目，其中也使用到了DDD中所推崇的一些设计，譬如说：

- 充血模型：简单来说，就是让实体对象也可以去处理一些逻辑，而不是简单的承载数据，在Service/Manager里面编写业务逻辑。

  ```java
  public class Customer {
  
      private Long id;
  
      private String code;
  
      private String name;
  
      ...
  
      public boolean isNew() {
          return this.getId() == null;
      }
  
      public void merge(Customer targetCustomer) {
          ClassUtil.combineObjects(this, targetCustomer);
      }
  
      public void calculateImportance(CustomerImportanceStrategyCondition condition) {
          this.setImportance(CustomerImportanceStrategyEnum.calculateImportance(condition));
      }
  
  
      public void checkDuplicateName(CustomerDuplicateCallable callBack) {
          ...
      }
  
      public void checkWhenTransforming(String customerName, String industryCode, CustomerSourceEnum source) {
          ...
      }
  
      public void transferPhones(List<CustomerPhoneReqDTO> phoneList) {
          ...
      }
  }
  ```

  这样做的好处是可以让职责划分更明确，依赖边界更清晰，Service层也不再杂糅所有的逻辑，造成后续改动越来越乱的情况。但要使用好充血模型，得明确划分好业务中的各个领域/角色/模块，需要在开发前做好充分的设计。

- 仓储模式：顾名思义，属于跟持久层打交道的一种设计模式。简单说就是采用**依赖倒置**的方式，在业务层抽象仓储接口，定义CRUD方法，然后持久层依赖业务层，实现具体的方法。这样做的好处就是业务逻辑可以不关心持久层的具体实现，持久层任意替换数据存储方式或者持久化框架而不影响业务逻辑。

  ```java
  // 注意这个包路径，业务领域层
  package cn.xx.domain.customer.repository;
  
  public interface CustomerRepository {
  
      Customer getById(Long id);
  
      ...
  
  }
  ```

  ```java
  // 持久层
  package cn.xx.persistence.repository.impl.customer;
  
  @Slf4j
  @Repository
  @RequiredArgsConstructor
  public class CustomerRepositoryImpl implements CustomerRepository {
  
      private final CustomerFactory customerFactory;
  
      private final CustomerMapper customerMapper;
  
      @Override
      public Customer getById(Long id) {
          return customerFactory.toCustomerDO(customerMapper.selectById(id));
      }
  
      ...
  }
  ```



## 实践

整体的实践过程的话，个人感觉就是理想很丰满，现实很骨感，也是由于我们对DDD的理解实践经验不够，其实大部分还是按照传统的一些方式在做。

### 战略设计

这一块的感觉最明显，基本上还是由产品去定义，其它角色主要是参与到业务讨论，或者架构讨论中去。

- 产品愿景和场景分析：不过多赘述，我们做的是2B的SaaS系统，当然也包含了内部使用的一些模块，都嵌入到整个平台，其实基本上定型了。也没有专门的领域专家、需求分析人员等角色来参与，都是CTO、产品、架构师等。

- 领域建模和微服务拆分：领域建模并不严格和完整，大致上都是按职责功能等传统方式来划分，也有部分按照领域的概念独立出来。最后拆分出来的主要微服务如下：

  - 营销系统：针对客户挖掘和CRM客户关系管理，内部使用

  - 客户端系统：对外提供给B端客户的APP或者Web客户端

  - 财务系统：内部供财务使用

  - 协议系统：统一管理与客户签订的合同协议

  - 配置中心：基础数据，如省市区、行业、字典等，基础功能，如导入等

  - 用户中心：员工数据和系统权限、资源配置

  - 授权中心：登录、认证等

  - 流程引擎：统一处理业务系统中所有的流程

  - 文件系统

  - 消息系统

  - BI服务

    ...

可以看出来整体的流程并不规范，但是很适合我们这样的小型团队，快速的定下产品基调，做好服务拆分，指派开发人员进行开发。

### 战术设计

主要是单个微服务的分层结构规范，这部分由我们的[**架构师**](https://github.com/yangyang0507)重新设计。初始的设计是这样的：

![](/img/image-20201117144112355.png)



从这张图也可以看出一些问题：

- VO、DTO、DO(Data Object or Domain Object)、PO的转换比较繁琐，最多有三层转换，极大的降低开发效率。
- 单独的Query层承担一些或复杂或简单的查询，但是下面还依赖了一层persistence，让简单的查询也变得麻烦，没有必要。

优化后的最终版：

![](/img/image-20201117162157114.png)



改进之后，可以看出更加合理，实践起来更接地气。

先定义下各种对象：

- VO：View Object，视图对象，主要是接口返回数据的封装，不同于系统内部的数据流转对象，接口返回的数据结构一定是经过各种转换封装的，易读易理解，不能简单的类似表字段。
- DO（Query）：Data Object，实际上在这一层我们是用的DTO的后缀。
- DTO：Data Transfer Object，数据传输对象，通用的后缀。
- DO（Domain）：Domain Object，领域对象，包含领域属性以及行为（方法逻辑）。
- PO：Persistent Object，持久化对象，映射数据库表的实体。

再做一下整体结构梳理：

- Bootstrap：顾名思义，启动类和整个项目的基础配置文件就在这一层。
- Interfaces：对外的接口层，包含Web和APP相关的接口和VO。
- RPC：由于我们使用的是Dubbo，所以在这一层里去依赖RPC API并实现。
- Query：query层，或简单或复杂的单纯的查询就走这一层，譬如分页等各种条件查询。
- Application：应用层，对多个领域服务的一个编排、封装，形成完整的业务逻辑，对外提供粗粒度的服务。这一层中还有对事件（消息）的处理，我们消息的消费都是在这一层进行，具体实现可以向下调用领域服务层。
- Domain：领域层，包含了某个领域的逻辑，使用充血模型在DO（Domain Object）中实现各个领域的逻辑，由DomainService统一封装向上提供接口。
- Persistence：持久层，仓储模式，依赖倒置。
- Infrastructure：基础设施层，Config（自定义配置）、Utils、Constant等。

实际上在落地的过程中，还有一些改动，像PO就单独提出来一层，供Query和Persistence（使用了依赖倒置，实际上是Domain层）去依赖，这样Query层就不用再去创建PO，只需要创建Mapper即可。

## 小结

总的来说，这次实践的收获还是蛮大的，对DDD的设计思想有了一个实际的落地方案，并在此过程中不断改进优化，使之更适合我们的团队。这让我们对它的理解更加深刻，在探索更合适的产品开发流程和微服务结构的道路上实实在在的迈出了一步。

细化一点，个人感觉在实际的代码设计和编写过程中，遵循**领域对象和充血模型**的方式去做，可能对于大多数业务场景来说是更好的选择，实际上这两者也是面向对象语言的抽象和封装的体现。