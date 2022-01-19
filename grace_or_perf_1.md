# 优雅 VS 性能

## 背景
公司是做跨境电商，由于外部原因，公司对IT费用做了大的调整，把服务器配置对半砍，以前没有出性能问题的代码，由于服务器配置对半砍了以后，性能问题就出现了，也就有了这篇文章。服务器具体表现是，一台专门跑定时任务的服务器在某几个时间段CPU跑满。cpu负载图如：
[![7D5Mq0.png](https://s4.ax1x.com/2022/01/19/7D5Mq0.png)](https://imgtu.com/i/7D5Mq0)
定时任务的框架是用PHP+Swoole+Laravel部分组件。由于是定时任务，xhprof在此就派不上用场了，swoole tracer又收费。只好另想别的办法，根据zabbix的cpu监控负载图，cpu沾满的时段去找相应的定时任务，找到可疑的定时任务后，在到代码里面分块统计代码执行时长，然后根据代码执行时长，找出性能瓶颈点。

## 业务背景
根据统计结果找到了相应的代码，这块代码实现的功能是查找指定商品的sku别名。比如商品sku:apple-13pro-red-64，在我们其它业务系统中可能叫apple-13p-red-64或apple-13pro-red，在匹配库存和价格的时候，就需要用apple-13pro-red-64,apple-13p-red-64,apple-13pro-red三个sku去匹配。一个sku对应多个别名，一个别名对应一个sku。

## 代码
数据表sku_alias的结构如下:

| 字段名称 | 数据类型 | 备注 |
| ------   | ------ | ------ | 
| sku | varchar | sku |
| alias |  varchar | sku对应的别名 |

``` php
private function getSKuAlias(string $sku): Collection
{
    if (is_null($this->allSkuAlias)) {
        $this->allSkuAlias = DB::table('sku_alias')->get();
    }

    $alias = $this->allSkuAlias->where('sku', $sku)->pluck('alias');
    $alias = $alias->merge($this->allSkuAlias->where('alias', $sku)->pluck('sku'));
    return $alias->push($sku)->toArray();
}
```
解释一下上面的代码，查询sku_alias数据表，将数据表的查询结果给allSkuAlias属性，然后去对allSkuAlias查找sku或别名等于参数sku的数据。然后再将查找结果返回。sku_alias的数表一共714条数据。服务器配置没有被砍的时候，可以掩盖此段代码的性能问题，当服务器配置被砍，性能问题就暴露了，因为getSKuAlias的时间复杂度是O(N²)，一共需要进行1428次查找，更为糟糕的是getSKuAlias方法还会被循环调用。cpu 100%也就是不足为怪了。为什么getSKuAlias的时间复杂度是O(N²)，熟悉Laravel的朋友都知道
```php
DB::table('sku_alias')->get(); //这里返回的是Collection对象
```
而Collection的where方法最终是用php array_filter函数实现，而array_filter的时间复杂度是O(N)。既然知道了原因，优化方向就有了，把时间复杂度O(N²)优化成O(1)，第一时间想到的是用Hash数组（索引数组，其它语言里面叫map或dict）。由于一个sku会有多个别名，所以需要两个Hash数组，在进行查找sku对应别名时需要一个一对多的Hash数组。在进行别名对应sku的查找时，需要定义一对一的Hash数组，以刚才apple-13pro-red-64的举例，两个Hash数组定义如下:

```php
[
    'apple-13p-red-64' => [
        'apple-13p-red-64',
        'apple-13pro-red',
    ],
] //sku对应的别名

[
    'apple-13p-red-64' => 'apple-13p-red-64',
    'apple-13pro-red' => 'apple-13p-red-64',
] //别名对应sku
```

优化后的最终代码:
```php
private function getSKuAlias(string $sku): array
{
    if ($this->allSkuAlias === null) {
        $source =  DB::table('sku_alias')
            ->get(['sku', 'alias']);
        $this->allSkuAlias = $source->groupBy('sku')
            ->map(fn($alias) => $alias->pluck('alias')->toArray())
            ->toArray();
        $this->allAliasSku = $source->pluck('sku', 'alias')
            ->toArray();
    }
    
    $alias = [$sku];
    if (isset($this->allSkuAlias[$sku])) {
        array_push($alias, ...$this->allSkuAlias[$sku]);
    }

    if (isset($this->allAliasSku[$sku])) {
        $alias[] = $this->allAliasSku[$sku];
    }

    return array_unique($alias);
}
```

## 感悟
优化前的代码用Collection的where和pluck方法，代码行数少，6行代码就实现功能。优化后的代码行数增加，优化前时间复杂度是O(N²)，优化后近乎O(1)。性能方面的差距只有自己去细品。如果不是服务器配置被砍半，这个问题也就不会出现。特别是当下服务器的cpu的性能来说，要不是穷，谁会去做这种优化。日常开发的时候，也不会注意到性能方面的差异。所以选择性能还是优雅？
