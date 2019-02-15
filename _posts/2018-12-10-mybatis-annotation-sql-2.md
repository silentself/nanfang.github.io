---
layout: post
title:  mybatis-注解版<2>动态sqlbug
excerpt: "mybatis-注解版sql一直以来被应用的很少，主要是由于sql修改之后还需要编译之后才能运行......"
categories: [java]
tags: [mybatis]
comments: true
---

[上一篇](https://silentself.github.io/articles/2018-11/mybatis-annotation-sql-1)

#### 1、动态sql不支持单引号

```java
@Select({"<script>",
		"SELECT mnh.mnh_mcn_num, mf.mf_name, mt.mt_name, ma.ma_name, mnh.mnh_plat_name, mnh.mnh_content",
		"FROM ( SELECT * FROM tbl_varys_m_mcnnotification_h h <where>", 
		"<if test = 'mcnNum != null and mcnNum != \'\' '>	h.mnh_mcn_num = #{mcnNum} </if>", 
		"<if test = 'belongDate != null'> AND h.mnh_belong_date = #{belongDate} </if>", 
		"<if test = 'platId != null '>	AND h.mnh_platid_fk = #{platId} </if>", 
		"<if test = 'content != null and content != \'\' '>	AND h.mnh_content LIKE concat('%',#{content},'%') </if>", 
		"</where>) mnh", 
		"LEFT JOIN tbl_varys_m_mcninfo mf ON mf.mf_mcnid = mnh.mnh_mcn_num", 
		"LEFT JOIN tbl_varys_m_organization og ON og.og_id = mf.mf_orgid_fk", 
		"LEFT JOIN tbl_varys_d_media_attr ma ON ma.ma_id = og.og_mediaattr_fk", 
		"LEFT JOIN tbl_varys_d_media_type mt ON mt.mt_id = mf.mf_mtlx_fk", 
		"<if test = 'mediaTypeId != null '> AND mf.mf_id = #{mediaTypeId} </if>",
		"limit #{startIndex} , #{pageSize}","</script>"
	})
```

启动报错:

```java
2018-12-10 17:56:43.366 ERROR 11148 --- [           main] o.m.spring.mapper.MapperFactoryBean      : Error while adding the mapper 'interface com.guwukeji.varysstat.dao.MCNPlatNotificDao' to configuration.

org.apache.ibatis.builder.BuilderException: Could not find value method on SQL annotation.  Cause: org.apache.ibatis.builder.BuilderException: Error creating document instance.  Cause: org.xml.sax.SAXParseException; lineNumber: 1; columnNumber: 208; 元素类型 "if" 必须后跟属性规范 ">" 或 "/>"。
```

修改之后：（**将\\'\\' 修改成\\"\\"**）

```java
@Select({"<script>",
		"SELECT mnh.mnh_mcn_num, mf.mf_name, mt.mt_name, ma.ma_name, mnh.mnh_plat_name, mnh.mnh_content",
		"FROM ( SELECT * FROM tbl_varys_m_mcnnotification_h h <where>", 
		"<if test = 'mcnNum != null and mcnNum != \"\" '>	h.mnh_mcn_num = #{mcnNum} </if>", 
		"<if test = 'belongDate != null'> AND h.mnh_belong_date = #{belongDate} </if>", 
		"<if test = 'platId != null '>	AND h.mnh_platid_fk = #{platId} </if>", 
		"<if test = 'content != null and content != \"\" '>	AND h.mnh_content LIKE concat('%',#{content},'%') </if>", 
		"</where>) mnh", 
		"LEFT JOIN tbl_varys_m_mcninfo mf ON mf.mf_mcnid = mnh.mnh_mcn_num", 
		"LEFT JOIN tbl_varys_m_organization og ON og.og_id = mf.mf_orgid_fk", 
		"LEFT JOIN tbl_varys_d_media_attr ma ON ma.ma_id = og.og_mediaattr_fk", 
		"LEFT JOIN tbl_varys_d_media_type mt ON mt.mt_id = mf.mf_mtlx_fk", 
		"<if test = 'mediaTypeId != null '> AND mf.mf_id = #{mediaTypeId} </if>",
		"limit #{startIndex} , #{pageSize}","</script>"
	})
```

成功启动！





