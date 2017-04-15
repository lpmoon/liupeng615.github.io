最近看到了好几篇关于string.intern()的技术博客，

* [http://hellojava.info/?p=514](http://hellojava.info/?p=514)

* [http://calvin1978.blogcn.com/articles/string-intern.html](http://calvin1978.blogcn.com/articles/string-intern.html)

这两篇文章介绍了一些intern的内幕，以及使用上应该避免的坑。但是没有更过的介绍intern的实现细节，于是自己去看了一下jvm的具体实现，在这里做一个笔记记录一下，方便以后翻阅。



**以下的源码基于[openjdk-8-src-b132-03](http://download.java.net/openjdk/jdk8/promoted/b132/openjdk-8-src-b132-03_mar_2014.zip)**



string的intern()入口在jdk/src/share/native/java/lang/String.c，

```

#include "jvm.h"

#include "java_lang_String.h"



JNIEXPORT jobject JNICALL

Java_java_lang_String_intern(JNIEnv *env, jobject this)

{

    return JVM_InternString(env, this);

}

```

进入到JVM_InternString中，

```

 JVM_ENTRY(jstring, JVM_InternString(JNIEnv *env, jstring str))

  JVMWrapper("JVM_InternString");

  JvmtiVMObjectAllocEventCollector oam;

  if (str == NULL) return NULL;

  oop string = JNIHandles::resolve_non_null(str);

  oop result = StringTable::intern(string, CHECK_NULL);

  return (jstring) JNIHandles::make_local(env, result);

JVM_END

```

有些人可能会疑惑JVM_ENTRY是啥意思，其实没有什么太神秘的，由于jvm内部很多函数调用的最开始都要进行相同的操作，所以jvm将这些相同的操作抽象成了一组宏。

```

#define JVM_ENTRY(result_type, header)                               \

extern "C" {                                                         \

  result_type JNICALL header {                                       \

    JavaThread* thread=JavaThread::thread_from_jni_environment(env); \

    ThreadInVMfromNative __tiv(thread);                              \

    debug_only(VMNativeEntryWrapper __vew;)                          \

    VM_ENTRY_BASE(result_type, header, thread)

```

代码的主要处理调用的使StringTable::intern()，进入到这个函数看看。

```

oop StringTable::intern(Handle string_or_null, jchar* name,

 int len, TRAPS) {

 unsigned int hashValue = hash_string(name, len);

 int index = the_table()->hash_to_index(hashValue);

 oop found_string = the_table()->lookup(index, name, len, hashValue);



 // Found

 if (found_string != NULL) return found_string;



 debug_only(StableMemoryChecker smc(name, len * sizeof(name[0])));

 assert(!Universe::heap()->is_in_reserved(name),

 "proposed name of symbol must be stable");



 Handle string;

 // try to reuse the string if possible

 if (!string_or_null.is_null()) {

 string = string_or_null;

 } else {

 string = java_lang_String::create_from_unicode(name, len, CHECK_NULL);

 }



 // Grab the StringTable_lock before getting the_table() because it could

 // change at safepoint.

 MutexLocker ml(StringTable_lock, THREAD);



 // Otherwise, add to symbol to table

 return the_table()->basic_add(index, string, name, len,

 hashValue, CHECK_NULL);

}

```

主要分为以下几步



* 根据查询的字符串以及长度计算hash值，

```

unsigned int StringTable::hash_string(const jchar* s, int len) {

  return use_alternate_hashcode() ? AltHashing::murmur3_32(seed(), s, len) :

  java_lang_String::hash_code(s, len);

}

```

* 根据hash值以及table的size，计算index值

```

int hash_to_index(unsigned int full_hash) {

int h = full_hash % _table_size;

assert(h >= 0 && h < _table_size, "Illegal hash value"); 

return h; }

```

* 字符串查找

```

oop StringTable::lookup(int index, jchar* name,

                        int len, unsigned int hash) {

  int count = 0;

  for (HashtableEntry<oop, mtSymbol>* l = bucket(index); l != NULL; l = l->next()) {

    count++;

    //　找到的条件

    if (l->hash() == hash) {

      if (java_lang_String::equals(l->literal(), name, len)) {

        return l->literal();

      }

    }

  }

  //　是否需要rebash

  // If the bucket size is too deep check if this hash code is insufficient.

  if (count >= BasicHashtable<mtSymbol>::rehash_count && !needs_rehashing()) {

    _needs_rehashing = check_rehash_table(count);

  }

  return NULL;

}

```

首先根据index值取出对应的bucket，然后依次遍历bucket中的每个值，如果相同则返回。如果找不到则判断是否需要进行rehash操作。



* 如果找到了，则直接返回

```

if (found_string != NULL) return found_string;

```

* 如果没有则获取锁

* 新建string，并且更新StringTable

```

oop StringTable::basic_add(int index_arg, Handle string, jchar* name,

                           int len, unsigned int hashValue_arg, TRAPS) {



  assert(java_lang_String::equals(string(), name, len),

         "string must be properly initialized");

  // Cannot hit a safepoint in this function because the "this" pointer can move.

  No_Safepoint_Verifier nsv;



  // Check if the symbol table has been rehashed, if so, need to recalculate

  // the hash value and index before second lookup.

  unsigned int hashValue;

  int index;

  if (use_alternate_hashcode()) {

    hashValue = hash_string(name, len);

    index = hash_to_index(hashValue);

  } else {

    hashValue = hashValue_arg;

    index = index_arg;

  }



  // Since look-up was done lock-free, we need to check if another

  // thread beat us in the race to insert the symbol.



  oop test = lookup(index, name, len, hashValue); // calls lookup(u1*, int)

  if (test != NULL) {

    // Entry already added

    return test;

  }



  HashtableEntry<oop, mtSymbol>* entry = new_entry(hashValue, string());

  add_entry(index, entry);

  return string();

}

```

在插入之前需要判断是否相同的值已经插入了stringtable，因为最开始的looup调用是lock-free的。如果已经有其他线程抢先更新了，则直接返回。否则进行添加。

