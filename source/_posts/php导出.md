---
title: 业务提升技术之php导出
date: 2016-11-13 10:57:09
tags: 业务驱使技术
category: "业务驱使技术"
---

### A 背景
前段时间做需求，产品妹子让优化导出相关的功能，她说目前的导出每次只能导1000条，而且贼慢。
？？？
我打开看了下导出功能的代码(订单相关的导出)，大致逻辑是这样的，从库中拿1000条订单的数据，由于导出关联到很多数据，商品、sku、优惠券、积分，这些数据都在不同表中，挨个查呗，然后就出现了最省事却最坑爹的方法。。
```
foreach() {
    //sql查库
}
```
查这些数据确实需要多次查，好歹批量查一下啊，查询条件都是一条一条的，1000条数据要查不知道多少次，对数据库压力太大。
商家点击导出后，就开始等等等，有时候导出的脚本挂了，商家也不知道。

### B 重写
>最终要解决的问题是，提高导出上限、加快导出速度、减少DB压力

#### 第一阶段
1.梳理了导出最终结果的数据源，需要哪些表、哪些字段。
2.从导出的Excel改为导出csv（借助php的fputcsv）
3.数据批量读取

梳理完之后，开始搞，我将上限定为2万进行测试，将所有的数据批量获取，获取后开始foreach将导出的数据结构进行拼凑，然后一次性导出。
OK，测试，结果还是好慢啊，而且点击完就是等等等，只能放弃了。。

>此阶段总结：导出的最终变量数据太大，临时变量也没有及时unset，脚本内存占用太高。sql语句limit太大，越往后拖的越慢。

#### 第二阶段
边查边导
在第一阶段的基础上，重新优化
1.查询数据以某个排序字段进行排序，比如时间、主键id（注意，此处最好对排序的sql进行explain进行观察，选择合适的字段进行排序，有时候因为某个字段的排序会导致整个sql变慢好多倍，），第一次查询后新增where条件(例time>XX)，保持limit起始值一直为0
2.由于需要批量查询，每次构建的变量where条件在查询后，及时进行unset，减少脚本内存
3.边查边导，假设每次查询1000，导出1000，清空临时变量，脚本sleep 0.5秒，继续查继续导。

为了观察脚本的执行时间和内存使用情况，在脚本开始和结束地方添加memory_get_usage()观察内存使用情况和脚本耗时。

>此阶段总结：效率提升，脚本内存占用稳定，sql查询效率提高。，查一次导一次，就像我们下载东西一样，用户点击导出后，瞬间将结果输出到浏览器，提升用户体验

#### 第三阶段
细节优化
1.此次导出，查询关联数据，构建where条件时，得到的数组中有重复的元素，要进行唯一处理，往往我们最先想到的是array_unique，不过这里建议先使用array_flip，再用array_keys获取唯一数组，效率较高。(array_unique内部进行了快排，效率较低，具体函数的比较和相关源码自行查询~)
2.构建最终结果集时，关联数据往往需要通过foreach进行循环构建，举个例子，结果集数组arr1，关联数组arr2
```
foreach($arr1 as $k => $v) {

    foreach($arr2 as $k1 => $v1) {
        //循环进行判断
        if ($v['字段'] == $v1['字段']) {
            $arr1[$k]['name'] = $v1['name'];
            break;
        }
    }
}

```


上述过程双层for循环遍历判断，假设每个数组有1000个元素，最多需要100万次循环。
优化方法，先对关联数据做一次循环，将需要判断的字段作为每个元素的key，重新生成新的数组。再将原数组unset掉
```

$arr2 = array();
foreach($arr1 as $k => $v) {
    $arr2['$v['字段']'] = $v;
}
unset($arr1);

```

这时再次构建结果集，大致是这样的
```
foreach($arr1 as $k => $v) {
    if (isset($arr2[$v['字段']])) {
        $arr1[$k]['name'] = $arr2[$v['字段']]['name'];
    }
}
unset($arr2);
```
上述代码，通过isset进行判断，直接通过数组的key查找元素，大大减少了由双重循环造成的时间复杂度


以上~
### C 总结
此次导出需求引发出的问题，大致几点
1.数据的批量获取，减少查询DB的次数以及合理的sql语句，尤其针对limit较大这种情况
2.防止php脚本占用过大内存，及时unset内存较大的临时变量(对于一些正常的增删改查业务接口，可以不使用unset，等脚本结束后GC自动回收)
3.细节相关，array_unique效率低于array_flip+array_keys，多层for循环构建数据的优化方法

#### 小插曲
之前在测试环境测试的时候，脚本老是中断，导出的数据量总是不一致，当时真的懵逼了，脚本怎么莫名中断，结果上服务器看了下php-fpm执行超时时间为10S，哔了狗。


### 附导出例子
```php
<?php
abstract class ExportCSV{ // class start

    //抽象方法，标题，数据


    /** 获取导出的title
     * @return Array
     */
    abstract protected function getExportFields();

    /** 获取数据
     * @param  int $offset 偏移量
     * @param  int $limit  获取的记录条数
     * @return Array
     */
    abstract protected function getExportData();


    // 定义类属性
    protected $exportName = 'export.csv'; // 导出的文件名
    protected $delimiter = ',';           // 设置分隔符
    protected $enclosure = '"';           // 设置定界符
    protected $isIconv = true;              //编码转换
    protected $fp;
    protected $times = 0;               //输出次数
    protected $flush = 10;              //多少次flush


    /** 设置每次flush时的次数
     * @param int $limit 每次flush时的次数
     */
    public function setLimit($limit = 0)
    {
        if (is_numeric($limit) && $limit > 0) {
            $this->limit = $limit;
        }
        return $this;
    }


    /** 设置导出的文件名
     * @param String $filename 导出的文件名
     */
    public function setExportName($filename)
    {
        if (substr($filename, -4) == '.csv') {

            $filename = substr($filename, 0, -4);
        }
        $this->exportName = $filename.'.csv';

        return $this;
    }

    /**
     * 分隔符和环绕符
     * setControls
     * @param $delimiter
     * @param $enclosure
     * @return $this
     */
    public function setControls($delimiter, $enclosure)
    {
        $this->delimiter = $delimiter;
        $this->enclosure = $enclosure;
        return $this;
    }


    public function exportInit() {
        // 设置导出文件header
        $this->setHeader();
        //直接输出到浏览器
        $fp = fopen('php://output', 'w');

        $this->fp = $fp;
        // 获取导出的列名
        $title = $this->getExportFields();

        $title = array_map(array($this,'formatCSV'), $title);

        fputcsv($fp, $title, $this->delimiter, $this->enclosure);
    }


    /** 导出csv */
    public function export()
    {
        $this->times++;
        $fp = $this->fp;
        //获取导出的数据
        $data = $this->getExportData();

        foreach ($data as $k => $item) {

            if ($this->times == $this->flush) {
                //刷新输出buffer
                ob_flush();
                flush();
                $this->times = 0;
            }

            $item = array_map(array($this,'formatCSV'), $item);

            //导出数据
            fputcsv($fp, $item, $this->delimiter, $this->enclosure);

        }
    }


    /** 设置导出文件header */
    private function setHeader(){

        header('content-type:text/x-csv');

        $ua = $_SERVER['HTTP_USER_AGENT'];

        if (preg_match("/MSIE/", $ua)) {

            header('content-disposition:attachment; filename="'.rawurlencode($this->exportName).'"');

        } else if(preg_match("/Firefox/", $ua)) {

            header("content-disposition:attachment; filename*=\"utf8''".$this->exportName.'"');

        } else {
            header('content-disposition:attachment; filename="'.$this->exportName.'"');
        }
    }

    //编码转换
    private function formatCSV($str)
    {
        return $this->isIconv ? iconv('utf-8', 'gbk', $str) : $str;
    }


}

class CCExport extends ExportCSV{



    protected  $data;
    protected  $title;



    public function setExportData($data)
    {
        $this->data = $data;
        return $this;
    }

    public function setExportTitle($title)
    {
        $this->title = is_array($title) ? $title : explode(',',$title);
        return $this;
    }

 
    protected function getExportFields()
    {
        return $this->title;
    }

    /* 返回数据

    * @return Array
    */
    protected function getExportData(){
        return $this->data;
    }

}

    
    $title = 'aaa'; //设置title
    $fileName = 'bbb' //设置文件名
    $exportObj = new CCExport(); //实例化
    $exportObj->setExportName($fileName)->setExportTitle($title)->exportInit();
    /*
        todo 构建结果集$rlt
    
    */
    $exportObj->setExportData($rlt)->export();

```


