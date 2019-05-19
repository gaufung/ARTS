---
date: 2017-06-17 12:38
status: public
title: Java IO Usages
---

# 1 Overview
The classes about I/O group in `java.io` package. Java's IO package involves reading raw data from a source and writing of processed data to a destinaiton.
Here list some most typical sources and destinations of data
+ Files
+ Pipes
+ Network Connections
+ In-memory Buffers(e.g. arrays)
+ System.in, System.out, System.err

All Java IO classes can be separate by input, output,being byte based or character based.
![](~/javaiooverview.png)
# 2 InputStream and OutputStream
The `java.io.InputStream` and `java.io.OutputStream` class are the base classes for all `Java.IO` input and output stream. 
## 2.1 InputStream
The *read()* method defined in `InputStream` returns a `int` contianing the byte value of byte read and it will be **-1** if thre is no more data can be read.
```Java
InputStream input = new FileStream("data.txt");
int data;
while((data = input.read())!=-1){
    //process the byte data
    process(data);
}
```
## 2.2 OutputStream
Just oppisite against `InputStream`, the `OutputStream` provides the *write()* method writing bytes to destinations.
```Java
// ture: append
// false: overwrite
OutputStream output = new FileStream("data.txt", true);
output.write("Hello world!".getBytes());
close();
```

# 3 File Stream
`FileInputStream` class makes it possiable to read content of a file as a stream of bytes. It is a subclass of `InputStream`. Besides the *read()* methods, *read(byte[])* can read data into `byte` array.
`FileOutputStream` class just overrides all methods in `OutputStream` class.
## 3.1 Try With Resources
Some exceptions will occur after opening inputStream and make resource releasing unproperly. Try-with cluase will solve this headache problem.
```Java
try{
    InputStream input = new FileInputStream("data.txt");
    // some procession
}catch(Exception e){
    // handle the exception
}finally{
    input.close();
}
// More Simplicity
try(InputStream input = new FileInputStream("data.txt“)){
    //some procession
}catch(Exception e){
    handle the exceptions
}//without finally cluase
```

# 4 Decorated Input/Output Stream
The `java.io` package designers take **Decorator Pattern** strategy to solve various I/O situations. See [Decorator Pattern](http://gaufung.info/post/python_decorator).

## 4.1 Buffer Stream
Buffering can speed up IO quite a lot as avoiding reading or writing only one byte at a time. It reads or writes a large block a time. 
```Java
InputStream input = new BufferedInpuStream(
                    new FileInputStream("data.txt"));
InputStream output = new BufferedOutputStream(
                    new FileOutputStream("data.txt", false));
```
## 4.2 Data Stream
The `DataInputStream` class enables you to read Java primitives(byte, int, float etc.) for an inputStream instead of only raw bytes. `DataOutputStream` also make it possiable write Java primitives to an outputStream, rather than raw bytes.
```Java
// output
DataOutputStream  output = new DataOutputStream(
                           new FileOutputStream("data.dat"));
output.writeInt(32); output.writeFloat(42.1F); output.writeUTF("hello world");
out.close();
// Input
DataInputStream input = new DataInputStream(
                        new FileInputStream("data.dat")):
System.out.println(String.format("%d,%f,%s", input.readInt(), input.readFloat(),
input.readUTF()); // 32,42.1,hello world
input.close();
```
## 4.3 PushbackInputStream
Sometimes you can only determine how to interpret the current byte only by a few ahead bytes. The `PushbackInputStream` enables you to push read bytes back into the stream. These byte will be read next time you call the *read*.
```Java
PushbackInputStream input = new PushbackInputStream(
                            new FileInputStream("data.txt"));
byte[] buffer = new byte[2];
while(input.read(buffer)!=-1){
    if(buffer[1]<100){
        input.unread(buffer[1]);
        //process only buffer[0] byte
    }else{
        //process those buffer[0] and buffer[1] bytes
    }
}
```
## 4.4 Object Stream
Serialization is a vital feature to load or dump objects. The ObejctInputStream and ObjectOutputStream are used to serialze.
```Java
//implement the Seriablizable interface
class Person implements Serializable{
    public String name;
    public int age;
    public Person(String name, int age){
        this.name = name;
        this.age = age;
    }
}
//
ObjectOutputStream output = new ObjectOutputStream(new              FileOutputStream("Person.dat"));
        output.writeObject(new Person(25,"Snail"));
        output.close();

ObjectInputStream input =new ObjectInputStream(new FileInputStream("Person.dat"));
        Person person = (Person)input.readObject();
        System.out.println(String.format("Name: %s, Age: %d", person.name,person.age));
```
# 5 Piped Stream
Pipes in Java IO provides the ability for two threads running in the same JVM to comunitcate. Therefore pipes can also be sources or destinations of data.
```Java
final PipedOuputStream output = new PipedOutputStream();
final PipedInputStream input = new PipedInputStream(output);
Thread thread1 = new Thread(new Runable(){
     @Override
     public void run(){
        output.write("Hello world, pipe".getBytes());
    }
});
Thread thread2 = new Thread(new Runable(){
    @Override
    public void run(){
        int data;
        while((data=input.read())!=-1){
            System.out.println((char)data);
        }
    });
thread1.start();
thread2.start();
```

# 6 Reader and Writer
`Reader` class is the base class for all `Reader` subclasses in java IO API. The *read()* method of Reader return an int which represents the `char` value. If *read()* returns -1 meaning that no more data can be read. That is, -1 as **int** value not as byte or char value. `InputStreamReader` class wraps an `InputStream` to turn the byte based input stream into a character based reader.
```Java
InputStream inputStream = new FileInputStream("data.txt");
Reader inputReader = new InputStreamReader(inputStream);
```

`Writer` class is the base class for all **Writer** subclass in the Java IO API. `OutputStreamWriter` class wraps an `OutputStream` to turn the byte based output stream into a character based writer.
```Java
OutputStream output = new FileOutputStream("data.txt");
Writer outputWriter = new OutputStreamWriter(output,"UTF-8");
```
# 7 Reader and Writer's SubClasses
## 7.1 FileReader and FileWriter
```Java
Reader fileReader = new FileReader("data.txt");
// read character
Writer fileWriter = ne FileWriter("data.txt");
//write character
```
## 7.2 PipedReader and PipedWriter
```Java
PipeWriter pipedWriter = new PipedWriter();
PipedReader pipedReader = new PipedReader(pipeWriter);
// write reader and write procession
```
## 7.3 BufferedReader and BufferdWriter
The `BufferedReader` has one extra method though, the `readLine()` method. This method can be handy once you want to process input line by line.
```Java
BufferedReader reader = new BufferedReader(new FileReader("data.txt"));
String line;
while((line=reader.readLine())!=null){
    //process line
}
```
# 8 System.in,out,err
The three streams `System.in`, `System.out` and `System.err` are also common sources or destinations of data. And they are initialized by the Java runtime when JVM starts up.
## 8.1 System.in
`System.in` is an `InputStream` which is typically connected to keyboard input of console programs.
```Java
BufferedReader br = new BufferredReader(new InputStreamREader(System.in));
String line;
while((line = br.readLine())!=null){
    //parse the line
}
```

## 8.2 System.out
`System.out` is `PrintStream` and outputs the data you write to it to console. You can set new System stream.
```Java
OutputStream output = new FileOutputStream("data.txt");
PrintStream printout = new PrintStream(output);
System.setOut(printOut);
```