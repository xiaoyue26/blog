---
title: groovy语法相关
date: 2017-01-01 19:19:51
tags: 
- gradle 
- groovy
categories:
- gradle
---


```groovy
/**
 * Created by xiaoyue26 on 17/7/10.
 */
// 1
def getSomething() {

    "getSomething return value" //如果这是最后一行代码，则返回类型为String

    1000 //如果这是最后一行代码，则返回类型为Integer

}

println getSomething()
// 2
String getString() {
    return "I am a string"
}

println getString()

// 3 单双引号(同php)
def x = 1
def doubleQuoteWithDollar = "I am $x dolloar" //输出I am 1 dolloar
def singleQuote = 'I am $x dolloar'
println doubleQuoteWithDollar
println singleQuote

//4 数据类型
def int y = 1
println y.getClass()
println y.getClass().getCanonicalName()

// 5. 容器 List
def aList = [5, 'string', true] //List由[]定义，其元素可以是任何对象
println aList

assert aList[1] == 'string'
assert aList[5] == null //第6个元素为空
aList[100] = 100  //设置第101个元素的值为10
assert aList[100] == 100
println aList.size // 101

// 6. Map
def aMap = ['key1': 'value1', 'key2': true]
println aMap['key1']
def key1 = "wowo"
def aConfusedMap1 = [key1: "aConfusedMap1"] // 不转义
println aConfusedMap1.key1
def aConfusedMap2 = [(key1): "aConfusedMap2"]  // 转义
println aConfusedMap2.wowo
println aConfusedMap2['wowo']

//7. Range
def aRange = 1..5
println aRange
def aRangeWithoutEnd = 1..<5  // <==包含1,2,3,4这4个元素
println aRangeWithoutEnd.from // 潜规则调用了 getFrom 方法 (from其实是private的)
println aRangeWithoutEnd.to

// 8. 闭包
def aClosure = {
        //闭包是一段代码，所以需要用花括号括起来..
    String param1, int param2 ->  //这个箭头很关键。箭头前面是参数定义，箭头后面是代码
        println "this is code $param1,$param2" //这是代码，最后一句是返回值，
        //也可以使用return，和Groovy中普通函数一样
}


aClosure('hello', 1)
aClosure.call('hello', 1)

// 默认参数 $it
def greeting = { "Hello, $it!" }
assert greeting('Patrick') == 'Hello, Patrick!'

// 显式声明无参数:
def noParamClosure = { -> true }

// 9. 闭包进阶

def iamList = [1, 2, 3, 4, 5]  //定义一个List
iamList.each {  //调用它的each，这段代码的格式看不懂了吧？each是个函数，圆括号去哪了？
    println it
}
// each函数的声明:
// public static <T> List<T> each(List<T> self, Closure closure)
// 第一个参数是self,省略;最后一个参数是闭包,因此可以省略圆括号.
iamList.each({  //加上圆括号
    println it
})

// 例2 :
def testClosure(Closure closure) {
    //do something
    closure() //调用闭包
}

testClosure {
    println "i am in closure"
}

//例3: 闭包传参

def aaMap = [k1: 'value1', k2: true]
def results = aaMap.findAll {
    key, value ->
        println "scan : key=$key,value=$value"
        if (key == 'k1')
            return true
        return false
}
print results

// 10. 成员变量
// eg 1
thisX = 1 // 不加def和类型,则是实例变量, 会在对应类的 main 函数中 动态添加变量.
def printx() {
    println thisX
}

printx()
// eg 2
def smallX = 1  // 普通 main函数中的 局部变量
def printSmallx() { // 类的实例方法
    println smallX; // error
}
// printSmallx() // error
// eg 3
import groovy.transform.Field;

//必须要先import
@Field fieldX = 1 // <==在x前面加上@Field标注，这样，x就彻彻底底是test的成员变量了。
def printFieldX() {
    println fieldX
}

printFieldX()

// 11. IO操作
def srcFile = new File('/Users/xiaoyue26/input')
srcFile.eachLine {
    println it
}
// 写入
def targetFile = new File('/Users/xiaoyue26/output')
targetFile.withOutputStream {
    os ->
        srcFile.withInputStream {
            ins -> os << ins
        }
}





```

