原文：http://blog.datumbox.com/drilling-into-sparks-als-recommendation-algorithm/

作者：Vasilis Vryniotis

翻译：[@TinyHaHa](https://github.com/TinyHaHa)

# Machine Learning Blog & Software Development News

![image](https://github.com/TinyHaHa/news/blob/master/img-storage/2019/11/22/01.png)

Hu等人介绍的ALS算法是一种非常受欢迎的技术，用于推荐系统问题，尤其是当我们具有隐式数据集（例如点击次数，点赞次数等）时。
它可以很好地处理大量数据，我们可以在各种机器学习框架中找到许多良好的实现。
Spark将算法包含在MLlib组件中，该组件最近经过重构以提高代码的可读性和体系结构。

Spark的实现要求Item和User ID的数字必须在整数范围内（整数类型或Long可以在整数范围内），这是合理的，因为这可以帮助加快操作速度并减少内存消耗。
我在阅读代码时注意到的一件事是，这些id列在fit / predict方法开始时被转换为Doubles，然后转换为Integers。 
这似乎有点骇人听闻，而且我已经看到它给垃圾收集器带来了不必要的压力。 这是ALS代码上将id转换为double的行：

![image](https://github.com/TinyHaHa/news/blob/master/img-storage/2019/11/22/02.png)

要理解为什么这样做，需要阅读checkedCast（）：
![image](https://github.com/TinyHaHa/news/blob/master/img-storage/2019/11/22/03.png)

此UDF接收Double并检查其范围，然后将其强制转换为整数。 此UDF用于架构验证。 问题是我们可以不使用丑陋的双铸件来实现这一目标吗？ 我相信是的：

>protected val checkedCast = udf { (n: Any) =>
>
>  n match {
>
>    case v: Int => v // Avoid unnecessary casting
>
>    case v: Number =>
>
>      val intV = v.intValue()
>
>      // True for Byte/Short, Long within the Int range and Double/Float with no fractional part.
>
>      if (v.doubleValue == intV) {
>
>        intV
>
>      }
>
>      else {
>
>        throw new IllegalArgumentException(s"ALS only supports values in Integer range " +
>
>          s"for columns ${$(userCol)} and ${$(itemCol)}. Value $n was out of Integer range.")
>
>      }
>
>    case _ => throw new IllegalArgumentException(s"ALS only supports values in Integer range " +
>
>      s"for columns ${$(userCol)} and ${$(itemCol)}. Value $n is not numeric.")
>
>  }
>
>}
>

上面的代码显示了一个修改后的checkedCast（），它接收输入，check断言该值是数字，否则引发异常。 
由于输入为Any，因此我们可以安全地从其余代码中删除所有对Double语句的强制转换。 
此外，可以合理地预期，由于ALS要求id在整数范围内，因此大多数人实际上使用整数类型。 
结果，在第3行，此方法显式处理Integer，以避免进行任何强制转换。 对于所有其他数字值，它将检查输入是否在整数范围内。 
该检查发生在第7行。

可以用不同的方式编写此代码，并显式处理所有允许的类型。 
不幸的是，这将导致重复的代码。 
相反，我在这里所做的是将数字转换为整数并将其与原始数字进行比较。 
如果值相同，则满足以下条件之一：

> 1. * The value is Byte or Short.
> 2. * The value is Long but within the Integer range.
> 3. * The value is Double or Float but without any fractional part.

为了确保代码正常运行，我使用Spark的标准单元测试对它进行了测试，并通过检查该方法的行为以查看各种合法和非法值来进行手动测试。 
为确保解决方案至少与原始解决方案一样快，我使用下面的代码片段多次测试。 
可以将其放在Spark的ALSSuite类中：

>test("Speed difference") {
>
>  val (training, test) =
>
>    genExplicitTestData(numUsers = 200, numItems = 400, rank = 2, noiseStd = 0.01)
>
> 
>
>  val runs = 100
>
>  var totalTime = 0.0
>
>  println("Performing "+runs+" runs")
>
>  for(i <- 0 until runs) {
>
>    val t0 = System.currentTimeMillis
>
>    testALS(training, test, maxIter = 1, rank = 2, regParam = 0.01, targetRMSE = 0.1)
>
>    val secs = (System.currentTimeMillis - t0)/1000.0
>
>    println("Run "+i+" executed in "+secs+"s")
>
>    totalTime += secs
>
>  }
>
>  println("AVG Execution Time: "+(totalTime/runs)+"s")
>
>
>}
>


经过几次测试，我们可以看到新的修复程序比原始修复程序快一点：

|Code        |   Number of Runs   |   Total Execution Time   |   Average Execution Time per Run
|:----------:|--------------------|--------------------|--------------------|
|Original	  |     100	           |        588.458s	        |           5.88458s
|Fixed	      |     100            |      	566.722s          |           5.66722s

我重复了多次实验以确认并得出一致的结果。 在这里，您可以找到原始代码和修复程序的一个实验的详细输出。 
对于一个很小的数据集来说，差异很小，但是在过去，我已经使用此修补程序设法显着减少了GC开销。 
我们可以通过在本地运行Spark并在Spark实例上附加Java分析器来确认这一点。 
我在官方的Spark存储库上打开了一张票和一个Pull-Request，它现在是Spark 2.2的一部分。
