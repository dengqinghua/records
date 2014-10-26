# Arel Inspection

[TOC]

## What is Arel

ActiveRelation -> Arel

Arel is a SQL AST manager for Ruby

Keywords:

1.  [AST](http://en.wikipedia.org/wiki/Abstract_syntax_tree)
2.  [Visitor Pattern](http://en.wikipedia.org/wiki/Visitor_pattern)

## Usage of Arel
### Using with ActiveRecord::Base.find\_by\_sql
```ruby
require 'arel'
require 'active_record'

# establish a connection
ActiveRecord::Base.establish_connection(
  adapter:  'mysql2',
  database: 'test',
  host:     'localhost',
  username: 'root',
  password: '1024'
)

# 'id'
id = Arel::SqlLiteral.new('id')

# "COUNT(id)"
id.count.to_sql

user = Arel::Table.new('user')

arel = user.
  project('id', 'user_name').
  where(user[:nick_name].eq('dsg')).
  order('created_at DESC').
  skip(10).
  take(5);

# SELECT  weight, hight FROM `products`
#   WHERE `products`.`name` = 'dsg'
#   ORDER BY created_at DESC
#   LIMIT 5
#   OFFSET 10
sql = arel.to_sql

User.find_by_sql(sql)
```

### Using with ActiveRecord::Base.where
```ruby
# ... Some codes are ommited

#  SELECT `users`.* FROM `users`
#     WHERE(
#       `users`.`id` < 10
#          AND `users`.`id` > 0
#          OR  `users`.`id` = 1024
#     )
User.where(
  t[:id].
    lt(10).
    and(t[:id].gt 0).
    or(t[:id].eq 1024)
)

```

## Arel-SQL Mapping
```ruby
File.write('arel.dot', arel.to_dot)
system %x(dot arel.dot -T png -o arel.png)
```
```SQL
SELECT  weight, hight FROM `products`
  WHERE `products`.`name` = 'dsg'
  ORDER BY created_at DESC
  LIMIT 5
  OFFSET 10
```

### The Arel-AST
![Arel-AST](https://github.com/dengqinghua/records/blob/master/arel_inspecting/arel.png)

### The ORIGIN DESIGN of AST:

- [SelectStatement](https://www.sqlite.org/syntax/select-stmt.html)

![ORIGIN-SQL1](https://www.sqlite.org/images/syntax/simple-select-stmt.gif)

- [SelectCore](https://www.sqlite.org/syntax/select-core.html)

![ORIGIN-SQL2](https://www.sqlite.org/images/syntax/select-core.gif)
