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