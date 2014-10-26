# Arel Inspection

## What is Arel

ActiveRelation -> Arel

Arel is a SQL AST manager for Ruby

Keywords:

1.  [AST][1]
2.  [SQL LANG][2]
3.  [Visitor Pattern][3]

[1]: http://en.wikipedia.org/wiki/Abstract_syntax_tree 'AST'
[2]: https://www.sqlite.org/lang_select.html 'SQL LANG'
[3]: http://en.wikipedia.org/wiki/Visitor_pattern 'Visitor Pattern'

## Usage of Arel
```ruby
require 'arel'
require 'active_record'

# establish a connection
ActiveRecord::Base.establish_connection(
  adapter:  'mysql2',
  database: 'test',
  host:     'localhost',
  username: 'root',
  password: '1024',
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
#   LIMIT 10
#   OFFSET 10
arel.to_sql
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
  LIMIT 10
  OFFSET 10
```

### The Arel-AST
![Arel-AST](https://github.com/dengqinghua/records/blob/master/arel_inspecting/arel.png)

### The ORIGIN DESIGN of AST:
- SelectStatement
![ORIGIN-SQL1](https://www.sqlite.org/images/syntax/simple-select-stmt.gif)
- SelectCore
![ORIGIN-SQL2](https://www.sqlite.org/images/syntax/select-core.gif)
