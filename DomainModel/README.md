# 领域模型

***“领域模型”就是“解决方案空间”***
是针对特定领域里的关键事物及其关系的可视化表现，是为了准确定义需要解决问题而构造的抽象模型，
是业务功能场景在软件系统里的映射转化，其目标是为软件系统构建统一的认知。

例如，
请假系统解决的是人力工时的问题，属于人力资源领域，对口的是HR部门；
费用报销系统解决的是员工和公司之间的财务问题，属于财务领域，对口的是财务部门；
电商平台解决的是网上购物问题，属于电商领域。
可以看出，每个软件系统本质上都解决了特定的问题，属于某一个特定领域，实现了同样的核心业务功能来解决该领域中核心的业务需求。

领域模型是能够精确反映领域中某一知识元素的载体，需要通过与领域专家(Domain Expert)进行频繁的沟通才能将专业知识转化为领域模型。
“领域驱动设计”是为业务专家和编程人员建立了一座沟通的桥梁，一种有效的操作手段和方法，其核心还是领域建模。

## 作用
帮助分析理解复杂业务领域问题，描述业务中涉及的实体及其相互之间的关系，是需求分析的产物，与问题域相关。
是需求分析人员与用户交流的有力工具，是彼此交流的语言。
分析如何满足系统功能性需求，指导项目后续的系统设计。

## 失血模型、贫血模型、充血模型、胀血模型 - Martin Fowler
```md
“血”指的是领域对象的model层内容。
```
### 失血模型
```md
基于数据库的领域设计方式其实就是典型的失血模型。

以Java为例，POJO 只有简单的基于field的setter，getter方法，POJO之间的关系隐藏在对象的某些ID里，
由外面的manager解释，比如son.fatherId，Son并不知道他跟Father有关系，
但 manager 会通过 son.fatherId 得到一个Father。
```
```md
失血模型中，domain object 只有属性的get set方法的纯数据类，所有的业务逻辑完全由Service层来完成的，
由于没有dao，Service直接操作数据库，进行数据持久化。
service:  肿胀的服务逻辑
model: 只包含get set方法
```
### 贫血模型

只有属性的类，贫血的意思就是没有行为，像木乃伊一样。
这种模型唯一的作用就是将一些 ORM 映射到对应的数据库上，
而我们的「服务层」通过「DAO层」加载这些「贫血模型」进行一些拼接之类的操作，
功能越复杂，这种操作就越频繁，这是我们的软件复杂度上升的直接原因。

```md
贫血模型中，domain object包含了不依赖于持久化的原子领域逻辑，而组合逻辑在Service层。
service ：组合服务，也叫事务服务
model：除包含get set方法，还包含原子服务
dao：数据持久化
```
```md
儿子不知道自己的父亲是谁是不对的，不能每次都通过中间机构（Manager）验DNA(son.fatherId)来找爸爸，领域模型可以更丰富一点。
```
```java
public class Son{
	private Father father;
	public Father getFather(){return this.father;}
}
public class Father{
	private Son son;
	private Son getSon(){return this.son;}
}
```
```md
通常一个object是通过一个repository（数据库查询），或者factory（内存新建）得到的
为了构建完整的son对象，sonRepo里需要一个fatherRepo来构建一个father去赋值son.father。
而fatherRepo在构建一个完整father的时候又需要sonRepo去构建一个son来赋值father.son。
这形成了一个无向有环圈，这个循环调用问题是可以解决的，但为了解决这个问题，领域模型会变得有些恶心和将就。
有向无环才是我们的设计目标，为了防止这个循环调用。
我们是否可以在father和son这两个类里省略掉一个引用？
```
```java
public class Father{
	//private Son son; 删除这个引用
	private SonRepository sonRepo;//添加一个Son的repo
	private getSon(){return sonRepo.getByFatherId(this.id);}
}
```
```md
这样在构造Father的时候就不会再构造一个Son了，但代价是我们在Father这个类里引入了一个SonRepository, 
也就是我们在一个domain对象里引用了一个持久化操作，这就是我们说的充血模型。 
```
### 充血模型
```md
充血模型中，绝大多业务逻辑都应该被放在domain object里面，包括持久化逻辑，
而Service层是很薄的一层，仅仅封装事务和少量逻辑，不和DAO层打交道。

service ：组合服务 也叫事务服务
model：除包含get set方法，还包含原子服务和数据持久化的逻辑
```
```md
充血模型和第二种模型差不多，所不同的就是如何划分业务逻辑，
即认为，绝大多业务逻辑都应该被放在domain object里面(包括持久化逻辑)

而Service层应该是很薄的一层，仅仅封装事务和少量逻辑，不和DAO层打交道。 
Service(事务封装) ---> domain object <---> DAO 
这种模型就是把第二种模型的 domain object和 business object合二为一了。

所以ItemManager就不需要了，在这种模型下面，只有三个类:

Item：包含了实体类信息，也包含了所有的业务逻辑 
ItemDao：持久化DAO接口类 
ItemDaoHibernateImpl：DAO接口的实现类 

在这种模型中，所有的业务逻辑全部都在Item中，事务管理也在Item中实现。
```
* 优点
```md
更加符合OO的原则 
Service层很薄，只充当Facade的角色，不和DAO打交道。 
```
* 缺点： 
```md
DAO和domain object形成了双向依赖，复杂的双向依赖会导致很多潜在的问题。 
如何划分Service层逻辑和domain层逻辑是非常含混的，在实际项目中，由于设计和开发人员的水平差异，可能导致整个结构的混乱无序。 
考虑到Service层的事务封装特性，Service层必须对所有的domain object的逻辑提供相应的事务封装方法，其结果就是Service完全重定义
充血模型的存在让domain object失去了血统的纯正性，他不再是一个纯的内存对象，这个对象里埋藏了一个对数据库的操作，这对测试是不友好的。
```
### 胀血模型 
```md
胀血模型取消了Service层，只剩下domain object和DAO两层，在domain object的 domain logic上面封装事务。
```
```md
基于充血模型的第三个缺点，有同学提出，干脆取消Service层，只剩下domain object和DAO两层，
在domain object的domain logic上面封装事务。 
domain object(事务封装，业务逻辑) <---> DAO 
似乎ruby on rails就是这种模型，他甚至把domain object和DAO都合并了。 
```
```md
该模型优点： 
1、简化了分层 
2、也算符合OO 
```
```md
1、很多不是domain logic的service逻辑也被强行放入domain object ，引起了domain object模型的不稳定 
2、domain object暴露给web层过多的信息，可能引起意想不到的副作用。
```
### 总结
```md
在这四种模型当中，失血模型和胀血模型应该是不被提倡的。

而贫血模型和充血模型从技术上来说，都已经是可行的了。
贫血模型和充血模型的差别在于，领域模型是否要依赖持久层，贫血模型是不依赖的，而充血模型是依赖的。
```
* 参考
[领域模型之失血、贫血、充血、胀血模型](https://jlife.iteye.com/blog/625018)

## 领域模型：测试友好
```md
失血模型和贫血模型是天然测试友好的（其实失血模型也没啥好测试的）,因为他们都是纯内存对象。

但实际应用中充血模型是存在的，要不就是把domain对象拆散，变得稍微不那么优雅（当然可以，贫血和充血的战争从来就没有断过）。
那么在充血模型下，对象里带上了persistence 特性，这就对数据库有了依赖，mock/stub掉这些依赖是高效单元化测试的基本要求。
```
## 领域模型:repository的实现方式
```md
设计了Tunnel这个接口，通过这个接口我们可以实现对domain对象在不同类型数据库的存取。
Repository并没有直接进行持久化工作，而是将domain对象转换成POJO交给Tunnel去做持久化工作，Tunnel具体实现可以在任何包实现，
这样，部署上，domain领域模型（domain objects+repositories）和持久化(Tunnels)完全的分开，domain包成为了单纯的内存对象集。
```
