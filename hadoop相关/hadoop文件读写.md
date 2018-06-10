
- [FileSystem文件的读取](#filesystem文件的读取)
- [HDFS的一致模型](#hdfs的一致模型)

## FileSystem文件的读取
- 重要的类:
    - FileSystem:文件系统主类，实例可以通过FileSystem.get()方法获得。
    - Configuration:配置类，无参默认构造core-site.xml中的内容。
    - IOUtils:处理输入输出的工具类，文中采取了其中一种重载，使其在读取完流之后并不自动关闭。
    - FSDataInputStream:fs.open()返回的实例。其中的seek方法是绝对定位，不像InputStream的skip只能定位到当前位置之后的相对位置.但此操作开销比较高
    - Progressable:这是一个回调接口。可以用于通知调用方法进度
    - PathFilter:用于匹配过滤文件路径
    ```java
    @Public
    @Stable
    public interface Progressable {
        void progress();
    }
    ```
- 实验代码
```java
package example.hadoop.Authority_Guide.part_3;

import java.net.URI;
import java.io.*;
import org.apache.hadoop.io.*;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.*;
import org.apache.hadoop.fs.FileSystem;

public class FileSystemReader {
	public static void main(String[] args) throws IOException{
		System.setProperty("hadoop.home.dir", "D:\\codeWareCollections\\hadoop-2.9.1");
		String uri =  "hdfs://192.168.181.1:9000/test.txt";
		Configuration conf = new Configuration();
		try {
			FileSystem fs = FileSystem.get(URI.create(uri), conf);
			FSDataInputStream in = fs.open(new Path(uri));
			IOUtils.copyBytes(in,System.out,4096,false);
            //在这里相当于把流重置，再读一遍
			in.seek(0);
			IOUtils.copyBytes(in,System.out,conf);
		} catch (Exception e) {
			e.printStackTrace();
		}

	}
}
```  
## HDFS的一致模型
- 一般情况：在一般情况下，hdfs的正在写入的文件是不能准确反映的，只能反映在之前已经完成写入的文件。如果一定要立即看到情况，可以使用hsync()和hflush()，但 *hflush*并不保证已经完成向磁盘的写入，它只是将datanode内存中的缓存刷新，因此hsync更安全。但注意，这类型的操作 **开销高**

 