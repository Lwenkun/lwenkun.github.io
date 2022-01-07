---
layout:     post
title:      "Glide 之磁盘缓存"
subtitle:   "Glide 之磁盘缓存"
catalog: true
date:       2017-08-29
author:     "lwenkun"
header-img: "img/post-bg-android-volley.png"
tags:
    - Android
    - 开源库
---

# Glide 之磁盘缓存
Glide 提供了灵活的磁盘缓存策略，用户可以定义自己的缓存策略，只需要实现 DiskCache 接口即可。Glide 中已经有两个 DiskCache 实现，一个是 DiskCacheWrapper，这个类什么都没干，就是一个空壳，用来包装其他的 DiskCache 实现类；另一个类是 DiskLruCaheWrapper，它是基于 DiskLruCache 实现的。接下来从 DiskLruCahce 分析一下 Glide 的磁盘缓存。
<!-- more -->
## 磁盘缓存核心 —— DiskLruCahce

DiskLruCache 的分析分为两部分：分别是日志，读、写缓存。

### 日志

DiskLruCache 的日志就是一个操作记录。例如，删除一条缓存条目，就会在日志文件中记录一条 REMOVE 记录；新建一条缓存，缓存文件刚建立时会增加一条 DIRTY 记录，缓存写入成功后再增加一条 CLEAN 记录；缓存被读取，会增加 READ 记录。日志文件的格式如下：


| 记录类型 | key | length1 | length2 |
| ------- | ------------------------------- | --- | --- | 
| CLEAN | 3400330d1dfc7f3f7f4b8d4d803dfcf6 | 832 | 21054|
| DIRTY | 335c4c6028171cfddfbaae1a9c313c52|
| CLEAN | 335c4c6028171cfddfbaae1a9c313c52 | 3934 | 2342 |
| REMOVE | 335c4c6028171cfddfbaae1a9c313c52 |
| DIRTY | 1ab96a171faeeee38496d8b330771a7a |
| CLEAN | 1ab96a171faeeee38496d8b330771a7a| 1600 | 234
| READ | 335c4c6028171cfddfbaae1a9c313c52 |
| READ | 3400330d1dfc7f3f7f4b8d4d803dfcf6 |

之所以记录日志，就是为了在 DiskLruCache 初始化时建立起缓存的 LRU 结构。DiskLruCache 内部使用了 lruEntries:LinkHashMap 来存储所有的缓存项，在初始化的时候，DiskLruCache 会读日志文件上的记录，根据该日志文件的记录将所有历史操作“重做”一遍：在执行历史操作时，会根据日志记录对缓存项进行添加和删除操作。为什么这样能保证缓存项是按 LRU 的顺序排序的呢？因为构造 LinkedHashMap 的时候选择使用 Access Order（访问顺序）来保持元素的顺序，因此只需遍历日志根据日志记录向 LinkedHashMap 中放入缓存项（Entry）自然而然地就保持了 LRU 的顺序。读区日志记录的文件如下：

```java
private void readJournalLine(String line) throws IOException {
  int firstSpace = line.indexOf(' ');
  if (firstSpace == -1) {
    throw new IOException("unexpected journal line: " + line);
  }
  int keyBegin = firstSpace + 1;
  int secondSpace = line.indexOf(' ', keyBegin);
  final String key;
  if (secondSpace == -1) {
    key = line.substring(keyBegin);
    if (firstSpace == REMOVE.length() && line.startsWith(REMOVE)) {
      lruEntries.remove(key);
      return;
    }
  } else {
    key = line.substring(keyBegin, secondSpace);
  }
  Entry entry = lruEntries.get(key);
  if (entry == null) {
    entry = new Entry(key);
    lruEntries.put(key, entry);
  }
  if (secondSpace != -1 && firstSpace == CLEAN.length() && line.startsWith(CLEAN)) {
    String[] parts = line.substring(secondSpace + 1).split(" ");
    entry.readable = true;
    entry.currentEditor = null;
    entry.setLengths(parts);
  } else if (secondSpace == -1 && firstSpace == DIRTY.length() && line.startsWith(DIRTY)) {
    entry.currentEditor = new Editor(entry);
  } else if (secondSpace == -1 && firstSpace == READ.length() && line.startsWith(READ)) {
    // This work was already done by calling lruEntries.get().
  } else {
    throw new IOException("unexpected journal line: " + line);
  }
}
```

这个方法根据记录类型进行缓存的添加和删除：读取到 REMOVE 记录便从 lruEntries 删除对应的缓存项；然后通过 lruEntries.get() 方法尝试获取键值为 key 所对应的缓存项，如果不存在便新建一个并放入 lruEntries 中；判断记录的类型是不是 CLEAN 类型，是则对 entry 的 readable 和 currentEditor 字段进行更新，表示该缓存可供读取；如果类型是 DIRTY 类型，那么为 currentEditor 赋值；如果时 READ 记录，那么什么也不干，原因在注释里已经写明了：因为前面调用 lruEntries.get() 方法时，已经是对缓存元素进行了一次访问，因此可以保证这个被访问的元素排在最前面，即保证了元素按照 LRU 的次序排列。

读取完之后还会对 lruEntries 中的元素进一步处理：

```java
private void processJournal() throws IOException {
  deleteIfExists(journalFileTmp);
  for (Iterator<Entry> i = lruEntries.values().iterator(); i.hasNext(); ) {
    Entry entry = i.next();
    if (entry.currentEditor == null) {
      for (int t = 0; t < valueCount; t++) {
        size += entry.lengths[t];
      }
    } else {
      entry.currentEditor = null;
      for (int t = 0; t < valueCount; t++) {
        deleteIfExists(entry.getCleanFile(t));
        deleteIfExists(entry.getDirtyFile(t));
      }
      i.remove();
    }
  }
}
```
该方法会将其中 currentEditor 不为 null 的 entry 全部删除。原因就是这些缓存项是脏数据，是上次缓存未成功提交的结果。一般来说，一条 DIRTY 记录后会紧跟着一条 key 值相同的 CLEAN 或者 REMOVE 记录，这表明了缓存经历了从被写入到之后成功提交或者删除的过程，这样一来，lruEntries 中要么就不存在该记录对应的 Entry 对象，要么该 Entry 对象的 currentEditor 为 null，即该缓存项有效。如果不清楚这一点可以先往下读再回来理解。

现在，假设有如下日志记录：

```
DIRTY KEY1；CLEAN KEY1；DIRTY KEY2；CLEAN KEY2；DIRTY KEY3；CLEAN KEY3；
READ KEY3；READ KEY2 ；DIRTY KEY4；CLEAN KEY4；REMOVE KEY1
```
这个日志反映的历史操作是：首先依次建立了 key 分别为 KEY1，KEY2，KEY3 的三个缓存文件，然后读取了 key 为 KEY2 和 KEY3 的两个文件，之后又建立了 key 为 KEY4 的文件，最后删除了 key 为 KEY1 的那个缓存文件。这样一来读取该日志文件时 LinkHashMap 添加和删除缓存项的过程就是这样的：首先依次添加 KEY1，KEY2，KEY3 对应的缓存项，再依次分别读取一次 KEY3，KEY2 对应的缓存项，再添加 KEY4 对应的缓存项，最后删除 KEY1 对应的缓存项。最终 lruEntries 中的缓存项的顺序为：KEY3，KEY2，KEY4，其中 KEY4 排在最前面，即最不容易被淘汰。

### 读写缓存

首先看 DiskLruCache 读缓存的方式：

```
DiskLruCache cache = DiskLruCache.open(directory, appVersion, valueCount, maxSize);
Value v = cache.get(key);
File cacheFile = v.getFile(0);
......
```
DisLruCache 的 open() 方法有四个参数：第一个参数是缓存文件的根目录，第二个是应用版本号，第三个是一个 key 对应的缓存文件数目（一般传 1 就可以了，我们通常只需要一份缓存文件），第四个是缓存的最大值，以字节为单位。Value 被称之为“快照”，它包含了缓存项的一些信息，比如缓存项对应的缓存文件。它的 getFile() 方法就是获取该缓存文件，客户端就可以将需要缓存的信息写入该文件。

写缓存的方式如下：

```java
Editor editor = cache.edit(key);
File cacheFile = editor.getFile(0);
...
eidtor.commit();
```

首先调用 edit() 方法开启了在键值为 key 的缓存项上的一次操作，它返回的是一个 Editor 对象，该对象的 getFile() （参数 0 表示的是该 key 对应的缓存项中的第一个缓存文件）方法返回缓存文件供外部执行写缓存操作。需要注意的是，对缓存文件操作完之后需要调用 editor.commit() 方法进行提交，否则这次写在日志文件中只会生成一条 DIRTY 记录与之对应（而不会生成 CLEAN 记录），使得缓存无效。

现在来看读写缓存的内部原理，首先要理解 DiskLruCache 的几个内部类及其角色。其中一个是 Entry，Entry 代表了一个缓存项，这个缓存项不一定就是一个缓存文件，它对应的是含有相同 key 的所有缓存文件，这些文件有 CLEAN 状态的，也有 DIRTY 状态的。其内部的数据结构大致是这样的：

```
class Entry {
    String key; // 缓存项对应的 key
    File[] dirtyFiles; // 临时文件，不完整，不可读取，只可写入
    File[] cleanFiles; // 完整缓存文件，已缓存完毕，可供读取
    long[] lengths; // 完整的缓存文件对应的大小
    Editor currentEditor; // 当前在此 Entry 上执行操作的 Editor 对象
    ......
}
```
其构造方法如下：

```java
private Entry(String key) {
  this.key = key;
  this.lengths = new long[valueCount];
  cleanFiles = new File[valueCount];
  dirtyFiles = new File[valueCount];
  // The names are repetitive so re-use the same builder to avoid allocations.
  StringBuilder fileBuilder = new StringBuilder(key).append('.');
  int truncateTo = fileBuilder.length();
  for (int i = 0; i < valueCount; i++) {
      fileBuilder.append(i);
      cleanFiles[i] = new File(directory, fileBuilder.toString());
      fileBuilder.append(".tmp");
      dirtyFiles[i] = new File(directory, fileBuilder.toString());
      fileBuilder.setLength(truncateTo);
  }
}
```
它的构造方法利用了 key 去构造那些 File 对象：key.index 就是 Clean File，key.index.tmp 就是 Dirty File。也就是说，在读取日志文件的时候，已经将所有缓存项的对应的缓存文件对象都初始化好了，需要注意的是，这里仅仅是根据文件名来构造相应的缓存文件对象而已，那些 cleanFiles 和 dirtyFiles 并不一定是磁盘上存在的文件。

另一个类是 Editor，它的内部字段有：

```
class Editor {
    Entry entry; //该 Editor 操作的缓存项
    boolean[] written; // 完整文件是否被修改
    ......
}
```

现在我们看看写缓存文件的过程。首先是获取 Editor 对象：

```java
private synchronized Editor edit(String key, long expectedSequenceNumber) throws IOException {
  checkNotClosed();
  Entry entry = lruEntries.get(key);
  if (expectedSequenceNumber != ANY_SEQUENCE_NUMBER && (entry == null
      || entry.sequenceNumber != expectedSequenceNumber)) {
    return null; // Value is stale.
  }
  if (entry == null) {
    entry = new Entry(key);
    lruEntries.put(key, entry);
  } else if (entry.currentEditor != null) {
    return null; // Another edit is in progress.
  }
  Editor editor = new Editor(entry);
  entry.currentEditor = editor;
  // Flush the journal before creating files to prevent file leaks.
  journalWriter.append(DIRTY);
  journalWriter.append(' ');
  journalWriter.append(key);
  journalWriter.append('\n');
  journalWriter.flush();
  return editor;
}
```
这个方法做的事可以描述如下：先从 lruEntries:LinkedHashMap 中获取缓存项，然后对缓存的 sequenceNumber 进行验证，如果不存在该缓存项，便新建一个。然后由该缓存项构造出一个 Editor 对象，最后记录一条 DIRTY 日志并返回 Editor 对象。获取 Editor 对象后，便可通过其 getFile(0) 方法获取到对应的缓存文件对象：

```java
public File getFile(int index) throws IOException {
  synchronized (DiskLruCache.this) {
    if (entry.currentEditor != this) {
        throw new IllegalStateException();
    }
    if (!entry.readable) {
        written[index] = true;
    }
    File dirtyFile = entry.getDirtyFile(index);
    if (!directory.exists()) {
        directory.mkdirs();
    }
    return dirtyFile;
  }
}
```

getFile() 返回的是缓存项的第 index 个临时文件对象，客户端便可由该对象建立输出流将缓存写入文件中。因为是写文件，所以在写的时候可能出现异常中断的现象（应用崩溃、应用进程被杀），因此写完缓存之后需要调用 editor.commit() 来提交本次操作：

```java
public void commit() throws IOException {
  completeEdit(this, true);
  committed = true;
}

// suecess 为 true 表示提交本次操作，fale 表示放弃本次操作
private synchronized void completeEdit(Editor editor, boolean success) throws IOException {
  Entry entry = editor.entry;
  if (entry.currentEditor != editor) {
    throw new IllegalStateException();
  }
  // If this edit is creating the entry for the first time, every index must have a value.
  if (success && !entry.readable) {
    for (int i = 0; i < valueCount; i++) {
      if (!editor.written[i]) {
        editor.abort();
        throw new IllegalStateException("Newly created entry didn't create value for index " + i);
      }
      if (!entry.getDirtyFile(i).exists()) {
        editor.abort();
        return;
      }
    }
  }
  // 对缓存重命名
  for (int i = 0; i < valueCount; i++) {
    File dirty = entry.getDirtyFile(i);
    if (success) {
      if (dirty.exists()) {
        File clean = entry.getCleanFile(i);
        dirty.renameTo(clean);
        long oldLength = entry.lengths[i];
        long newLength = clean.length();
        entry.lengths[i] = newLength;
        size = size - oldLength + newLength;
      }
    } else {
      deleteIfExists(dirty);
    }
  }
  redundantOpCount++;
  entry.currentEditor = null;
  
  // 记录日志
  if (entry.readable | success) {
    entry.readable = true;
    journalWriter.append(CLEAN);
    journalWriter.append(' ');
    journalWriter.append(entry.key);
    journalWriter.append(entry.getLengths());
    journalWriter.append('\n');
    if (success) {
      entry.sequenceNumber = nextSequenceNumber++;
    }
  } else {
    lruEntries.remove(entry.key);
    journalWriter.append(REMOVE);
    journalWriter.append(' ');
    journalWriter.append(entry.key);
    journalWriter.append('\n');
  }
  journalWriter.flush();
  if (size > maxSize || journalRebuildRequired()) {
    executorService.submit(cleanupCallable);
  }
}
```
这个方法的逻辑是：先校验一下是不是只有一个 Editor 在操作该缓存项（保证同一时刻一个缓存项上只能有一个写操作），然后判断该缓存项是不是新建的，如果是，那么是不是该缓存项对应的所有的临时缓存文件都存在并且都被写过，如果不是，那么放弃这次操作。验证工作做完后就是重命名的过程了，接下来会把所有的该缓存项的所有实际存在的临时文件的名称都重命名为对应的完整文件的名称（即把后面的 .tmp 后缀去除）。最后就是记录日志，如果成功提交本次操作，那么记下一条 CLEAN 日志，如果选择放弃本次操作，那么记下 REMOVE 日志。通过这个方法可以知道，如果客户端写完缓存后没有调用 Editor#commit() CLEAN 记录是不会被记下的，因此下次读取日志文件的时候，会把该缓存项删除，原因前面已经讲了。

如果缓存一直持续下去，是不是日志记录越来越大呢？答案是否定的，日志文件有一个上限，如果超过该上限便会重建。同样的，lruEntries 中的缓存项也不会无限增长，它受 maxSize 参数的限制，如果容量太大，便会从 LruEntries 中删除最久未被使用的缓存项，并将结果反映到磁盘上的缓存文件上，具体的细节这里就不再讲述了。

## 感谢阅读
以上便是对 Glide 磁盘缓存的核心类 DiskLruCache 的分析，如有不对还请大家不吝赐教。