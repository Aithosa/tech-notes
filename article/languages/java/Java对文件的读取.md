# Java 对文件的读取

在 Java 中，可以使用 BufferedReader 或者 BufferedInputStream 等缓冲流来读取大文件，这种方式可以避免一次性将整个文件加载到内存中，避免内存溢出。这里提供一个使用 BufferedReader 来读取文件的示例：

```java
import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;

public class ReadLargeFile {
    public static void main(String[] args) {
        BufferedReader reader = null;
        try {
            reader = new BufferedReader(new FileReader("largefile.txt"));
            String line = null;
            while ((line = reader.readLine()) != null) {
                System.out.println(line);
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (reader != null) {
                    reader.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

在以上代码中，我们首先创建了一个 BufferedReader 对象，然后通过 readLine()方法逐行读取文件内容，这样可以有效地处理大文件，而不会因为文件过大而导致内存溢出。注意，当读取完文件后，一定要记得关闭 BufferedReader 对象，以释放系统资源。

采用 FileInputStrea：

在 Java 中，我们通常会使用 BufferedInputStream 来包装 FileInputStream 以提高读取大文件的效率。下面是一个例子：

```java
import java.io.*;

public class ReadBigFile {
    public static void main(String[] args) {
        File file = new File("your_file_path"); //替换为你的文件路径
        try {
            FileInputStream fis = new FileInputStream(file);
            BufferedInputStream bis = new BufferedInputStream(fis);
            byte[] buffer = new byte[1024]; //可以根据实际情况调整缓冲区大小
            int length = 0;
            while ((length = bis.read(buffer)) != -1) {
                //处理读取的数据，例如打印
                System.out.write(buffer, 0, length);
            }
            //记得关闭流
            bis.close();
            fis.close();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

上述代码每次会读取 1024 字节的数据到缓冲区，然后进行处理。这样可以有效地减少磁盘 I/O 操作，提高读取大文件的速度。
