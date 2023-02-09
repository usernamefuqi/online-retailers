**java后端面试问题**

1. **某电商业务场景1：**

业务背景：电商实际业务中，不同的商品，适合在不同的区域售卖，比如空气净化器，优先适合在空气质量不好的城市售卖，雪地靴，优先适合在北方天气冷的地区售卖，即每个城市适合按当地用户喜好，优先展示不同的商品。

已知：

目前商城系统商品表大体设计如下：

```
-- 商品表
CREATE TABLE `cy_goods` (   
 `id` int(10) unsigned NOT NULL AUTO_INCREMENT,   
  `goods_name` varchar(100) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT '商品名称',    
  `cate_id` int(10) unsigned NOT NULL COMMENT '分类ID',  
  `price` bigint(20) unsigned NOT NULL,   
  `original` bigint(20) unsigned NOT NULL COMMENT '商品原价',
  `tags` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT '商品标签',  
  `content` text COLLATE utf8mb4_unicode_ci NOT NULL COMMENT '商品内容', 
  `summary` text COLLATE utf8mb4_unicode_ci NOT NULL COMMENT '商品描述',  
  `is_sale` tinyint(4) NOT NULL COMMENT '上架状态: 1是0是',
  `created_at` timestamp NULL DEFAULT NULL,   
  `updated_at` timestamp NULL DEFAULT NULL,   
  PRIMARY KEY (`id`),   
  KEY `goods_cate_id_foreign` (`cate_id`),   
  CONSTRAINT `goods_cate_id_foreign` FOREIGN KEY (`cate_id`) REFERENCES `cy_categories` (`id`) ON DELETE CASCADE ON UPDATE CASCADE ) 
 ENGINE=InnoDB AUTO_INCREMENT=18 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci

 -- 分类表 --
 CREATE TABLE `cy_categories` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `pid` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '父级分类ID，0为顶级分类',
  `cate_name` varchar(50) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT '分类名称',
  `sort` smallint(5) unsigned NOT NULL DEFAULT '99' COMMENT '排序字段',
  `created_at` timestamp NULL DEFAULT NULL,
  `updated_at` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `categories_cate_name_unique` (`cate_name`)
) ENGINE=InnoDB AUTO_INCREMENT=33 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci

-- sku表 --
CREATE TABLE `cy_goods_sku` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `goods_id` int(10) unsigned NOT NULL COMMENT '商品ID',
  `title` varchar(100) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT '规格名称',
  `num` int(10) unsigned NOT NULL COMMENT 'SKU库存',
  `price` bigint(20) unsigned NOT NULL COMMENT '商品售价',
  `properties` varchar(255) COLLATE utf8mb4_unicode_ci DEFAULT NULL COMMENT '商品属性，以逗号分隔',
  `bar_code` varchar(50) COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '条码',
  `goods_code` varchar(100) COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '商品码',
  `status` tinyint(4) NOT NULL DEFAULT '1' COMMENT '状态:1启用,0禁用',
  `created_at` timestamp NULL DEFAULT NULL,
  `updated_at` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `goods_sku_goods_id_foreign` (`goods_id`),
  CONSTRAINT `goods_sku_goods_id_foreign` FOREIGN KEY (`goods_id`) REFERENCES `cy_goods` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB AUTO_INCREMENT=6 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
```

1.如何让返回给前端的商品列表，区域(即城市)动态变化。

2.假定上述效果已经完成实现，如何扩展现有系统，在不影响现有版本(无区域化差异)的情况下，做对比测试？

给出大致设计思路和实现？



第一题解决方案：

对于数据库的构建我们可以设置对应的省份表、市表、区域表，城市区域对应特征表，按照用户所在的城市的区域（通过定位或者用户自主选择来实现），查找其区域对应特征表，通过区域对应特征表中的关联商品类别，去查询商品并优先显示对应特征的商品。

对应表设计：

```
 省份表：

 CREATE TABLE `cy_province` (

`p_id` varchar(50) NOT NULL COMMENT '省份编号',

`p_name` varchar(50) NOT NULL COMMENT '省份名称',

`a_id` varchar(50) COMMENT '区域编号',

PRIMARY KEY (`p_id`), 

CONSTRAINT `province_a_id_foreign` FOREIGN KEY (`a_id`) REFERENCES `cy_area` (`a_id`) ON DELETE CASCADE ON UPDATE CASCADE 

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci 

城市表：

CREATE TABLE `cy_city` ( 

`c_id` varchar(50) NOT NULL COMMENT '城市编号', `c_name` varchar(50) NOT NULL COMMENT '城市名称',

`p_id` varchar(50) COMMENT '省份编号', 

PRIMARY KEY (`c_id`), 

CONSTRAINT `city_p_id_foreign` FOREIGN KEY (`p_id`) REFERENCES `cy_province` (`p_id`) ON DELETE CASCADE ON UPDATE CASCADE 

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci

区域表：

CREATE TABLE `cy_area` (

`a_id` varchar(50) NOT NULL COMMENT '区域编号',

`a_name` varchar(50) NOT NULL COMMENT '区域名称', 

`a_desc` varchar(200) NOT NULL COMMENT '区域描述',

PRIMARY KEY (`a_id`) ) 

ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci

区域对应特征表：

CREATE TABLE `cy_area_categories` ( 

`id` int(10) unsigned NOT NULL AUTO_INCREMENT, 

`a_id` varchar(50) COMMENT '区域编号',

`categories_id` int(10) COMMENT '商品分类编号', 

`level` int(10) COMMENT '特征级别', 

`ac_desc` varchar(50) NOT NULL COMMENT '特征描述',

PRIMARY KEY (`id`), 

CONSTRAINT `cates_aid_foreign` FOREIGN KEY (`a_id`) REFERENCES `cy_area` (`a_id`) ON DELETE CASCADE ON UPDATE CASCADE, CONSTRAINT `categories_id_foreign` FOREIGN KEY (`categories_id`) REFERENCES `cy_categories` (`id`) ON DELETE CASCADE ON UPDATE CASCADE 

) ENGINE=InnoDB AUTO_INCREMENT=6 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci 
```



mapper.xml对应分页查询语句如下：

```java
<!--商品表和库存表外连接区域对应特征表查询商品启用的信息，按区域特征等级降序分页查询-->
<select id="selectByPage" resultType="goods">
    select g.* from cy_goods g inner join cy_goods_sku k on g.id=k.goods_id left join cy_area_categories c on c.categories_id=g.cate_id and c.a_id=(select a_id from cy_province where p_id=#{pid}) order by c.level desc
    limit ${limit},${size} 
</select>
```

前端用户通过当前用户地理位置，获取所在省份信息，将省份编号传递回后台，实现分页查询

```js
//获取当前用户所在省份编号，发送ajax请求，局部刷新页面，实现分页查询商品功能
var pid=$("#pid");
$.ajax({
    url: baseURL+"goods/selectBypage",
    dataType: "json",
    data: {
        pid:pid,
    },
    success: function (data) {
        this.list=data.goodslist;
    }
});

```

还有一种方法：

可以通过用户选择对应省份城市区域，然后通过区域对应特征表中的关联商品类别，去查询商品并优先显示对应特征的商品,并进行排序。

数据库设计：

```
区域表：

CREATE TABLE `cy_area` (

`a_id` varchar(50) NOT NULL COMMENT '区域编号',

`a_name` varchar(50) NOT NULL COMMENT '区域名称', 

`a_desc` varchar(200) NOT NULL COMMENT '区域描述',

PRIMARY KEY (`a_id`) ) 

ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci

区域对应特征表：

CREATE TABLE `cy_area_categories` ( 

`id` int(10) unsigned NOT NULL AUTO_INCREMENT, 

`a_id` varchar(50) COMMENT '区域编号',

`categories_id` int(10) COMMENT '商品分类编号', 

`level` int(10) COMMENT '特征级别', 

`ac_desc` varchar(50) NOT NULL COMMENT '特征描述',

PRIMARY KEY (`id`), 

CONSTRAINT `cates_aid_foreign` FOREIGN KEY (`a_id`) REFERENCES `cy_area` (`a_id`) ON DELETE CASCADE ON UPDATE CASCADE, CONSTRAINT `categories_id_foreign` FOREIGN KEY (`categories_id`) REFERENCES `cy_categories` (`id`) ON DELETE CASCADE ON UPDATE CASCADE 

) ENGINE=InnoDB AUTO_INCREMENT=6 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci 
```

mapper.xml对应分页查询语句如下：

```java
<!--商品表和库存表外连接区域对应特征表查询商品启用的信息，按区域特征等级降序分页查询-->
<select id="selectByPage" resultType="goods">
    select g.* from cy_goods g inner join cy_goods_sku k on g.id=k.goods_id left join cy_area_categories c on c.categories_id=g.cate_id and c.a_id=#{aid} order by c.level desc  limit ${limit},${size} 
</select>
```

 前端用户选择对应省份城市区域

```
vue+elementUi：
安装城市数据：npm install element-china-area-data -S
1.导入三级数据：
import { regionDataPlus } from 'element-china-area-data'

html：
<template>
  <div id="app">
    <el-cascader
      size="large"
      :options="options"
      v-model="selectedOptions"
      @change="handleChange">
    </el-cascader>
  </div>
</template>

js：
export default {
    data () {
      return {
        options: regionDataPlus ,
        selectedOptions: []
      }
    },
 
    methods: {
      handleChange () {
        var loc = "";
        for (let i = 0; i < this.selectedOptions.length; i++) {
            loc += CodeToText[this.selectedOptions[i]];
        }
        console.log(loc)//打印区域码所对应的属性值即中文地址
        //然后通过选择的城市区域调用邮编查询接口去查询对应的邮编
        var aid=//调用查询接口得到的区域邮编号;
        $.ajax({
            url: baseURL+"goods/selectBypage",
            dataType: "json",
            data: {
                aid:aid,
            },
            success: function (data) {
                this.list=data.goodslist;
            }
        });
      }
    }

//邮编查询接口： http://api.k780.com
```



第二题解决方案：

需要对现有功能进行扩展，保留原来的项目代码不动，复制原有GoodsController建立第二个GoodsControllercp，改变对应查询地址，继承原有的GoodsService接口，创建新的GoodsServiceImpl，重写商品分页查询方法，对于service可以使用@Resource注解按byName注入，实现然后可以与原有的功能进行效果对比。



**2.某电商业务场景2：**

同上，运营推广部门策划活动后，曾遇到恶意刷单或者系统被注水的情况。为保护开放接口不被恶意调用，请设计一个方案，对恶意接口请求流量做区分，同时不能影响对正常访问用户。

解决方案：

1.在redis数据库db0中新建一个名为rd_sms_request_count表，表结构：

Ip:客户请求的ip

Success_count:成功次数

Failure_count:失败次数

Is_close:是否已经加入到防火墙黑名单，1是  0否 ，默认0

2.结合业务，判单哪个ip是恶意的，每当一个ip访问接口，按照代码返回码，如果是都是错误请求，添加到redis数据库中的Failure_count字段加1，如果都是返回正确结果，那么Success_count加1，Java后台启动一个线程,每天统计一次rd_sms_request_count，先删除rd_sms_request_coun表的Success_count大于0的记录，证明这个ip是正常用户；如果Success_count等于0而且Failure_count大于10000（规则可自定义）就通过java代码把这个ip加入到iptables黑名单中，加入到iptables黑名单中的记录设置为1。当然也可以通过运维人员查看日志，判断恶意ip，手动加入到黑名单中。

3.把满足条件的ip加入到iptables黑名单中核心代码

```
String[] cmd = new String[] { "/bin/sh", "-c", "iptables -I INPUT -s 211.1.0.24 -j DROP;iptables -I INPUT -s 211.1.0.25 -j

DROP" };

        try {
    
        Runtime.getRuntime().exec(cmd);
    
       } catch (Exception e) {
    
           e.printStackTrace();
    
       }finally {
    
       Runtime.getRuntime().exec("service iptables save");

}
```

