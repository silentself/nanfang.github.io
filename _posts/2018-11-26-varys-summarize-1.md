---
layout: post
title:  varys项目总结<1>
excerpt: "整个项目开发过程很糟心，由于产品概念设计不严谨，表结构也是改了又改，代码逻辑也是翻来覆去的重写,不过还是有一些心得的，在这里简单总结一下避免再次出现同样的经历"
categories: [java]
tags: [project]
comments: true
---

#### 1、事务处

1、不可以在业务层catch异常，如果非抓不可，抛个运行时异常

```java
try {
    n = userDao.insertMcnPrefix(insertObj);
} catch (Exception e) {
    logger.error("saveMcninfoPlats is failure cased by ---->{}", ResultMessage.SAVE_EXCEPTION);
    throw new RuntimeException();
}
```

当然，自定义异常也是可以的

2、封装返回结果在控制层进行

```java
/**
 * 根据uuid获取组织信息
 * 
 * @param uuid
 * @return
 */
@PostMapping("/getOrginfoByuuid")
public VarysResult<Organization> getOrginfoByuuid(Integer uuid) {
    if (null == uuid) {
        logger.error("getOrginfoByuuid is failure cased by ---->{}", ResultMessage.PARAMS_EMPTY);
        return ResultUtils.doFailure(ResultMessage.PARAMS_EMPTY);
    }
    Organization org = null;
    try {
        org = userServiceImp.getOrginfoByuuid(uuid);
    } catch (Exception e) {
        logger.error("getOrginfoByuuid is failure cased by ---->{}", e.getMessage());
        return ResultUtils.doFailure(e.getMessage());
    }
    logger.info("getOrginfoByuuid is successed ---->{}", null == org ? "没有获取组织信息" : org.getAuthorizeOwner());
    return ResultUtils.doSuccess(org);
	}
```

3、java每运行到一个java类上就会创建一个jvm，开启一个线程，最好每层都做一下基本验证，如果加上事务，在还没有调用持久层方法时，可以直接return，如果前面已经调用dao的方法，就需要Throw new RuntimeException（）；

```java
@Override
@Transactional
public Boolean saveMcninfoPlats(MCNAccountDetailForm mcninfo) {
    if (null == mcninfo || null == mcninfo.getThirdPartyPlatFormList()
        || mcninfo.getThirdPartyPlatFormList().size() < 1 || null == mcninfo.getPrefixId()) {
        logger.error("saveMcninfoPlats is failure cased by ---->{}", ResultMessage.PARAMS_EMPTY);
        return false;
    }
    // 根据PrefixId生成获取mcn前缀
    String prefix = userDao.getPrefix(mcninfo.getPrefixId());
    if (null == prefix) {
        logger.error("saveMcninfoPlats is failure cased by ---->{}", ResultMessage.MCNID_GENERATE_ERROR);
        throw new RuntimeException();
    }
    // 生成mcnid序列
    Integer mcnSequence = null;
    String tableName = SequenceTable.getTableName(mcninfo.getPrefixId());
    MCNSequence insertObj = new MCNSequence(mcnSequence, prefix, tableName);
    int n = userDao.insertMcnPrefix(insertObj);
    if (n != 1) {
        logger.error("saveMcninfoPlats is failure cased by ---->{}", ResultMessage.SAVE_EXCEPTION);
        throw new RuntimeException();
    }
    if (null == insertObj.getMcnSequence()) {
        logger.error("saveMcninfoPlats is failure cased by ---->{}", ResultMessage.MCNID_GENERATE_ERROR);
        throw new RuntimeException();
    }
    // 生成mcnid，并设置给mcninfo
    String MCNID = String.valueOf(prefix + insertObj.getMcnSequence());
    mcninfo.setAccountID(MCNID);
    int count = 0;
    // 新建mcn信息，并返回mcninfo表的主键
    count += userDao.insertMcninfo(mcninfo);
    if (null == mcninfo.getId() || count < 0) {
        logger.error("saveMcninfoPlats is failure cased by ---->{}", ResultMessage.PK_RETURN_ERROR);
        logger.info(ResultMessage.TRANSACTION_ROLLBACK);
        throw new RuntimeException();
    }
    List<ThirdPartyPlatForm> platFormList = mcninfo.getThirdPartyPlatFormList();
    for (ThirdPartyPlatForm platform : platFormList) {
        if (null == platform) {
            break;
        }
        // 给平台信息循环设置mcninfo表主键，并入库平台信息
        platform.setMcninfoid(mcninfo.getId());
        count += userDao.insertPlatformInfo(platform);
    }
    if (count < 2) {
        logger.error("saveMcninfoPlats is failure cased by ---->{}", ResultMessage.SAVE_EXCEPTION);
        throw new RuntimeException();
    }
    logger.info("saveMcninfoPlats is success 工具写操作 ---->{}条", count);
    return true;
}
```

4、事务是通过数据源进行控制的，mysql的innodb存储引擎，默认是行级锁，当写操作涉及到主键的时候，会自动升级成表级锁

#### 2、mybatis主键生成策略

1、在持久层使用注解@Options/或者在mapper.xml文件设置主键生成策略之后，是在调用当前属性的get方法时才会执行，如果不调用get方法时返回不回来的，所以如果需要主键返回就需要创建个java对象，使用map、和传多个参数都不可行

```java
@Test
public void test2() {
    User user = new User().setUsername("刘德华");
    Integer id = user.getId();
    testDao.insertInfo2(user);
    System.out.println("调用dao之前get获取的id为--->" + id);
    Integer id2 = user.getId();
    System.out.println("调用dao之后get获取的id为--->" + id2);
}
```

打印结果如下

```console
调用dao之前get获取的id为--->null
调用dao之后get获取的id为--->7
```

#### 3、建表的时候命名真的很重要













