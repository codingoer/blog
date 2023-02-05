---
title: 日常开发及提测的注意事项
date: "2021/5/21 15:46:25"
tags: [总结]
---

## 前言

到新公司两个月，做了一些代码开发，对需求开发及代码质量有一些感想。那么对于写业务代码的我们来说，如何在平时的CRUD中尽量做到完美？

每个人对于自己写代码的习惯都不一样，都有自己的思维方式。下面我把我自己在日常开发中的一些习惯分享出来，仅供参考！! 

## 常见代码错误示例

### 编程基础

1. 数值这部分总会出错，BigDecimal转换常常会报空指针，还有就是精度问题。
2. 基础数据类型与包装类，定义好，不要转来转去的
3. 基本数据类型的范围
4. 写while注意死循环
5. switch注意break!
6. 递归函数注意栈溢出
7. Java中的语法糖，for-each，try-with-resources
8. 还有啥......

### 集合处理

1. Collection.singletonList()得到的list不能add
2. ArrayList.addAll入参不能为空
3. lambda表达式的使用注意空指针

### 上下文

在平时做需求中，总会改动之前原有方法，这时候就要提高警惕了。已不小心就改动了原有业务逻辑，对原有业务造成影响，那就得吃个High级别的bug。

所以要避免这类问题的产生，就一定要关注该方法的所有引用。

随便举个例子：比如MNT中报备，合同，修改单接口中，都会将BOS的DTO转换为APP使用的DTO，那么在改动这部分代码时就要关注所有用到的地方。

选中方法右键选择Find Usages或者快捷键Option+F7，会列出所有引用的方法。

![20230205015403](https:image.codingoer.top/blog/20230205015403.jpg)

在改动convertReportDetail时，就要先阅读所有引用此方法的代码逻辑，然后再做修改，这样就不会出现改动A影响B的问题。

<!-- more --> 

### 空指针异常

最常见的错误，没有之一！！

要想杜绝空指针异常就要在写代码中时刻想着，在对象 . 的时候就要考虑这个对象会不会为空。

1. 调用别的方法得到结果，使用该结果

```java
// 省略一万行代码
PersonalCenterInfoRespDTO personalCenterInfoRespDTO = masServiceProxy.queryPersonCenter(requestDTO);
// 省略一万行代码
Integer authStatus = personalCenterInfoRespDTO.getAuthStatus();
```

对于调用别的方法，一定点进去看这个方法的实现，有没有返回为空的情况。

2. 方法入参

```java
@Override
public Boolean accountBindEmail(AccountBindEmailRequestDTO accountBindEmailRequestDTO) {
    verifyCodeService.verifySmsVerifyCodeV2(accountBindEmailRequestDTO.getVerifyCodeType(), accountBindEmailRequestDTO.getLoginPhone());
    return masServiceProxy.accountBindEmail(accountBindEmailRequestDTO);
}
```

对于方法入参，确定入参来源，考虑所有来源有没有为空的情况，如果没有可以不用判断为空情况。

## 关于自测

每次开发完成都会自测自己写的代码，那么问题来了，自测我们都测什么？仅仅是业务吗？或者仅仅是代码吗？

- 所有的逻辑分支都要覆盖
- 所有的逻辑组合都要覆盖
- 正常系、异常系都要覆盖
- 各种错误码的场景都要覆盖
- 入参边界值都要覆盖

### 单元测试

单元测试不仅可以减少BUG的产生，还可以提高开发效率，促使我们写出高内聚低耦合的代码，我们每个人都应该重视。

如何做单测？什么样的代码可以做单测？下面我总结几点：

1. 工具类中方法，也就是静态方法
2. 外部系统接口
3. servcie方法

栗子

- Utils中的方法或者公用的静态方法

新加的方法一定写个main方法跑一下，多模拟几组入参。对于之前存在的方法也要跑下，如果你放心别的写的代码，那就随意。

```java
public static Integer matchThreshold(LocalDate srcDate, LocalDate targetDate) {
    int days = (int) (srcDate.toEpochDay() - targetDate.toEpochDay());
    if (days < 0) {
        days = Math.abs(days);
    }
 
    List<LayingDeviceDTO> dtoList = sortLabelConfig();
    if (CollectionUtils.isEmpty(dtoList)) {
        return 0;
    }
    for (LayingDeviceDTO layingDeviceDTO : dtoList) {
        if (days > layingDeviceDTO.getThreshold()) {
            return layingDeviceDTO.getThreshold();
        }
    }
 
    return 0;
}
 
 
public static void main(String[] args) {
    LocalDate srcDate = LocalDate.of(2021, 4, 30);
    LocalDate targetDate = LocalDate.of(2021, 5, 2);
 
    System.out.println(matchThreshold(srcDate, targetDate));
}
```

- Feign接口

```java
@RunWith(SpringRunner.class)
@SpringBootTest
@Slf4j
public class BesClientTest {
 
    @Resource
    private BesClient besClient;
 
    @MockBean
    private EurekaAutoServiceRegistration eurekaAutoServiceRegistration;
 
    @Test
    public void getContractAmountInfo() {
        RectifyDetailReqDTO reqDTO = new RectifyDetailReqDTO();
        reqDTO.setRectifyId(70L);
 
        BaseResponse<RectifyEsDTO> rectifyDetail = besClient.getRectifyDetail(reqDTO);
        System.out.println(rectifyDetail.getModel());
 
        log.info("===============");
    }
}
```

- 业务方法接口

先通过抓包找入参再编写测试代码

```java
@SpringBootTest
@RunWith(SpringRunner.class)
@Slf4j
public class ContractQueryTest {
 
    @Resource
    private ContractV2Business contractV2Business;
 
    @Test
    public void testContractDetail() {
        ContractDetailRequestDTO requestDTO = new ContractDetailRequestDTO();
        requestDTO.setContractId(1308477L);
        requestDTO.setCode("GSZN22805");
        requestDTO.setCodeType(1);
        requestDTO.setIsOriContract(false);
 
        BaseResponse<ContractQueryDetailResponseDTO> contractDetail = contractV2Business.getContractDetail(requestDTO);
        System.out.println(JSON.toJSONString(contractDetail.getModel(), SerializerFeature.PrettyFormat));
    }
}
```

### 用例评审

用例评审会议要认真听，尤其是涉及自己那部分功能，这样才知道测试要测那块。后面针对评审的文档，进行自测。

还有一点很重要，需求改动涉及到此次需求外的代码了，比如老的业务逻辑，要通知测试回归测试，不要以为没有问题就是没有问题了。总之，对于你改动的代码一定要全面覆盖。

### 交叉测试

APP，底层服务的人都会测，这样会测试出自己测试不到的问题。如果是单一的后台功能，自测就要费点心了。

## 需求文档

其实大部分BUG产生的源头都来自这里，有以下几点：

1. 产品文档模糊，有些描述不是很清楚，如果我们不细问就会吃Bug
2. 产品文档变动频繁，后续又补充一些东西，产品口喷没有更新文档
3. 阅读文档不仔细，遗漏

对于上述几点内容，我觉得我们自己的原因要避免，如阅读文档不仔细，一目三行这种。一定要好好阅读需求文档，跟自己相关的内容一定看仔细。对于文档中有模糊的地方要问清楚。

其次，产品后续变更的内容要落实到文档，关注开发群内容，变更内容及时知晓。

综上所述：

1. 最重要的就是仔细阅读需求文档，对于不清楚的地方及时确认问清，要不就吃BUG了
2. 文档有变动要落实文档，或有聊天记录，日后有证据
3. 关注群聊天记录，这个很容易遗漏东西，导致BUG产生

## 代码之外的因素

在需求开发中都会遇到产品文档频繁变更，上线前遇到改代码是经常发生，遇到这种不确定因素，我们应该怎么处理？

1. 临上线改代码要慎重，自行评估，能改就改，改动很大就不要动
2. 这种一般就是产品的问题，咬死不改没毛病！

## CodeReview

CodeReview很重要！CodeReview很重要！CodeReview很重要！

就目前来说，针对大部分需求开发我们没有进行code review。但是这不重要！重要的是自己写的代码要自己负责！

我自己会做这几件事：

- 每次commit时，对比文件，看看自己改了什么，提交之前再检查一遍。
- 开发一段时间后，进行代码比对，看看自己改了什么。
- 在Gitlab中进行分支合并请求，查看改动的代码，提测之前一定要对比，防止遗漏的地方。

### Commit之前对比

每次Commit之前对比文件，也就是提交之前进行检查，这样可以对将要提交上来的代码进行最后检查。

这样做的好处是:

1. bug检查，有可能之前没有发现的bug，看了一眼后就突然发现了呢......
2. 代码规范！！！注释没加的，无用的包引入的，
3. 一些测试代码忘记改回去的，不小心就提交上去了，那就惨了
4. 字段名字拼写错的都可以检查出来，也许吧...

步骤：

1. 选择Commit视图
2. 选中某个文件，按Command+D，会出现改动的地方

![20230205201952](https:image.codingoer.top/blog/20230205201952.jpg)

### Idea中代码对比

代码开发到中期或者后期，可以进行代码比对，看看自己都改了什么。在对比文件的过程中可以加深思考，比如我改了这个方法会不会对原有业务有影响呢？?

好处：
1. 对代码进行二次BUG检查。
2. 对改动的地方，涉及上下文的地方再加深思考

步骤：
1. 切换到自己的分支
2. 鼠标移动到com.enmonster包下，右键选择Git里面的比对分支
3. 选择比对origin/master分支后，会出现你改动的所有文件列表
4. 选择某一个文件，按Command+D，会出现改动的地方

![20230205202101](https:image.codingoer.top/blog/20230205202101.jpg)

![20230205202114](https:image.codingoer.top/blog/20230205202114.jpg)

![20230205202124](https:image.codingoer.top/blog/20230205202124.jpg)

### GitLab中进行合并请求

在提测之前，在GitLab中进行分支合并请求，可以对比出改动的文件。最后一次进行CodeReview。

![20230205202146](https:image.codingoer.top/blog/20230205202146.jpg)

![20230205202159](https:image.codingoer.top/blog/20230205202159.jpg)

## 提测、上线前注意

一些配置项，数据的处理