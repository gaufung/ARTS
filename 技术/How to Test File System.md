---
title: How to Test File System
date: 2019-09-06
status: public
---

# 1 Background
Imaging you are writing the application or library on manipulating file on local machine, How will you test your code? Supposing you use `C#` programming language, you probable type down those codes.
```csharp:n
private void DumpSomething(object obj, string filePath)
{
    using(StreamWriter writer = new FileStream(filePath, FileMode.Create))
    {
        writer.Write(obj);
        //... elided
    }
}
```
Now that, if you write unit tests, you might write a lot of useless file in the workspace. Perhaps you could do it well by adding perfect test initialization and tearing down work. 

# 2 Improve
Have you ever bear in mind about Design Pattern's rule of thumb: **Never relay on concret but abstract**. So in above case, Do we actually need a file to write or just a stream writer without caring about what they are. The theory is same if you want to read something from a file or anything else. So we can refactor with our previous demo as follow
```csharp:n
private void DumpSomething(object obj, StreamWriter writer)
{
    writer.Write(obj);
}
```
In this case, you don't need create actual file instead of memory stream writer. 

# 3 System.IO.Abstraction
`IO` is basic componnet in daily coding life. Do we need write every baisc interface whenever encounter `I/O` operation? In `.Net` language, `System.IO` namespace include all those operation. Could we abstract all of them into an interface so that we could mock them if necessary? Fortunately someone has completed that: [System.IO.Abstraction](https://github.com/System-IO-Abstractions/System.IO.Abstractions) which wrapper another interface with default `system.IO`. Let's take a look about how it use. 
```csharp:n
public interface IFileSystem
{
    IFile File { get; }
    IDirectory Directory { get; }
    IFileInfoFactory FileInfo { get; }
    IFileStreamFactory FileStream { get; }
    IPath Path { get; }
    IDirectoryInfoFactory DirectoryInfo { get; }
    IDriveInfoFactory DriveInfo { get; }
    IFileSystemWatcherFactory FileSystemWatcher { get; }
}
```
This interface almost contains all possible IO operations, including `File`, `Directionary`, `Path` and so on. Every interface has exactly same methods or properites. So it make no differences with  `System.IO` usages. The only thing you has to change is as follow:
```csharp:n
public class MyClass()
{
    private IFileSystem _fileSystem;
    public MyClass(IFileSystem fileSystem=null)
    {
        if(fileSystem==null)
        {
            this._fileSystem = new FileSystem();
        }
        else
        {
            this._fileSystem = fileSystem;
        }
    }
}
```
You bet, the `FileSystem` wrappers the `System.IO` all basic usge. In normal case, it doesn't make any impact on origin code flow. When writing unit tests, you can pass your mocked interface into this constructor which any specific behaviors are predefined by you. In unit tests, [NSubstitute](https://github.com/nsubstitute/NSubstitute) is strongly recommended. Here are sample code.
```csharp:n
public void Initialize()
{
    var fileSystem = Substitude.For<IFileSystem>();
    fileSystem.File.ReadAllText("D:\sample.dat").Returns("Hello World!");
    fileSystem.FileInfo.FromFileName("D:\sampe.dat").Length.Return(10);
    //elided
    myClass = new MyClass(fileSystem);
}
```
As we known, any `IFileSystem` calls wil return result as we want, rather than creating actual file before or after our unit tests.

