## 写在代码之前
在网上搜了很多都说要打包jar，或者需要在环境变量中写入，比较繁琐。但是笔者写的这个代码也是能直接跑的，正常输出结果。主要是要有下面这行代码，会在控制台打印出运行结果并在完成后退出，这样看着比较直观一点。
```
System.exit(job.waitForCompletion(true) ? 0 : 1);

``` 
### Mapper类
```
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

import java.io.IOException;
import java.util.Arrays;

public class MapperTest extends Mapper<LongWritable
		, Text, Text, IntWritable> {
	/**
	 * @param key     文本的行号
	 * @param value   待统计单词的文本
	 * @param context
	 * @throws IOException
	 * @throws InterruptedException
	 */
	@Override
	protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
		//得到文件的文本,每一行为一个value
		String text = value.toString();
		String[] words = text.split(" ");
		//将每个单词进行统计 出现次数为1
		Arrays.stream(words).forEach(
				word -> {
					try {
						context.write(new Text(word), new IntWritable(1));
					} catch (IOException e) {
						e.printStackTrace();
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
				}
		);
	}
}

```
### Reducer

```
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

import java.io.IOException;

public class ReducerTest extends Reducer<Text, IntWritable, Text, IntWritable> {
	/**
	 * @param key     单词
	 * @param values  对应频数
	 * @param context
	 */
	@Override
	protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
		//统计频数之和
		int sum = 0;
		for (IntWritable count : values) {
			sum += count.get();
			context.write(key, new IntWritable(sum));
		}
	}
}

```
### 启动类主函数

```
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;

public class Runner {
	private static final String inpath = "D:\\codeWareCollections\\hadoop-2.9.1\\input\\README.txt";

	public static void main(String[] args) {
		Configuration conf = new Configuration();
		try {
			System.setProperty("hadoop.home.dir", "D:\\codeWareCollections\\hadoop-2.9.1");
			Job job = Job.getInstance(conf, "first Hadoop");
			job.setJarByClass(Runner.class);
			job.setMapperClass(MapperTest.class);
			job.setReducerClass(ReducerTest.class);

			// 输出key类型
			job.setOutputKeyClass(Text.class);
			// 输出value类型
			job.setOutputValueClass(IntWritable.class);
			//设置输入输出路径
			Path inputPath = new Path(inpath);
			FileInputFormat.addInputPath(job, inputPath);
			FileOutputFormat.setOutputPath(job, new Path("D:\\output"));
			System.exit(job.waitForCompletion(true) ? 0 : 1);

		} catch (IOException e) {
			e.printStackTrace();

		} catch (InterruptedException e) {
			e.printStackTrace();
		} catch (ClassNotFoundException e) {
			e.printStackTrace();
		}
	}
}

```