---
layout: post
title:  mybatis-注解版<1>
excerpt: "mybatis-注解版sql一直以来被应用的很少，主要是由于sql修改之后还需要编译之后才能运行......"
categories: [java]
tags: [mybatis]
comments: true
---
mybatis-注解版sql一直以来被应用的很少，主要是由于sql修改之后还需要编译之后才能运行，下面根据本人在工作中用到的情况分需求来一一演示，如有不对之处还望指正，请轻喷。
<!---more--->

#### 1、四个本命注解@Select、@Update、@Delete、@Insert

CRUD使用的就是上面的注解，这个都较为普通，在这里简单描述一下,需要注意的时：**四个注解都是支持String数组的**

所以可以这样写

```java
@Insert({ "insert into tbl_varys_r_account(ac_parent_account_id,ac_accont_id) values(#{pid},#{sid})" })
	int insertAccountLinkedInfo(@Param("pid") Integer parentId, @Param("sid") Integer subId);
```

当然也可以这样写

```java
@Insert("insert into tbl_varys_r_account(ac_parent_account_id,ac_accont_id) values(#{pid},#{sid})" )
	int insertAccountLinkedInfo(@Param("pid") Integer parentId, @Param("sid") Integer subId);
```

1）@Select

```java
@Select("select * from tbl_varys_m_user t where t.us_username = #{username} and t.us_pwd = #{password}")
@Results(id = "user_resultMap",value = {
    @Result(property = "id",  column = "us_uuid",id=true),
    @Result(property = "orgid",  column = "us_orgid_fk"),
    @Result(property = "username",  column = "us_username"),
    @Result(property = "password",  column = "us_pwd"),
    @Result(property = "nickname",  column = "us_nickname"),
    @Result(property = "creatorid",  column = "us_usid_creator"),
    @Result(property = "telNum",  column = "us_tel_num"),
    @Result(property = "status",  column = "us_status"),
    @Result(property = "type",  column = "us_type"),
    @Result(property = "email",  column = "us_email"),
    @Result(property = "isSub",  column = "us_issub"),
    @Result(property = "createTime",  column = "us_create_time"),
    @Result(property = "modifyTime",  column = "us_modify_time")
})
User selectUserByPwdAndUsername(@Param("password") String password, @Param("username") String username);
```

2)@Update

```java
@Update({ "update tbl_varys_m_user set us_pwd = #{newPwd} where us_uuid = #{uuid} and us_pwd = #{oldPwd}" })
int modifyPassword(@Param("uuid") Integer uuid, @Param("oldPwd") String oldPassword,@Param("newPwd") String newPassword);
```

3)@Insert

```java
@Insert({ "insert into tbl_varys_r_account(ac_parent_account_id,ac_accont_id) values(#{pid},#{sid})" })
int insertAccountLinkedInfo(@Param("pid") Integer parentId, @Param("sid") Integer subId);
```

4)@Delete

由于当前项目我负责部分还为涉及delete操作，就暂时不展示了，不过用户和上面的都一样

#### 2、@Results注解复用resultMap

之前在写xml文件时是可以复用ResultMap的，注解版依旧可以复用，就像上面Select展示的代码

```java
@Results(id = "user_resultMap",value = {
    @Result(property = "id",  column = "us_uuid",id=true),
    ....
```

只需要在@Results注解中设置属性id的值，再其他方法放可以通过注解@ResultMap注解复用

```java
@Select("select * from tbl_varys_m_user t where t.us_username = #{username} and t.us_email = #{email}")
@ResultMap("user_resultMap")
User selectUserByAccountAndEmail(@Param("username") String username, @Param("email") String email);
```

#### 3、如何设置主键返回

此处主页分成两种情况，返回自增主键和非自增主键

##### 1）返回自增主键

```java
@Insert({ "insert into tbl_varys_m_user(us_orgid_fk,us_username,", "us_pwd,us_nickname,us_usid_creator,","us_tel_num,us_email,us_status,", "us_type,us_issub,us_create_time) ", "values(#{orgid},#{username},",
"#{password},#{nickname},#{creatorID},", "#{telNum},#{email},#{status},", "'1',#{isSub},#{createTime})" })
@Options(useGeneratedKeys = true, keyProperty = "userid")
int insertUserinfo(CustomerRegisterForm registerForm);
```

增加注解@Options并设置属性useGeneratedKeys为true，同时指定主键返回到当前对象的那个字段keyProperty = "userid"

##### 2）返回非自增主键

返回非自增主键需要用到注解@SelectKey，由于当前项目未涉及到非自曾主键的返回，在此就引用《Mybatis从入门到精通》一书中的内容

![返回非自增主键](/img/posts/2018-11-15-mybatis-annotation-sql-1.png)

#### 4、如何验证参数非空

##### 1）使用标签"\<script>"

自此先做个补充，使用此标签之后类同xml文件中的动态sql，可以跟在xml中的动态sql写法一样，同样也可以用foreach、when、if等标签

```java
@Select({ "<script>", "SELECT c.td_mcn_name AS clientName, a.us_tel_num AS phoneNum,a.us_email AS email, ",
			"b.og_wechatnum AS wechatNum, b.og_publicnum_type AS publicNumType, d.nt_name AS industry, ",
			"CONCAT(e.ap_name, f.ac_name) AS provinceAndCity,g.mt_name AS mediaType, a.us_status AS 'status' ",
			"FROM tbl_varys_m_user a, tbl_varys_m_organization b, tbl_varys_m_thirdplatinfo c, ",
			"tbl_varys_d_industry d, tbl_varys_d_area_province e,tbl_varys_d_area_city f, tbl_varys_d_media_type g ",
			"WHERE 	a.us_orgid_fk = b.og_id AND a.us_uuid = c.td_usid_fk AND b.og_industry_fk = d.nt_id ",
			"AND b.og_arid_province = e.ap_id AND b.og_arid_city = f.ac_id AND c.td_mdid_fk = g.mt_id ",
			"<when test = 'provinceID != null'> AND b.og_arid_province = #{provinceID} </when> ",
			"<when test = 'cityID != null'> AND b.og_arid_city = #{cityID} </when> ",
			"<when test = 'countyID != null'> AND b.og_arid_county = #{countyID} </when> ",
			"<when test = 'status != null '> AND a.us_status = #{status} </when> ",
			"<when test = 'publicNumType != null'> AND b.og_publicnum_type = #{publicNumType} </when> ",
			"<when test = 'industryID != null'> AND b.og_industry_fk = #{industryID} </when> ",
			"<when test = 'mediaTypeID != null'> AND c.td_mdid_fk  </when> ",
			"<when test = 'startIndex != null and pageSize != null'> LIMIT #{startIndex},#{pageSize} </when> ",
			"</script>" })
List<ClientSimpleForm> selectUserInfoListByPage(SearchClientListForm condition);
```

另外，**上面的sql中没有用到ResulMap做映射，使用关键字AS改了结果集中的别名，这两种方式都可以**

##### 2）使用Provider注解

CRUD的四个基本注解都有个Provider形式的注解（@SelectProvider...）

①、写个方法返回sql，在该方法中进行非空验证

```java
public static String queryConditionNotEmpty(final SearchMCNListForm condition) {
		SQL sqlStr = new SQL()
				.SELECT("b.td_mcn_num,c.ap_name,a.og_publicnum,a.og_wechatnum,f.us_username,"
						+ "d.mt_name,a.og_expire_time,e.tp_id,e.tp_name,b.td_tel_num,b.td_loginstate,e.tp_logo_url")
				.FROM("tbl_varys_m_thirdplatinfo b,tbl_varys_m_organization a,tbl_varys_d_area_province c"
						+ ",tbl_varys_d_media_type d,tbl_varys_d_third_plat e,tbl_varys_m_user f");
		sqlStr.WHERE("b.td_orgid_fk = a.og_id AND b.td_mdid_fk = d.mt_id AND a.og_arid_province = c.ap_id "
				+ "AND b.td_platid_fk = e.tp_id AND b.td_usid_fk = f.us_uuid");
		if (null != condition.getCityID()) {
			sqlStr.WHERE("a.og_arid_city = #{countyID}");
		}
		if (null != condition.getCountyID()) {
			sqlStr.WHERE("a.og_arid_city = #{cityID}");
		}
		if (null != condition.getProvinceID()) {
			sqlStr.WHERE("a.og_arid_province = #{provinceID}");
		}
		if (null != condition.getStatus()) {
			sqlStr.WHERE("b.td_loginstate = #{status}");
		}
		if (null != condition.getThirdPartyPlatInfoID()) {
			sqlStr.WHERE("b.td_platid_fk = #{thirdPartyPlatInfoID}");
		}
		if (null != condition.getMcnNum() && "".equals(condition.getMcnNum())) {
			sqlStr.WHERE("b.td_mcn_num = #{mcnNum}");
		}
		if (null != condition.getPublicNum() && "".equals(condition.getPublicNum())) {
			sqlStr.WHERE("a.og_publicnum = #{publicNum}");
		}
		return sqlStr.toString();
	}
```

②、使用Provider注解（如下：@SelectProvider）在持久层对应方法中指定，相当于代替了CRUD的源注解（如：@Select）

```java
@Results({ 
    @Result(property = "mcnID", column = "td_mcn_num"),
    @Result(property = "provinceName", column = "ap_name"),
    @Result(property = "publicName", column = "og_publicnum"),
    @Result(property = "wechatNum", column = "og_wechatnum"),
    @Result(property = "number121", column = "us_username"),
    @Result(property = "mediaTypeName", column = "mt_name"),
    @Result(property = "lastTime", column = "og_expire_time"),
    @Result(property = "id", column = "tp_id"), 
    @Result(property = "platName", column = "tp_name"),
    @Result(property = "phone", column = "td_tel_num"), 
    @Result(property = "url", column = "tp_logo_url"),
    @Result(property = "status", column = "td_loginstate"
           )
})
@SelectProvider(method = "queryConditionNotEmpty",type=Test.class)
List<MCNInfoForm> selectOrganizationList(SearchMCNListForm condition);
```

*写在后面：不过本人在使用Provider注解替代CRUD注解时，通过单元测试报错，BuidlerException，由于时间的原因没来得及细分析原因，直接改成了使用"\<script>"标签的写法，再次先留下疑问，后续补充......*

#### 5、如何根据结果集封装复杂对象

注解版本将结果集映射到java的POJO对象还是有很多不完善的功能，这一点没有xml文件上的功能全面，本人体会的就是在一个对象里面有个集合属性的时候在映射里面的集合属性的时候就出现了问题，后来改用java代码来二次处理数据.....

##### 1）collection（对象内韩集合类型的属性）

```java
//Dao层的接口方法如下
List<MCNAccountSimpleForm> selectOrganizationList(SearchMCNListForm condition);
```

```java
//返回的对象如下
@SuppressWarnings("serial")
@Data
@AllArgsConstructor
@NoArgsConstructor
public class MCNAccountSimpleForm implements Serializable {

	private String mcnID;//mcnNum
	private String provinceName;// 省份名称
	private String publicName; // 公众号名称
	private String wechatNum; // 微信号
	private String number121; // 121账号
	private String mediaTypeName; // 媒体类型
	private String lastTime; // 授权到期日
	private List<ThirdPartyPlatForm> thirdPartyPlatFormList;// 第三方平台开通状况

}
```

```java
//上面对象对应的thirdPartyPlatFormList属性的内置对象属性如下
@SuppressWarnings("serial")
@Data
@AllArgsConstructor
@NoArgsConstructor
public class ThirdPartyPlatForm implements Serializable {
	
	private Long id;
	private String platName;// 平台名称
	private String phone;// 手机号
	private String url;// 主页链接
	private Integer status;// 登陆状态
	
}

```



如上面的情况，在结果集映射的时候，如果是在xml文件里面写

```xml
<!-- 如下为resultMap指定映射结果集 -->
<resultMap id="org_resultMap"
type="com.guwukeji.varyscommon.form.response.user.MCNAccountSimpleForm">
    <result property="mcnID" , column="td_mcn_num" />
    <result property="provinceName" , column="ap_name" />
    <result property="publicName" , column="og_publicnum" />
    <result property="wechatNum" , column="og_wechatnum" />
    <result property="number121" , column="us_username" />
    <result property="mediaTypeName" , column="mt_name" />
    <result property="lastTime" , column="og_expire_time" />
    <collection property="thirdPartyPlatFormList" >
        <result property = "id", column = "tp_id"/>
        <result property = "platName", column = "tp_name"/>
        <result property = "phone", column = "td_tel_num"/>
        <result property = "url", column = "tp_logo_url"/>
        <result property = "status", column = "td_loginstate"/>
    </collection>
</resultMap>
```

---以上为原来的mapper文件的写法，到了注解版，大同小异，下面来做演示

```java
@Select({"<script>",
		"SELECT b.td_mcn_num, c.ap_name,a.og_publicnum, a.og_wechatnum, f.us_username, d.mt_name, "
		+ "a.og_expire_time, e.tp_logo_url,e.tp_id, e.tp_name,b.td_tel_num, b.td_loginstate " + 
		"FROM tbl_varys_m_thirdplatinfo b,tbl_varys_m_organization a, tbl_varys_m_user f, "
		+ "tbl_varys_d_area_province c, tbl_varys_d_media_type d,tbl_varys_d_third_plat e " + 
		"WHERE b.td_orgid_fk = a.og_id AND b.td_mdid_fk = d.mt_id AND a.og_arid_province = c.ap_id "
		+ "AND b.td_platid_fk = e.tp_id AND b.td_usid_fk = f.us_uuid",
		"</script>"})
@Results({ 
    @Result(property = "mcnID", column = "td_mcn_num"),
    @Result(property = "provinceName", column = "ap_name"),
    @Result(property = "publicName", column = "og_publicnum"),
    @Result(property = "wechatNum", column = "og_wechatnum"),
    @Result(property = "number121", column = "us_username"),
    @Result(property = "mediaTypeName", column = "mt_name"),
    @Result(property = "lastTime", column = "og_expire_time"),
    @Result(property= "thirdPartyPlatFormList", column= "param",		many=@Many(select="com.guwukeji.varysucenter.dao.UserDao.selectThirdPlat"))
	})
List<MCNAccountSimpleForm> selectOrganizationList(SearchMCNListForm condition);
```

上述的@Many注解替代的就是"\<collection>"标签，注解里面的属性select的值另一个方法的返回的结果集所封装成的集合对象（即thirdPartyPlatFormList），需要注意的是@Many注解所在的@Result注解中的column属性里面填写的值是另一个子查询接口方法的参数

```java
@Select({"<script>",
		"....",//由于此处的sql并非重点，且在当前项目中本人也并没有使用此方法，所以就忽略不写
		"</script>"})
@Results({
    @Result(property = "id", column = "tp_id"), 
    @Result(property = "platName", column = "tp_name"),
    @Result(property = "phone", column = "td_tel_num"), 
    @Result(property = "url", column = "tp_logo_url"),
    @Result(property = "status", column = "td_loginstate")
})
List<ThirdPartyPlatForm> selectThirdPlat(String param);
```

综上：可以看出，使用注解版映射对象里面的集合属性相当于走了两次查询，而且在mybatis的官方文档中也指出了，***（@Many与@One注解）的联合映射在注解 API中是不支持的***，所以本人亲身体验注解版之后只想说，还是mapper文件好，且本人在实验中并没有成功，感觉也没有再次实验下去的必要，希望之后的注解版可以再次加强

##### 2）association（对象内涵单个自定义引用类型对象）

将上面的MCNAccountSimpleForm对象的属性换成单个ThirdPartyPlatForm对象，其他不变

```java
//返回的对象如下
@SuppressWarnings("serial")
@Data
@AllArgsConstructor
@NoArgsConstructor
public class MCNAccountSimpleForm implements Serializable {

	private String mcnID;//mcnNum
	private String provinceName;// 省份名称
	private String publicName; // 公众号名称
	private String wechatNum; // 微信号
	private String number121; // 121账号
	private String mediaTypeName; // 媒体类型
	private String lastTime; // 授权到期日
	private ThirdPartyPlatForm thirdPartyPlatFormList;// 第三方平台开通状况
}
```

```xml
<!-- 如下为resultMap指定映射结果集 -->
<resultMap id="org_resultMap"	type="com.guwukeji.varyscommon.form.response.user.MCNAccountSimpleForm">
    <result property="mcnID" , column="td_mcn_num" />
    <result property="provinceName" , column="ap_name" />
    <result property="publicName" , column="og_publicnum" />
    <result property="wechatNum" , column="og_wechatnum" />
    <result property="number121" , column="us_username" />
    <result property="mediaTypeName" , column="mt_name" />
    <result property="lastTime" , column="og_expire_time" />
    <association property="thirdPartyPlatFormList" >
        <result property = "id", column = "tp_id"/>
        <result property = "platName", column = "tp_name"/>
        <result property = "phone", column = "td_tel_num"/>
        <result property = "url", column = "tp_logo_url"/>
        <result property = "status", column = "td_loginstate"/>
    </association>
</resultMap>
```

至于但对象内的属性是自定义的引用类型，无非就是将注解@Many换成了@One

```java
@Select({"<script>",
		"SELECT b.td_mcn_num, c.ap_name,a.og_publicnum, a.og_wechatnum, f.us_username, d.mt_name, "
		+ "a.og_expire_time, e.tp_logo_url,e.tp_id, e.tp_name,b.td_tel_num, b.td_loginstate " + 
		"FROM tbl_varys_m_thirdplatinfo b,tbl_varys_m_organization a, tbl_varys_m_user f, "
		+ "tbl_varys_d_area_province c, tbl_varys_d_media_type d,tbl_varys_d_third_plat e " + 
		"WHERE b.td_orgid_fk = a.og_id AND b.td_mdid_fk = d.mt_id AND a.og_arid_province = c.ap_id "
		+ "AND b.td_platid_fk = e.tp_id AND b.td_usid_fk = f.us_uuid",
         "</script>"})
@Results({ 
    @Result(property = "mcnID", column = "td_mcn_num"),
    @Result(property = "provinceName", column = "ap_name"),
    @Result(property = "publicName", column = "og_publicnum"),
    @Result(property = "wechatNum", column = "og_wechatnum"),
    @Result(property = "number121", column = "us_username"),
    @Result(property = "mediaTypeName", column = "mt_name"),
    @Result(property = "lastTime", column = "og_expire_time"),
    @Result(property= "thirdPartyPlatFormList", column= "",
 //将注解@Many换成了@One           
            one=@One(select="com.guwukeji.varysucenter.dao.UserDao.selectThirdPlat"))
})
List<MCNAccountSimpleForm> selectOrganizationList(SearchMCNListForm condition);
```

```java
@Select({"<script>",
		"....",//由于此处的sql并非重点，且在当前项目中本人也并没有使用此方法，所以就忽略不写
		"</script>"})
@Results({
    @Result(property = "id", column = "tp_id"), 
    @Result(property = "platName", column = "tp_name"),
    @Result(property = "phone", column = "td_tel_num"), 
    @Result(property = "url", column = "tp_logo_url"),
    @Result(property = "status", column = "td_loginstate")
})
//此处返回的是一个对象，不再是集合
ThirdPartyPlatForm selectThirdPlat(String param);
```



写在后面：***最近做的一个子模块用户中心，一时心血来潮，想尝试不写mapper文件了，而用完注解版之后只想说，。。。。，好了大家还是用mapper文件吧，如果简单的sql可以用注解来实现，复杂的尤其是映射的对象比较复杂的，还是老老实实在mapper文件里面写吧***

最后跟搭建推荐一本感觉还不错的书*刘增辉*编著的《Mybatis从入门到精通》，下面是本人百度网盘分享链接

[点击这里](https://pan.baidu.com/s/1zOhaDl7Ef2Pj640gD5pkUQ )  提取码：if38 

[下一篇](https://silentself.github.io/articles/2018-12/mybatis-annotation-sql-2)









