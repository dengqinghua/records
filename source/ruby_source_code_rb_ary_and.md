Ruby Array Set Intersection
===========================

本文档主要涉及以下内容：

* Array#& 介绍
* `&`操作的源码分析

--------------------------------------------------------------------------------

背景
----

过滤参数

```ruby
params        = [1,2,3]
validate_keys = [1,3]
expect(some_method(params, validate_keys)).to eql([1, 3])
```

查找文档
--------

如果params是一个Hash对象，可以通过slice方法对params进行过滤
Array中是否有一个类似slice的方法呢？

```ruby
    [1, 0, 2, 4].slice(0) # => 1
```

&操作
-----

```ruby
    [8, 1, 0, 2, 4, 7] & [9, 1, 0, 2, 4, 6] # => [1, 0, 2, 4]
```

&源码分析 [rb_ary_and][1]
--------------------

```C
 static VALUE
rb_ary_and(VALUE ary1, VALUE ary2)
{
    VALUE hash, ary3, v;
    st_table *table;
    st_data_t vv;
    long i;

    ary2 = to_ary(ary2);
    ary3 = rb_ary_new();
    if (RARRAY_LEN(ary2) == 0) return ary3;
    hash = ary_make_hash(ary2);
    table = rb_hash_tbl_raw(hash);

    for (i=0; i<RARRAY_LEN(ary1); i++) {
        v = RARRAY_AREF(ary1, i);
        vv = (st_data_t)v;
        if (st_delete(table, &vv, 0)) {
            rb_ary_push(ary3, v);
        }
    }
    ary_recycle_hash(hash);

    return ary3;
}
```

算法分析
--------
* ary1为[8, 1, 0, 2, 4, 7]
* ary2为[9, 1, 0, 2, 4, 6]
* 新建一个数组ary3,作为返回值
* 新建一个Hash表,存储ary2的值，以及值的地址
* 对ary1中的元素做循环, 如果能够找到Hash表找到ary1的元素
便将该元素从表中删除，并将该元素push值ary3

讨论
----
* 如何优化算法?
* 是否有其他更高效的方式?

[1]: http://rxr.whitequark.org/mri/source/array.c#3865
