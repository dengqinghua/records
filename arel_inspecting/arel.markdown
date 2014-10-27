# Arel Inspection
## What is Arel

ActiveRelation -> Arel

Arel is a SQL AST manager for Ruby.

Keywords:

1. [AST](http://en.wikipedia.org/wiki/Abstract_syntax_tree)
2. [Visitor Pattern](http://en.wikipedia.org/wiki/Visitor_pattern)

## How
### How to use Arel
#### Using with `ActiveRecord::Base.find_by_sql`
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

user = Arel::Table.new('users')

arel = user.
  project('id', 'user_name').
  where(user[:nick_name].eq('dsg')).
  order('created_at DESC').
  skip(10).
  take(5);

# SELECT  id, user_name FROM `users`
#   WHERE `user`.`nick_name` = 'dsg'
#   ORDER BY created_at DESC
#   LIMIT 5
#   OFFSET 10
sql = arel.to_sql

User.find_by_sql(sql)
```

#### Using with `ActiveRecord::Base.where`
```ruby
# ... Some codes are ommited
#
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

#### Arel-SQL Mapping
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

#### The ORIGIN DESIGN of AST:

- [SelectStatement](https://www.sqlite.org/syntax/select-stmt.html)

![ORIGIN-SQL1](https://www.sqlite.org/images/syntax/simple-select-stmt.gif)

- [SelectCore](https://www.sqlite.org/syntax/select-core.html)

![ORIGIN-SQL2](https://www.sqlite.org/images/syntax/select-core.gif)

#### The Arel-AST
![Arel-AST](https://github.com/dengqinghua/records/blob/master/arel_inspecting/arel.png)

### How Arel works?
```ruby
id    = Arel::SqlLiteral.new('id')
count = id.count
count.to_sql
```
#### Inspecting to\_sql
```ruby
def to_sql engine = Table.engine
  engine.connection.visitor.accept self
end
```

Arel needs a engine: *ActiveRecord::Base*
```ruby
Arel::Table.engine
```

When connectioned, the connection has a visitor
```ruby
connection = ActiveRecord::Base.connection
visitor    = connection.visitor
```

Visitor can accept query object
```ruby
visitor.accept(id.count)
```

Where is the method `visitor.accept`?
```ruby
visitor.ancestors
[
  ActiveRecord::ConnectionAdapters::AbstractMysqlAdapter::BindSubstitution,
  Arel::Visitors::BindVisitor,
  Arel::Visitors::MySQL,
  Arel::Visitors::ToSql,
  Arel::Visitors::Visitor,
  Object,
  JSON::Ext::Generator::GeneratorMethods::Object,
  ActiveSupport::Dependencies::Loadable,
  PP::ObjectMixin,
  Kernel,
  BasicObject
]

# Arel::Visitors::Visitor
def accept object
  visit object
end
```

Simply combine some strings!
```ruby
  # in Arel::Visitors::ToSql
  visitor.visit_Arel_Nodes_Count(id.count)

  def visit_Arel_Nodes_Count o
    "COUNT(#{o.distinct ? 'DISTINCT ' : ''}#{o.expressions.map { |x|
    visit x
    }.join(', ')})#{o.alias ? " AS #{visit o.alias}" : ''}"
end

  visitor.visit_Arel_SqlLiteral(id)
```

Go through complexer sql
```ruby
user = Arel::Table.new('users')

arel = user.
  project('id', 'user_name').
  where(user[:nick_name].eq('dsg')).
  order('created_at DESC').
  skip(10).
  take(5);

arel.to_sql
```

### Conclusion
  1. Arel uses `to_sql` to get the sql.
  2. Arel didn't really connect to sql server, she only does the `join sql string` thing.
  3. All `to_sql` comes to visit\_Arel\_Nodes\_XXX, that is, Arel will visit each node of the AST.

## Why
### Why use AST
```ruby
class NewArel
  attr_accessor :where, :select, :order, :skip, :limit

  def where(string)
    @wheres ||= []
    @wheres << string

    self
  end

  def select(*args)
    @selects ||= []
    @selects = @selects.concat(args).compact.uniq
  end

  def order(string)
    @orders ||= []
    @orders << string
  end

  # ... omited

  def to_sql
    [
      "SELECT #{@selects.join(', ')}",
      "WHERE #{@where.join('AND ')}"
    ].join(' ')
  end
end

arel = NewArel.new.
  where('id < 10').
  where('id > 5').
  select(:id, :user_name)
arel.to_sql
```
### Why use Visitor Pattern?
