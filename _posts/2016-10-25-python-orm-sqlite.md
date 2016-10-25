---
layout: post
title: python orm使用Demo
date: 2016-10-25 14:32:56
category: 技术
tags: Python
excerpt: Python orm使用Demo
---

python orm使用Demo

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

import sqlalchemy
import sqlalchemy.exc
import sqlalchemy.ext.declarative
import sqlalchemy.orm
import sqlalchemy.sql.functions

Base = sqlalchemy.ext.declarative.declarative_base()

class Version (Base):
    "Schema version for the search-index database"
    __tablename__ = 'version'

    id = sqlalchemy.Column(sqlalchemy.Integer, primary_key=True)

    def __repr__(self):
        return '<{0}(id={1})>'.format(type(self).__name__, self.id)



if __name__ == '__main__':
    database = "sqlite:///docker-registry.db"
    engine = sqlalchemy.create_engine(database)
    session = sqlalchemy.orm.sessionmaker(bind=engine)
    version = 1
    _session = session()
    if engine.has_table(table_name=Version.__tablename__):
        _version = _session.query(
            sqlalchemy.sql.functions.max(Version.id)).first()[0]
        if _version == version:
            print 'exist version table'
    else:
        Base.metadata.create_all(engine)
        _session.add(Version(id=version))
        _session.commit()
        print ('create sqlite and version table')

    _session.close()
    print 'final'
```

第一次运行

```bash
[root@CentOS-64-duyanghao SQLite_operation]# python orm_sqlite.py
create sqlite and version table
final
```

第二次运行

```bash
[root@CentOS-64-duyanghao SQLite_operation]# python orm_sqlite.py
exist version table
final
```

```bash
#!/bin/bash 
# list a content summary of a number of RPM packages 
# USAGE: showrpm rpmfile1 rpmfile2 ... 
# EXAMPLE: showrpm /cdrom/RedHat/RPMS/*.rpm 
for rpmpackage in $*; do
　if [ -r "$rpmpackage" ];then
　　echo "=============== $rpmpackage =============="
　　rpm -qi -p $rpmpackage 
　else
　　echo "ERROR: cannot read file $rpmpackage"
　fi
done
```

```perl
#!/usr/bin/perl
 
use strict;
use warnings;
use Getopt::Std;
 
sub show_help {
  print "Useage:\n";
  print "newp -aAnsl\n";
  print "-a\t\t the password contains lower case letters(a-z)\n";
  print "-A\t\t the password contains upper case letters(A-Z)\n";
  print "-n\t\t the password contains numerical character(0-9)\n";
  print "-s\t\t the password contains special symbols\n";
  print "-u\t\t the password contains only unique characters\n";
  print "-l length\t set the password length(default: 6)\n";
 
  exit 0;
}
 
sub show_version {
  print "Version: 0.2.1 Changed the default option: -l 9 -Ana. 2016-4-15\n";
 
  exit 0;
}
 
### main program
 
use vars qw($opt_a $opt_A $opt_h $opt_l $opt_n $opt_s $opt_u $opt_v);
getopts('aAhl:nsuv');
 
&show_version if $opt_v;
&show_help if $opt_h;
 
my $len = $opt_l || 9;  # default length 9
my $opt_cnt = 0;
my @rand_str = ();
 
# store all the characters
my @num = qw(0 1 2 3 4 5 6 7 8 9);
my @ABC = qw(A B C D E F G H I J K L M N O P Q R S T U V W X Y Z);
my @abc = qw(a b c d e f g h i j k l m n o p q r s t u v w x y z);
# my @sym = qw(! " $ % & ' * + - . / : ; < = > ? @ [ \ ] ^ _ ` { | } ~);
my @sym = qw(! $ % & * + - . / : ; < = > ? @ [ ] ^ _ ` { | } ~); # no " ' \
unshift (@sym, '(', ')', '#', ','); # to prevent perl's complains or warnings.
my @all_sym = (@num, @ABC, @abc, @sym);
my @ch_src = ();
 
if ((!$opt_a) && (!$opt_A) && (!$opt_n) && (!$opt_s)) {
  $opt_a++;
  $opt_A++;
  $opt_n++;
}
 
if ($opt_a) {
  $opt_cnt++;
  my $i = rand @abc;
  unshift @rand_str, $abc[$i];
 
  if ($opt_u) {
    if ($i>=1) {
      $abc[$i-1] = shift @abc;
    } else {
      shift @abc;
    }
  }
 
  unshift (@ch_src, @abc);
}
 
if ($opt_A) {
  $opt_cnt++;
  my $i = rand @ABC;
  unshift @rand_str, $ABC[$i];
 
  if ($opt_u) {
    if ($i>=1) {
      $ABC[$i-1] = shift @ABC;
    } else {
      shift @ABC;
    }
  }
 
  unshift (@ch_src, @ABC);
}
 
if ($opt_n) {
  $opt_cnt++;
  my $i = rand @num;
  unshift @rand_str, $num[$i];
 
  if ($opt_u) {
    if ($i>=1) {
      $num[$i-1] = shift @num;
    } else {
      shift @num;
    }
  }
 
  unshift (@ch_src, @num);
}
 
if ($opt_s) {
  $opt_cnt++;
  my $i = rand @sym;
  unshift @rand_str, $sym[$i];
 
  if ($opt_u) {
    if ($i>=1) {
      $sym[$i-1] = shift @sym;
    } else {
      shift @sym;
    }
  }
 
  unshift (@ch_src, @sym);
}
 
if ($len < $opt_cnt) {
  print "The count of characters[$len] should not be smaller " .
     "than count of character types[$opt_cnt].\n";
  exit -1;
}
 
if ($opt_u && $len > (@ch_src + @rand_str)) {
  print "The total number of characters[".(@ch_src + @rand_str).
     "] which could be contained " .
     "in password is smaller than the length[$len] of it.\n";
  exit -1;
}
 
foreach (1..$len-$opt_cnt) {
  my $i = rand @ch_src;
  unshift @rand_str, $ch_src[$i];
 
  if ($opt_u) {
    if ($i>=1) {
      $ch_src[$i-1] = shift @ch_src;
    } else {
      shift @ch_src;
    }
  }
}
 
foreach (1..$len) {
  my $i = rand @rand_str;
  print $rand_str[$i];
 
  if ($i>=1) {
    $rand_str[$i-1] = shift @rand_str;
  } else {
    shift @rand_str;
  }
}
 
print "\n";
exit 0;
```

```java
package com.qyf404.learn.maven;

import org.junit.After;
import org.junit.Assert;
import org.junit.Before;
import org.junit.Test;

public class AppTest {
    private App app;
    @Before
    public void setUp() {
        app = new App();
    }
    @Test
    public void testAdd() throws InterruptedException {
        int a = 1;
        int b = 2;
        int result = app.add(a, b);
        Assert.assertEquals(a + b, result);
    }
    @After
    public void tearDown() throws Exception {
    }
}
```

```c#
using System;
using UnityEngine;


// ReSharper disable All


public class ChaserPlayer : MonoBehaviour
{
    public float moveSpeed = 0.1f;//以编辑器里的为准
    
    //related player
    public GameObject player=null;
    //self position
    private Transform cachedTransform;
    //target position
    private Vector2 nextpoint;
    //whether crash
    private bool is_crash;
    private Vector3 choose_pos;
    private Vector3 inf;

  
    //event
    private LogicEventHelper _logicEventHelper = new LogicEventHelper();
   
    
    #region  by fightingdu
    void Awake()
    {
        //...
        cachedTransform = gameObject.transform;
        //get GameObject
        //player_object = GameObject.FindWithTag("player");
        //rect_object = GameObject.FindWithTag("rect");
        //cir_object = GameObject.FindWithTag("cir");
        //init chaser state
        //
        is_crash = false;
        choose_pos = new Vector3(0,0,-1000);
        inf = new Vector3(0,0,-1000);
    }

    void FixedUpdate()
    {
        //move
        if (player == null)
        {
            return;
        }
        if (choose_pos == cachedTransform.position)
        {
            is_crash = false;
            choose_pos = inf;
        }
        if (!is_crash)
        {
            nextpoint = player.transform.position;
        }

        MoveAndTurn();
    }
    //switch to the latest escape point
    void OnTriggerEnter2D(Collider2D other)
    {
        //player
        if (other.gameObject == player)
        {
            //TODO 现发送 to_die消息
            EventParam ep = EventDispatcher.GetInstance().GetEventParam();
            EventDispatcher.GetInstance().DispatchEvent((uint)LogicEvent.PLAYER_ON_DIE, ep);
        }
        else //other obstacles
        {
            is_crash = true;
            float dis = 10000;
            for (int i=0; i<other.transform.childCount; i++)
            {
                Debug.Log(other.transform.childCount);
                Debug.Log("yes");
                Transform transform = other.transform.GetChild(i);
                EscapePoint ep = transform.GetComponent<EscapePoint>();
                if (ep)
                {
                    float tmp = (ep.pos - cachedTransform.position).magnitude;
                    if (tmp < dis)
                    {
                        dis = tmp;
                        nextpoint = ep.pos;
                    }
                }
   
            }
            choose_pos = nextpoint;
            Debug.Log("enter crash");
        }
        //(other.gameObject.transform.position - other.gameObject.transform.position).magnitude

    }
    //switch to the player position
    void OnTriggerExit2D(Collider2D other)
    {
        //set target to player
        //nextpoint = player.transform.position;
        //reset crash state
        is_crash = false;
        Debug.Log("out crash");
    }

    //add die event 
    void OnEnable()
    {
        _logicEventHelper.AddEventListener((uint)LogicEvent.PLAYER_ON_DIE, Dead);
    }
    //destroy die event
    void OnDestroy()
    {
        _logicEventHelper.RemoveEventListener((uint)LogicEvent.PLAYER_ON_DIE, Dead);
    }
    //kill itself
    void Dead(EventParam param)
    {
        //destory self
        Destroy(gameObject);
        //is_crash = false;
    }
    //move towards target position
    void MoveAndTurn()
    {
    
        //cachedTransform.position = Vector3.MoveTowards(cachedTransform.position, (Vector3) mousePosition, moveSpeed*Time.deltaTime);
        cachedTransform.position = Vector3.Lerp(cachedTransform.position, (Vector3)nextpoint, moveSpeed * Time.deltaTime);
        //TODO: 慢慢地转向，然后变速朝目标点前进
    }

    #endregion



}
```

```php

<?php 
$password = "1234"; // 这里是密码 
$p = ""; 
if(isset($_COOKIE["isview"]) and $_COOKIE["isview"] == $password){ 
$isview = true; 
}else{ 
if(isset($_POST["pwd"])){ 
if($_POST["pwd"] == $password){ 
setcookie("isview",$_POST["pwd"],time()+3600*3); 
$isview = true; 
}else{ 
$p = (empty($_POST["pwd"])) ? "需要密码才能查看，请输入密码。" : "密码不正确，请重新输入。"; 
} 
}else{ 
$isview = false; 
$p = "请输入密码查看，获取密码可联系我。"; 
} 
} 
if($isview){ ?> 
这里是密码成功后显示的地方 
<?php }else{ ?> 
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" " http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd"> 
<html xmlns=" http://www.w3.org/1999/xhtml"> 
<head> 
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" /> 
<meta http-equiv="pragma" content="no-cache" /> 
<meta http-equiv="cache-control" content="no-cache" /> 
<meta http-equiv="expires" content="0" /> 
<title>脚本之家提醒你输入密码</title> 
<!--[if lt IE 6]> 
<style type="text/css"> 
.z3_ie_fix{ 
float:left; 
} 
</style> 
<![endif]--> 
<style type="text/css"> 
<!-- 
body{ 
background:none; 
} 
.passport{ 
border:1px solid red; 
background-color:#FFFFCC; 
width:400px; 
height:100px; 
position:absolute; 
left:49.9%; 
top:49.9%; 
margin-left:-200px; 
margin-top:-55px; 
font-size:14px; 
text-align:center; 
line-height:30px; 
color:#746A6A; 
} 
--> 
</style> 
<div class="passport"> 
<div style="padding-top:20px;"> 
<form action="?yes" method="post" style="margin:0px;">输入查看密码 
<input type="password" name="pwd" /> <input type="submit" value="查看" /> 
</form> 
<?php echo $p; ?> 
</div> 
</div> 
<?php 
} ?> 
</body> 
</html> 

```

```css
* sext
*
* @author linghua.zhang@me.com
* @link http://lhzhang.com/
* @update 2011-12-20
* 
* |/ | (~ (~
* |\.|._)._)
* --------------------------------- */

@import url(http://fonts.googleapis.com/css?family=Galdeano);
@import url(http://fonts.googleapis.com/css?family=Electrolize);
@import url(http://fonts.googleapis.com/css?family=Cuprum);
body{font-size:14px;word-spacing:1px;
  margin-left:auto;
margin-right:auto;
  font-family:"Trebuchet MS","Galdeano","Hiragino Sans GB","Microsoft YaHei",Trebuchet,Tahoma,"Lucida Grande","Lucida Sans
 Unicode",Verdana,sans-serif;color:#000000;background-color:#ffffff;}
*{padding:0;margin: 0;border:none;outline:none;}
a{color:#dd0000;text-decoration:none;}
a:hover{color:#333333;text-decoration:underline;}
hr{margin:.7em 0;border-top:1px dashed #d0d0d0;border-bottom:1px dashed #f9f9f9;}
p{padding:.5em 0;}
li{padding:.2em 0;}
ol,ul{list-style-position:inside;}
ol ul, ul ol, ul ul, ol ol {margin-left:1em;}
pre code{margin:1em 0;font-size:13px;line-height:1.6;display:block;overflow:auto;}
blockquote{display:block;text-align:justify;border-left:4px solid #eeeeee;}
blockquote{margin:1em 0em;padding-left:1.5em;}
blockquote p{margin:0;padding:.5em 0;}
#container{width:900px;margin:2em auto;display:block;padding:0 2em;}
.content{font-size:16px;line-height:24px;}
```

```js
var data="  
{  
root:  
[  
{name:'1',value:'0'},  
{name:'6101',value:'北京市'},  
{name:'6102',value:'天津市'},  
{name:'6103',value:'上海市'},  
{name:'6104',value:'重庆市'},  
{name:'6105',value:'渭南市'},  
{name:'6106',value:'延安市'},  
{name:'6107',value:'汉中市'},  
{name:'6108',value:'榆林市'},  
{name:'6109',value:'安康市'},  
{name:'6110',value:'商洛市'}  
] 
}";  
alert(eval("{}"); // return undefined 
alert(eval("({})");// return object[Object]
(function(){ 
 
})(jQuery);  //做闭包操作 
$.getJSON("http://www.exp99.com/",{param:"USER_GUID"},function(data){  
//此处返回的data已经是json对象  
  $.each(data.root,function(idx,item){  
    if(idx==0){  
      return true;//同continue，返回false同break  
    }  
    console.log("name:"+item.name+",value:"+item.value);  
  });  
}); 
```

```html
---
layout: page_title
title: <i class="fa fa-user fa-lg"></i>
titlename: 关于
---
<h3>关于博客</h3>
<p style=font-size:18px><B>duyanghao.github.io</B>是我的个人博客。用于学习、技术、生活等总结沉淀。</p>

<h3>关于作者</h3>
<p style=font-size:18px>一个对生活充满激情，并希望有所成就的年轻人！</p>

<center><img src="/public/img/love.jpg" style="width:100%"></center>

<hr>
<a href="{{ site.author.linkedin }}">
  <i class="fa fa-linkedin fa-lg" style="color:#16a095;"></i>
</a>
<a href="{{ site.author.weibo }}" target="_blank">
  <i class="fa fa-weibo fa-lg" style="color:#16a095;"></i>
</a>
<a href="{{ site.author.github }}" target="_blank">
  <i class="fa fa-github fa-lg" style="color:#16a095;"></i>
</a>
<a href="/pages/atom.xml" target="_blank">
  <i class="fa fa-rss fa-lg" style="color:#16a095;"></i>
</a>
<a href="mailto:{{ site.author.email }}">
  <i class="fa fa-envelope-o fa-lg" style="color:#16a095;"></i>
</a>
<a href="{{ site.author.music }}">
  <i class="fa fa-music fa-lg" style="color:#16a095;"></i>
</a>
<hr>

<h4>关于模板</h4>
<p style=font-size:12px>模板来源于<a href="https://github.com/lay1010/lay1010.github.io">PainterLin.com</a></p>
```


```
---
title: 首页
layout: page
---

<ul class="listing">
{% for post in paginator.posts %}
  {% capture y %}{{post.date | date:"%Y"}}{% endcapture %}
  {% if year != y %}
    {% assign year = y %}
    <li class="listing-seperator">{{ y }}</li>
  {% endif %}
  <li class="listing-item">
    <time datetime="{{ post.date | date:"%Y-%m-%d" }}">{{ post.date | date:"%Y-%m-%d" }}</time>
    <a href="{{ post.url }}" title="{{ post.title }}">{{ post.title }}</a>
    <p>   
      {{ post.excerpt | remove: '<p>' | remove: '</p>' }} &raquo;
          <a href="{{ post.url }}">read more...</a>
          </p>
  </li>
{% endfor %}
</ul>

<div id="post-pagination" class="paginator">

  {% if paginator.previous_page %}
    {% if paginator.previous_page == 1 %}
    <a href="/"><前页</a>
    {% else %}
    <a href="/page{{paginator.previous_page}}">&lt;前页</a>
    {% endif %}
  {% else %}
    <span class="previous disabled">&lt;前页</span>
  {% endif %}

      {% if paginator.page == 1 %}
      <span class="current-page">1</span>
      {% else %}
      <a href="/">1</a>
      {% endif %}

    {% for count in (2..paginator.total_pages) %}
      {% if count == paginator.page %}
      <span class="current-page">{{count}}</span>
      {% else %}
      <a href="/page{{count}}">{{count}}</a>
      {% endif %}
    {% endfor %}

  {% if paginator.next_page %}
    <a class="next" href="/page{{paginator.next_page}}">后页&gt;</a>
  {% else %}
    <span class="next disabled" >后页&gt;</span>
  {% endif %}
  (共{{ paginator.total_posts }}篇)
</div>
```

```js
<html> 
<head> 
<script language="javascript"> 
//行的追加 
function addRow() { 
var testTable = document.getElementById("testTable"); 
var bodies = testTable.tBodies; 
var aBody = null; 
if(bodies){ 
aBody = bodies[0]; 
} 
if(aBody){ 
var row = document.createElement("tr"); 
for(var i = 0 ; i < testTable.tHead.rows[0].cells.length; i++){ 
var cell = document.createElement("td"); 
var str = "内容第" + (aBody.rows.length + 1) + "行第" + (i + 1) + "列"; 
if(i == (testTable.tHead.rows[0].cells.length - 1)) { 
str = "  <a href='javascript:void(0);' onclick=\"removeRow(this);\">删除</a>"; 
}else if(i == 0){ 
str = "<input type=\"radio\" name=\"RAd\" >"; 
} cell.innerHTML = str; 
row.appendChild(cell); 
} 
aBody.insertBefore(row); 
} 
} 
//行的删除 
function removeRow(obj) { 
var testTable = document.getElementById("testTable"); 
var bodies = testTable.tBodies; 
var aBody = null; 
if(bodies){ 
aBody = bodies[0]; 
if(aBody){ 
aBody.removeChild(obj.parentNode.parentNode); 
} 
} 
} 
//行的上移 
function moveUp(src){ 
  var rowIndex = 0;
  var rad = document.getElementsByName("RAd");
  for(var i = 0; i < rad.length; i++){
   if(rad[i].checked){
  rowIndex = rad[i].parentElement.parentElement.rowIndex; 
 }
  }
 if (rowIndex >= 2){ 
 change_row(rowIndex-1,rowIndex); 
} 
} 
//行的下移 
function moveDown(src){ 
  var rowIndex = 0;
  var rad = document.getElementsByName("RAd");
  for(var i = 0; i < rad.length; i++){
   if(rad[i].checked){
  rowIndex = rad[i].parentElement.parentElement.rowIndex; 
 }
  }
var tl = document.getElementById("testTable"); 
if (rowIndex < tl.rows.length - 1){ 
change_row(rowIndex + 1,rowIndex); 
} 
} 
function change_row(line1, line2){ 
var tl = document.getElementById("testTable"); 
tl.rows[line1].swapNode(tl.rows[line2]); 
} 
</script> 
</head> 
<body> 
<div> 
<table id="testTable" border="1" width="80%"> 
<thead> 
<tr> 
<th scope="col">单选按钮</th> 
<th scope="col">序列</th> 
<th scope="col">ID</th> 
<th scope="col">名字</th> 
</tr> 
</thead> 
</table> 
<input type="button" name="addButton" value="追加一行" onClick="addRow();"/> 
<input type="button" name="moveUP" value="上移一行" onClick="moveUp(this);"/> 
<input type="button" name="moveDOWN" value="下移一行" onClick="moveDown(this);"/> 
</div> 
</body> 
</html>
```