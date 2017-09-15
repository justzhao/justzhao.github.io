---
layout: post
title:  MappedByteBuffer
category: 技术基础
tags: 虚拟内存，内存映射
description: 虚拟内存，内存映射，内存管理。
---

内存映射文件提供了一种进程存取文件的机制。使用映射文件可显著减少 I/O 数据移动，因为文件数据不必如 read 和 write 子例程所做的那样被复制到进程数据缓冲区中。

#### 内存管理

多道程序设计系统中，必须在内存中分出用户的部分，以满足多个进程的要求，细分的任务由操作系统完成，称为内存管理。
内存管理最基本的操作是由处理器吧程序装入内存中执行。

##### 页宽

内存中一个固定长度快。

##### 页

一个固定长度的数据块，存储在耳机存储中，磁盘。数据页可以临时复制到页框中。


##### 虚拟内存


计算机系统内存管理的一种技术。它使得应用程序认为它拥有连续的可用的内存（一个连续完整的地址空间），而实际上，它通常是被分隔成多个物理内存碎片，还有部分暂时存储在外部磁盘存储器上，在需要时进行数据交换。
虚拟内存其实就是硬盘的一部分，是计算机RAM与硬盘的数据交换区,文件名是PageFile.Sys，称为是“页面文件”

##### 物理内存


内存条


##### 虚拟内存地址和物理内存地址



 每个进程有独立的虚拟地址空间，进程访问地址是一段逻辑虚拟机制,虚拟地址可通过
进程上页表与物理地址进行映射，获得真正物理地址。

如果虚拟地址对应物理地址不在物理内存中，也就是需要访问的页不在内存。
则产生缺页中断，此时需要操作系统将其对应的页加载到内存中。如果内存满了，需要把内存中最近最少的页换出。被内存映射的文件实际上成了一个分页交换文件。




###### 虚拟内存地址:

由页号（与页表中的页号关联）和偏移量组成
###### 物理内存地址:
物理上真正的内存地址。


###### 总的来说：

   


![](http://7x00ae.com1.z0.glb.clouddn.com/17-9-13/8596806.jpg)

##### 内存管理单元(MMU)
管理虚拟存储器、物理存储器的控制线路，同时也负责虚拟地址映射为物理地址，映射表就存在这。






##### 内存映射

内存映射文件是由一个文件到一块内存的映射,适合操作大文件。
原理是使进程虚拟地址空间的某个区域建立映射磁盘文件的全部或部分内容，通过该区域可以直接对被映射的磁盘文件进行访问，而不必执行文件I/O操作也无需对文件内容进行缓冲处理。

#### Java的 内存映射文件


Java中的内存文件映射是通过MappedByteBuffer实现的。
MappedByteBuffer 是Java NIO 的缓冲区的类。作用是把文件映射到虚拟内存，此时操作虚拟内存就相当于操作文件，通常都用来操作大文件。




![](http://7x00ae.com1.z0.glb.clouddn.com/17-9-10/39678180.jpg)







#### 写文件的例子


       File file = new File("D://mapped");
        int fileSize = 1024 * 1024;

        try {
            FileChannel fileChannel = new RandomAccessFile(file, "rw").getChannel();
            MappedByteBuffer mappedByteBuffer =fileChannel.map(FileChannel.MapMode.READ_WRITE, 0, fileSize);
            mappedByteBuffer.putInt(1);
            mappedByteBuffer.flip();
        } catch (Exception e) {

        }
        


#### 读文件的例子

     File file = new File("D://mapped");
        int fileSize = 1024 * 1024;
        try {
            FileChannel fileChannel = new RandomAccessFile(file, "rw").getChannel();
            MappedByteBuffer mappedByteBuffer = fileChannel.map(FileChannel.MapMode.READ_WRITE, 0, fileSize);

            int res = mappedByteBuffer.getInt();
            System.out.println("mappedByteBuffer result is " + res);
        } catch (Exception e) {

        }

 在上面的例子可以快速的读写，通过一块缓冲区映射到文件中。
 
#### 原理


打开一个文件

        public RandomAccessFile(File file, String mode)
        throws FileNotFoundException
    {
        String name = (file != null ? file.getPath() : null);
        int imode = -1;
        
        // 检测读写权限
        if (mode.equals("r"))
            imode = O_RDONLY;
        else if (mode.startsWith("rw")) {
            imode = O_RDWR;
            rw = true;
            if (mode.length() > 2) {
                if (mode.equals("rws"))
                    imode |= O_SYNC;
                else if (mode.equals("rwd"))
                    imode |= O_DSYNC;
                else
                    imode = -1;
            }
        }
        if (imode < 0)
            throw new IllegalArgumentException("Illegal mode \"" + mode
                                               + "\" must be one of "
                                               + "\"r\", \"rw\", \"rws\","
                                               + " or \"rwd\"");
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkRead(name);
            if (rw) {
                security.checkWrite(name);
            }
        }
        if (name == null) {
            throw new NullPointerException();
        }
        if (file.isInvalid()) {
            throw new FileNotFoundException("Invalid file path");
        }
        fd = new FileDescriptor();
        fd.incrementAndGetUseCount();
        this.path = name;
        // 调用native方法,返回文件描述符.
        open(name, imode);
    }
    
接下来看获取fileChannel


每个文件描述符都有一个使用计算器，当文件描述符被输入输出流使用时候，计算器加1.当调用相关的close方法时候，计算器会减一。当没有流使用这个文件描述符的时候会被GC回收。


     public final FileChannel getChannel() {
        synchronized (this) {
        
            if (channel == null) {
                // 根据文件描述符生成Channel。    
                channel = FileChannelImpl.open(fd, path, true, rw, this);

                /*
                 * FileDescriptor could be shared by FileInputStream or
                 * FileOutputStream.
                 * Ensure that FD is GC'ed only when all the streams/channels
                 * are done using it.
                 * Increment fd's use count. Invoking the channel's close()
                 * method will result in decrementing the use count set for
                 * the channel.
                 */
                 // 文件描述符的计数器，
                fd.incrementAndGetUseCount();
            }
            return channel;
        }
    }
    
    
    
    
接下来着重看map方法fileChannel的map方法



         MappedByteBuffer mappedByteBuffer = fileChannel.map(FileChannel.MapMode.READ_WRITE, 0, fileSize);
            mappedByteBuffer.putInt(1);
            

     public abstract MappedByteBuffer map(MapMode mode,long position, long size)
 
 
     mode 一般取读写两个模式，表示需要做的操作
     
     position 表示缓冲区映射文件的开始位置
     
     size 表示映射缓冲区的大小
     
     
     
如下是map方法的实现，主要是下面三个工作


- 校验参数
- 调用native方法，把文件映射到内存
-  调用Util.newMappedByteBuffer 返回缓冲区




        public MappedByteBuffer map(MapMode mode, long position, long size)
        throws IOException
    {
        ensureOpen();
        if (position < 0L)
            throw new IllegalArgumentException("Negative position");
        if (size < 0L)
            throw new IllegalArgumentException("Negative size");
        if (position + size < 0)
            throw new IllegalArgumentException("Position + size overflow");
        if (size > Integer.MAX_VALUE)
            throw new IllegalArgumentException("Size exceeds Integer.MAX_VALUE");
        int imode = -1;
        if (mode == MapMode.READ_ONLY)
            imode = MAP_RO;
        else if (mode == MapMode.READ_WRITE)
            imode = MAP_RW;
        else if (mode == MapMode.PRIVATE)
            imode = MAP_PV;
        assert (imode >= 0);
        if ((mode != MapMode.READ_ONLY) && !writable)
            throw new NonWritableChannelException();
        if (!readable)
            throw new NonReadableChannelException();

        long addr = -1;
        int ti = -1;
        try {
            begin();
            ti = threads.add();
            if (!isOpen())
                return null;
            if (size() < position + size) { // Extend file size
                if (!writable) {
                    throw new IOException("Channel not open for writing " +
                        "- cannot extend file to required size");
                }
                int rv;
                do {
                    rv = nd.truncate(fd, position + size);
                } while ((rv == IOStatus.INTERRUPTED) && isOpen());
            }
            if (size == 0) {
                addr = 0;
                // a valid file descriptor is not required
                FileDescriptor dummy = new FileDescriptor();
                if ((!writable) || (imode == MAP_RO))
                    return Util.newMappedByteBufferR(0, 0, dummy, null);
                else
                    return Util.newMappedByteBuffer(0, 0, dummy, null);
            }

            int pagePosition = (int)(position % allocationGranularity);
            long mapPosition = position - pagePosition;
            long mapSize = size + pagePosition;
            try {
            // 上面都是一些校验参数，这里调用native方法，把文件映射进内存.
                // If no exception was thrown from map0, the address is valid
                // 这里会返回文件实际地址，可以直接通过此地址进行读写操作。
                addr = map0(imode, mapPosition, mapSize);
            } catch (OutOfMemoryError x) {
            // 如果第一次映射发生益处，先GC，等1秒后开始。
                // An OutOfMemoryError may indicate that we've exhausted memory
                // so force gc and re-attempt map
                System.gc();
                try {
                    Thread.sleep(100);
                } catch (InterruptedException y) {
                    Thread.currentThread().interrupt();
                }
                try {
                    addr = map0(imode, mapPosition, mapSize);
                } catch (OutOfMemoryError y) {
                    // After a second OOME, fail
                    throw new IOException("Map failed", y);
                }
            }

            // On Windows, and potentially other platforms, we need an open
            // file descriptor for some mapping operations.
            FileDescriptor mfd;
            try {
                mfd = nd.duplicateForMapping(fd);
            } catch (IOException ioe) {
                unmap0(addr, mapSize);
                throw ioe;
            }

            assert (IOStatus.checkAll(addr));
            assert (addr % allocationGranularity == 0);
            int isize = (int)size;
            Unmapper um = new Unmapper(addr, mapSize, isize, mfd);
            // 根据读写模式返回MappedByteBuffer
            if ((!writable) || (imode == MAP_RO)) {
                return Util.newMappedByteBufferR(isize,
                                                 addr + pagePosition,
                                                 mfd,
                                                 um);
            } else {
                return Util.newMappedByteBuffer(isize,
                                                addr + pagePosition,
                                                mfd,
                                                um);
            }
        } finally {
            threads.remove(ti);
            end(IOStatus.checkAll(addr));
        }
    }     
    
    
  
  Util 的  newMappedByteBuffer方法。可以看到返回的缓冲区是directByteBufferConstructor产生的，也就是directByteBuffer（堆外内存）
    
       static MappedByteBuffer newMappedByteBuffer(int size, long addr,
                                                FileDescriptor fd,
                                                Runnable unmapper)
    {
        MappedByteBuffer dbb;
        if (directByteBufferConstructor == null)
            initDBBConstructor();
        try {
            dbb = (MappedByteBuffer)directByteBufferConstructor.newInstance(
              new Object[] { new Integer(size),
                             new Long(addr),
                             fd,
                             unmapper });
        } catch (InstantiationException e) {
            throw new InternalError();
        } catch (IllegalAccessException e) {
            throw new InternalError();
        } catch (InvocationTargetException e) {
            throw new InternalError();
        }
        return dbb;
    }
    
    
#### 读分析

    mappedByteBuffer.getInt(); 最终会调用到DirectByteBuffer 的如下方法
    实际上可以看到getInt会根据前面返回的address去读文件。 这样是直接从硬盘读文件到内存中，相比传统的read()方法少了一次拷贝的过程。
    
    public int getInt() {
        return getInt(ix(nextGetIndex((1 << 2))));
    }

     private int getInt(long a) {
            if (unaligned) {
                int x = unsafe.getInt(a);
                return (nativeByteOrder ? x : Bits.swap(x));
            }
            return Bits.getInt(a, bigEndian);
    }
    
    // ix 是用来获取物理地址的方法。
     private long ix(int i) {
        return address + (i << 0);
    }


#### 写分析

写也是计算address来写。



    public ByteBuffer putInt(int x) {

        putInt(ix(nextPutIndex((1 << 2))), x);
        return this;
    }
    

     private ByteBuffer putInt(long a, int x) {

        if (unaligned) {
            int y = (x);
            unsafe.putInt(a, (nativeByteOrder ? y : Bits.swap(y)));
        } else {
            Bits.putInt(a, x, bigEndian);
        }
        return this;

    }
    
    // 计算下一次存放的位置和返回当前写的位置    
      final int nextPutIndex(int nb) {                    // package-private
        if (limit - position < nb)
            throw new BufferOverflowException();
        int p = position;
        position += nb;
        return p;
    }
    


#### 总结

Java的内存映射MappedByteBuffer，使用的虚拟内存技术，申请一块非堆内存和文件映射。
当访问某个内存地址时候发现不存在的数据时候，会发生缺页中断，此时中断响应会从页面文件找相应的数据，如果没有在从硬盘中读取相应的数据到内存中。


