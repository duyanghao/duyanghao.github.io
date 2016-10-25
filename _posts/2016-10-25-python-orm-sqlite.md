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

```sh
[root@CentOS-64-duyanghao SQLite_operation]# python orm_sqlite.py
create sqlite and version table
final
```

第二次运行

```sh
[root@CentOS-64-duyanghao SQLite_operation]# python orm_sqlite.py
exist version table
final
```

```sh
#!/bin/sh 
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