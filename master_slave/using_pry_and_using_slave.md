主从库分离设计
==============

该文档涵盖了 MasterSlave gem的设计思路.

阅读完该文档后，您将会了解到:

* 如何使用 Pry 阅读 Rails 源代码.
* ActiveRecord::Relation源码分析.
* MasterSlave gem的设计.

--------------------------------------------------------------------------------

Using Pry
---------
[Pry]()是一个很有用的读源码的工具, 用过的人都说好!

安装pry:

```shell
gem install pry
```

### show-method

1. 查看实例方法

```ruby
show-method String#blank?

# =>
From: /home/dengqinghua/.rvm/gems/ruby-2.0.0-p481@somewhere_zhaoshang/gems/activesupport-4.1.1/lib/active_support/core_ext/object/blank.rb @ line 116:
Owner: String
Visibility: public
Number of lines: 3

def blank?
  BLANK_RE === self
end
```

2. 查看类方法

```ruby
show-method ActiveRecord::Base.where

From: /home/dengqinghua/.rvm/gems/ruby-2.0.0-p481@somewhere_zhaoshang/gems/activerecord-4.1.1/lib/active_record/querying.rb @ line 10:
Owner: ActiveRecord::Querying
Visibility: public
Number of lines: 3

delegate :where, to: :all
```

3. 查看动态方法

```ruby
show-method TbShop#status_name

From: /home/dengqinghua/workspace/somewhere_zhaoshang/app/models/concerns/status_able.rb @ line 116:
Owner: TbShop
Visibility: public
Number of lines: 3

define_method "#{status_name}_name".to_sym do
  self.send("#{status_name}_config")[:key]
end
```

4. 查看某个类的所有方法

```ruby
show-method ActiveRecord

From: /home/dengqinghua/.rvm/gems/ruby-2.0.0-p481@somewhere_zhaoshang/gems/activerecord-4.1.1/lib/active_record/gem_version.rb @ line 1:
Module name: ActiveRecord
Number of monkeypatches: 3. Use the `-a` option to display all available monkeypatches
Number of lines: 15

module ActiveRecord
  # Returns the version of the currently loaded ActiveRecord as a <tt>Gem::Version</tt>
  def self.gem_version
    Gem::Version.new VERSION::STRING
  end

  module VERSION
    MAJOR = 4
    MINOR = 1
    TINY  = 1
    PRE   = nil

    STRING = [MAJOR, MINOR, TINY, PRE].compact.join(".")
  end
end
```

5. 查看当前self的方法

```ruby
show-method
```

### ls
查看当前的self中存有哪些变量和方法, 和Linux类似

```ruby
ls
```

```ruby
ls Deal
```

### cd
通过cd 我们可以`进入`任何一个对象, 如同 instance\_eval 一样

```ruby
cd Deal
show-method
ls
```

```ruby
cd Deal.last
show-method
ls
```

### edit
edit可以直接帮你打开某个方法所在的文件

```ruby
show-method ActiveRecord::Base.establish_connection

From: /home/dengqinghua/.rvm/gems/ruby-2.0.0-p481@somewhere_zhaoshang/gems/activerecord-4.1.1/lib/active_record/connection_handling.rb @ line 47:
Owner: ActiveRecord::ConnectionHandling
Visibility: public
Number of lines: 13

def establish_connection(spec = nil)
  spec     ||= DEFAULT_ENV.call.to_sym
  resolver =   ConnectionAdapters::ConnectionSpecification::Resolver.new configurations

  spec     =   resolver.spec(spec)

  unless respond_to?(spec.adapter_method)
    raise AdapterNotFound, "database configuration specifies nonexistent #{spec.config[:adapter]} adapter"
  end

  remove_connection
  connection_handler.establish_connection self, spec
end

edit ActiveRecord::Base.establish_connection
edit /home/dengqinghua/.rvm/gems/ruby-2.0.0-p481@somewhere_zhaoshang/gems/activerecord-4.1.1/lib/active_record/connection_handling.rb
```

ActiveRecord::Relation
----------------------
### ActiveRecord::Base.where

```ruby
show-method ActiveRecord::Base.where
# =>

From: /home/dengqinghua/.rvm/gems/ruby-2.0.0-p481@somewhere_zhaoshang/gems/activerecord-4.1.1/lib/active_record/querying.rb @ line 10:
Owner: ActiveRecord::Querying
Visibility: public
Number of lines: 3

delegate :select, :group, :order, :except, :reorder, :limit, :offset, :joins,
         :where, :rewhere, :preload, :eager_load, :includes, :from, :lock, :readonly,
         :having, :create_with, :uniq, :distinct, :references, :none, :unscope, to: :scoped
```

`ActiveRecord::Base.scoped`

```ruby
show-method ActiveRecord::Base.scoped

From: /home/dengqinghua/.rvm/gems/ruby-2.1.1@somewhere/gems/activerecord-3.2.13/lib/active_record/scoping/named.rb @ line 30:
Owner: ActiveRecord::Scoping::Named::ClassMethods
Visibility: public
Number of lines: 13

def scoped(options = nil)
  if options
    scoped.apply_finder_options(options)
  else
    if current_scope
      current_scope.clone
    else
      scope = relation
      scope.default_scoped = true
      scope
    end
  end
end
```

`ActiveRecord::Base.relation`

```ruby
show-method ActiveRecord::Base.relation

From: /home/dengqinghua/.rvm/gems/ruby-2.1.1@somewhere/gems/activerecord-3.2.13/lib/active_record/base.rb @ line 452:
Owner: #<Class:ActiveRecord::Base>
Visibility: private
Number of lines: 9

def relation #:nodoc:
  relation = Relation.new(self, arel_table)

  if finder_needs_type_condition?
    relation.where(type_condition).create_with(inheritance_column.to_sym => sti_name)
  else
    relation
  end
end
```

```ruby
show-method ActiveRecord::Relation#where

From: /home/dengqinghua/.rvm/gems/ruby-2.1.1@somewhere/gems/activerecord-3.2.13/lib/active_record/relation/query_methods.rb @ line 132:
Owner: ActiveRecord::QueryMethods
Visibility: public
Number of lines: 7

def where(opts, *rest)
  return self if opts.blank?

  relation = clone
  relation.where_values += build_where(opts, rest)
  relation
end
```

故可以得到, `ActiveRecord::Base.where`方法代理到`ActiveRecord::Base.relation`的where方法, 即最后调用的是
是ActiveRecord::Relation#where方法

### When query the database?
In console, every time we use `Role.where`, the console calls

```ruby
Role.where.inspect
```

```ruby
show-method ActiveRecord::Relation#inspect

From: /home/dengqinghua/.rvm/gems/ruby-2.1.1@somewhere/gems/activerecord-3.2.13/lib/active_record/relation.rb @ line 497:
Owner: ActiveRecord::Relation
Visibility: public
Number of lines: 3

def inspect
  to_a.inspect
end

show-method ActiveRecord::Relation#to_a

From: /home/dengqinghua/.rvm/gems/ruby-2.1.1@somewhere/gems/activerecord-3.2.13/lib/active_record/relation.rb @ line 150:
Owner: ActiveRecord::Relation
Visibility: public
Number of lines: 13

def to_a
  # We monitor here the entire execution rather than individual SELECTs
  # because from the point of view of the user fetching the records of a
  # relation is a single unit of work. You want to know if this call takes
  # too long, not if the individual queries take too long.
  #
  # It could be the case that none of the queries involved surpass the
  # threshold, and at the same time the sum of them all does. The user
  # should get a query plan logged in that case.
  logging_query_plan do
    exec_queries
  end
end
```

NOTE: 通过注释可以看到, 最后Relation真正的查询都是通过调用`ActiveRecord::Relation#to_a`方法, `to_a`方法返回的是数组,
每一个relation最后通过查询都将返回一个数组.
可以猜测, `ActiveRecord::Relation#each`方法, 调用的是`ActiveRecord::Relation#to_a.each`

```ruby
show-method ActiveRecord::Relation#each

From: /home/dengqinghua/.rvm/gems/ruby-2.1.1@somewhere/gems/activerecord-3.2.13/lib/active_record/relation/delegation.rb @ line 5:
Owner: ActiveRecord::Delegation
Visibility: public
Number of lines: 1

delegate :to_xml, :to_yaml, :length, :collect, :map, :each, :all?, :include?, :to_ary, :to => :to_a
```

INFO: 像 each, map 这种常用的方法, rails用了代理的方法, 代理到 to_a 上,
但是Array的其他方法呢? 如 Array#uniq等, rails是如何处理的?

```ruby
# in ActiveRecord::Delegation
def method_missing(method, *args, &block)
  if @klass.respond_to?(method)
    ::ActiveRecord::Delegation.delegate_to_scoped_klass(method)
    scoping { @klass.send(method, *args, &block) }
  elsif Array.method_defined?(method)
    ::ActiveRecord::Delegation.delegate method, :to => :to_a
    to_a.send(method, *args, &block)
  elsif arel.respond_to?(method)
    ::ActiveRecord::Delegation.delegate method, :to => :arel
    arel.send(method, *args, &block)
  else
    super
  end
end
```

```ruby
show-method ActiveRecord::Relation
```

### ActiveRecord::Relation总结
1. `ActiveRecord::Base` 通过 scoped 方法, 将where, find等方法代理到`ActiveRecord::Relation`中
2. 任何一个Relation查询, 最后真正执行的时候都是通过`ActiveRecord::Relation#to_a`方法进行的
3. 如果未调用任何方法, `User.where(id: 1)` 将不会执行查询. 但是如果调用了非Relation自身的方法时,
Rails会将这个方法传递给 `method_missing`, 进行二次分发, 此时如果能够找到对应的方法, 则再次调用
`to_a`方法进行最终的查询.


### More about ActiveRecord::Relation#to\_a

```ruby
def to_a
  # We monitor here the entire execution rather than individual SELECTs
  # because from the point of view of the user fetching the records of a
  # relation is a single unit of work. You want to know if this call takes
  # too long, not if the individual queries take too long.
  #
  # It could be the case that none of the queries involved surpass the
  # threshold, and at the same time the sum of them all does. The user
  # should get a query plan logged in that case.
  logging_query_plan do
    exec_queries
  end
end
```

INFO: 如果对上述的`exec_queries`继续pry, 则可以知道最后的查询调用了`ActiveRecord::Base.find_by_sql`方法.

```ruby
def exec_queries
  return @records if loaded?

  default_scoped = with_default_scope

  if default_scoped.equal?(self)
    @records = if @readonly_value.nil? && !@klass.locking_enabled?
      eager_loading? ? find_with_associations : @klass.find_by_sql(arel, @bind_values)
    else
      IdentityMap.without do
        eager_loading? ? find_with_associations : @klass.find_by_sql(arel, @bind_values)
      end
    end

    preload = @preload_values
    preload +=  @includes_values unless eager_loading?
    preload.each do |associations|
      ActiveRecord::Associations::Preloader.new(@records, associations).run
    end

    # @readonly_value is true only if set explicitly. @implicit_readonly is true if there
    # are JOINS and no explicit SELECT.
    readonly = @readonly_value.nil? ? @implicit_readonly : @readonly_value
    @records.each { |record| record.readonly! } if readonly
  else
    @records = default_scoped.to_a
  end

  @loaded = true
  @records
end
private :exec_queries
```

```ruby
def find_by_sql(sql, binds = [])
  logging_query_plan do
    connection.select_all(sanitize_sql(sql), "#{name} Load", binds).collect! { |record| instantiate(record) }
  end
end
```

NOTE: 结合上次讲解的 Arel 的gem, 我们可以进一步总结ActiveRecord::Base的查询步骤
1. 利用ActiveRecord::Relation, 将所有的条件传入Arel
2. Arel将所有的sql组装, 获取最后的sql语句
3. 调用 `find_by_sql` 方法, 建立数据库连接, 并将第二步的sql传入, 获取数据库的数据

MasterSlave概述
---------------
### 什么是MasterSlave
MasterSlave是一个gem, 该gem将主库和从库进行了分离, 这样便可以通过将耗时的查询放入从库中, 提供系统的整体的性能.

NOTE: 该gem在zhaoshang\_cpc项目下, 目录为: `vendor/engines/master_slave`

### 如何配置从库

```shell
cd 你的zhaoshang_cpc项目目录
cp config/config_files/bj_dev/shards.yml config/
```

在 setting.local.yml 文件中, 可以添加从库开关

```ruby
using_slave: false
```

### MasterSlave的使用

```ruby
# in shards.yml
development:
  somewhere_slave:
    adapter: mysql2
    encoding: utf8
    reconnect: false
    database: dsg
    pool: 5
    username: dashuaige
    password: dsgv587
    host: 127.0.0.1
    port: 13307
```

从库设置如上所示, 其中`somewhere_slave`为slave的名称

我们在项目中应该使用的从库为: **somewhere_slave**. 如果你的代码中使用了很复杂的查询, 可以显式地调用从库, 如:

```ruby
TbShop.using_slave(:somewhere_slave).
  where(nick_name: 'DaShuaiGe').
  where(id: 1024)
# Or
TbShop.where(nick_name: 'DSG').
  order('created_at DESC').
  using_slave(:somewhere_slave)
```

NOTE: ActiveRecord::Base.using_slave 方法可以添加在所有的relation后面,
和where使用方法一样.

### MasterSlave的设计思路
#### 主库连接
在database.yml文件中的数据库为**主库连接**, 在设计从库连接时,
我们先需要看主库是如何连接的:

![active\_record\_connection](https://github.com/dengqinghua/records/blob/master/master_slave/active_record_connection.png)

NOTE: 在这里的连接还有几个细节需要注意一下:
1. 我们在项目启动的时候就建立mysql连接
2. 在第一步建立的连接是可以重复利用的, 该连接会存在连接池中

#### 从库连接
在设计从库连接时, 思路和主库连接即为相似

在shards.yml文件中的数据库为**从库连接**, 从库连接设计如下:

![master\_connection](https://github.com/dengqinghua/records/blob/master/master_slave/master_slave.png)

NOTE: 在这里的连接还有几个细节需要注意一下:
1. 我们也需要在项目启动的时候就建立mysql连接
2. 从库连接也应该是可重用的

MasterSlave的代码设计
--------------------

### 项目启动时建立连接
In Rails, 建立连接时调用的是`ActiveRecord::Base.establish_connection`方法.

```ruby
# ActiveRecord::Base.establish_connection
#
# From: /home/dengqinghua/.rvm/gems/ruby-2.0.0-p481@somewhere_zhaoshang/gems/activerecord-4.1.1/lib/active_record/connection_handling.rb @ line 47:
# Owner: ActiveRecord::ConnectionHandling
# Visibility: public
# Number of lines: 13

def establish_connection(spec = nil)
  spec ||= DEFAULT_ENV.call.to_sym
  resolver = ConnectionAdapters::ConnectionSpecification::Resolver.new configurations
  spec = resolver.spec(spec)

  unless respond_to?(spec.adapter_method)
    raise AdapterNotFound, "database configuration specifies nonexistent #{spec.config[:adapter]} adapter"
  end

  remove_connection
  connection_handler.establish_connection self, spec
end
```

In MasterSlave

在建立从库连接的时候, 模仿了上述方法的思路, 并利用Rails自己的方法建立了连接
仅仅是传入的配置文件不同.

下为 `MasterSlave::ConnectionHandling.establish_connection` 方法

```ruby
##
# ==== Description
#   启动的时候, 根据配置文件设置的从库, 建立连接
#
#   该设计参考自activerecord/lib/active_record/connection_handling.rb
# 中的ActiveRecord::Base.establish_connection方法.
#
# ==== WARNING
#   请在参考Rails源码的时候注意Rails的版本: 4.1.1
#
def self.establish_connection
  unless Rails.env.inquiry.test?
    ActiveRecord::Base.slave_connection_names ||= []

    MasterSlave.configurations.slaves.each do |slave_name, config|
      proxy = Proxy.new(slave_name)

      adapter_method = "#{config[:adapter]}_connection"
      spec = ActiveRecord::ConnectionAdapters::
        ConnectionSpecification.new(config, adapter_method)

      ActiveRecord::Base.slave_connection_names << proxy.name
      ActiveRecord::Base.remove_connection(proxy)
      ActiveRecord::Base.connection_handler.establish_connection(proxy, spec)
    end
  end
rescue ActiveRecord::AdapterNotSpecified => ex
  raise 'sharding.yml文件配置错误, 请参考config/config_file/bj_dev/sharding.yml文件'
end
```

NOTE:
spec: 配置文件信息, 即shards.yml文件的信息
proxy: 从库的名称, 如 somewhere_slave 等
ActiveRecord::Base.remove_connection: 在建立新的连接之前, 需要现将现有的同名字的连接移除.
ActiveRecord::Base.connection_handler.establish_connection(proxy, spec): 将`配置文件信息`, `连接名`作为参数, 利用Rails的方法建立连接.

将连接添加到Rails启动项中

```ruby
module MasterSlave
  class Railtie < Rails::Railtie
    config.after_initialize do
      if Settings.using_slave
        if File.exist?(MasterSlave.config_file)
          Rails.logger.info "\033[32mmaster_slave is on!\033[0m"
          MasterSlave::ConnectionHandling.establish_connection
        else
          Rails.logger.error "\033[31mNo such file #{MasterSlave::Configuration.config_file}\033[0m"
          Rails.logger.error "\033[31mPlease execute `rails g master_slave:config`\033[0m"
          Rails.logger.error "\033[31mmaster_slave is off!\033[0m"
        end
      end
    end

    generators do
      require File.expand_path('../../rails/generators/config_generator', __FILE__)
    end
  end
end
```

### 添加trigger: 'using\_slave'
NOTE: trigger为枪柄, pull the trigger 即为开枪的意思, 这里的trigger可以理解为从库的触发器.

```ruby
User.using_slave(:somewhere_slave).where(nick_name: 'DSG')
```

添加 `ActiveRecord::Base.using_slave` 方法, 该方法

1. 获取当前的 从库名称: somewhere_slave, 可通过该名称获取 从库连接
2. 该方法的返回值为`<#ActiveRecord::Relation>`

```ruby
class ActiveRecord::Base
  class << self
    delegate :using_slave, to: :scoped
  end
end

class ActiveRecord::Relation
  def using_slave(slave_name)
    if !Rails.env.inquiry.test? && Settings.using_slave
      @using_slave = true
      relation     = clone
      slave_name   = slave_name.to_s.strip

      if ActiveRecord::Base.slave_connection_names.include?(slave_name)
        MasterSlave::RuntimeRegistry.current_slave_name = slave_name

        relation
      else
        raise "#{slave_name} not exist"
      end
    else
      clone
    end
  end
end
```

### 使用`Module#prepend`复写ActiveRecord::Base.connection和ActiveRecord::Relation#to\_a方法
#### prepend in ruby 2.0

```ruby
class DSG
  def name
    'Da Shuai Ge'
  end
end

DSG.new.name
#=> Da Shuai Ge
```

要求: `DSG.new.name` 输出 'Yo!, my name is Da Shuai Ge'

- alias method chain

```ruby
# before ruby 2.0
class DSG
alias old_name name
alias name name_with_yo!

def name_with_yo!
  "Yo!, my name is #{old_name}"
end
```

- prepend

```ruby
module YoName
  def name
    "Yo!, my name is #{super}"
  end
end

#DSG.send(:prepend, YoName)

class DSG
  prepend YoName
end

DSG.new.name #=> 'Yo!, my name is Da Shuai Ge'
DSG.ancestors == [YoName, DSG, Object, Kernel, BasicObject]
```

如何触发从库连接? 可以对比主库的连接流程

![master\_slave\_relation](https://github.com/dengqinghua/records/blob/master/master_slave/master_slave_relation.png)

在这个过程中最重要的为`ActiveRecord::Relation#to_a`和`ActiveRecord::Base.connection`方法

NOTE:
1. `ActiveRecord::Relation#using_slave`:
  获取从库名称, 如'dsg', 并打开从库的设置开关
2. `ActiveRecord::Relation#to_a`:
  检查是否调用`using_slave`, 如果有, 则设置一个和从库名'dsg'的全局变量
3. `ActiveRecord::Base.connection`:
  检查是否有和`dsg`相关的全局变量存在,
  检查从库的设置开关是否打开
    如果都满足, 则去获取**从库连接**,
    否则,       则去获取**主库连接**

故应该写的方法为:

- `ActiveRecord::Relation#using_slave`
- `ActiveRecord::Relation#to_a`
- `ActiveRecord::Base.connection`

```ruby
module MasterSlave
  module Relation
    SOME_ZHAOSHANG_RAILS_VERSION = '4.1.1'

    ##
    # ==== Description
    #   添加从库的scope
    #
    # ==== Examples
    #
    #   Deal.using_slave(:somewhere_slave).where(id: 2)
    #   Deal.where(id: 1024).using_slave(:somewhere_slave)
    #
    # ==== Returns
    #   <#ActiveRecord::Relation>|Raise Exception
    #
    def using_slave(slave_name)
      if !Rails.env.inquiry.test? && Settings.using_slave
        @using_slave = true
        relation     = clone
        slave_name   = slave_name.to_s.strip

        if ActiveRecord::Base.slave_connection_names.include?(slave_name)
          MasterSlave::RuntimeRegistry.current_slave_name = slave_name

          relation
        else
          raise "#{slave_name} not exist"
        end
      else
        clone
      end
    end

    def to_a
      using_master_database unless @using_slave

      if Rails.version == SOME_ZHAOSHANG_RAILS_VERSION
        super
      else
        raise "current Rails Version: #{Rails.version} is not supported"
      end
    ensure
      using_master_database
    end if defined?(Rails)

    private
    def using_master_database
      MasterSlave::RuntimeRegistry.current_slave_name = nil
    end
  end
end

ActiveRecord::Relation.send(:prepend, MasterSlave::Relation)
```

```ruby
module Connection
  def self.prepended(base)
    class << base
      prepend ClassMethods
    end
  end

  module ClassMethods
    ##
    # ==== Description
    #   在这里用了 Module#prepend 的方式, 重写了
    #
    #     ActiveRecord::Base.connection
    #
    #   方法, 最后ActiveRecord::Base.connection调用的是下面这个方法
    #
    #   请见 http://www.justinweiss.com/blog/2014/09/08/rails-5-module-number-prepend-and-the-end-of-alias-method-chain/
    #
    #   该方法会去当前的运行时线程中取 current_slave_name, 如果能够取到,
    # 则说明曾经建立过从库的连接, 再调用
    #
    #   ActiveRecord::Base.connection_handler.retrieve_connection(proxy)
    #
    # 方法来重新获取该连接.
    #
    #   如果找不到 current_slave_name, 则直接连主库.
    #
    def connection
      if Settings.using_slave && confirmed_to_use_master_slave?
        slave_name = MasterSlave::RuntimeRegistry.current_slave_name
        proxy      = MasterSlave::Proxy.new(slave_name)

        ActiveRecord::Base.connection_handler.retrieve_connection(proxy)
      else
        super
      end
    end

    private
    def confirmed_to_use_master_slave?
      slave_name = MasterSlave::RuntimeRegistry.current_slave_name
      slave_name && slave_connection_names.include?(slave_name)
    end
  end
end

ActiveRecord::Base.send(:prepend, MasterSlave::Connection)
```

TODO
----
- 支持多个slave, 且在有一个slave宕掉的时候, 可以自动连接其他slave
