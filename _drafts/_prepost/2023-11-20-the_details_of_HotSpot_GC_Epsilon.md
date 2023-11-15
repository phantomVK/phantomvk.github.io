---
layout:     post
title:      "HotSpot Epsilon源码实现"
date:       2023-11-20
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - JVM
---

## 一、简介

__Epsilon__ 垃圾回收器来自 [JEP-318](https://openjdk.java.net/jeps/318) 提案，目标是实现最低的内存分配延迟、与其他GCs无关的新算法。可以实现无垃圾回收算法影响，去测试Java应用的真实吞吐量及性能指标。


原作者 Aleksey Shipilev 还在其博客里，介绍如何基于 __Epsilon__ 源码通过简单改造，实现初步的回收机制 [Do It Yourself (OpenJDK) Garbage Collector](https://shipilev.net/jvm/diy-gc/#_epsilon_gc)。本文涉及源码基于git master分支、节点 252aaa9249d8979366b37d59487b5b039d923e35。

源码树结构：

```
├── epsilon
│ ├── epsilonArguments.cpp
│ ├── epsilonArguments.hpp
│ ├── epsilonBarrierSet.cpp
│ ├── epsilonBarrierSet.hpp
│ ├── epsilonHeap.cpp
│ ├── epsilonHeap.hpp
│ ├── epsilonInitLogger.cpp
│ ├── epsilonInitLogger.hpp
│ ├── epsilonMemoryPool.cpp
│ ├── epsilonMemoryPool.hpp
│ ├── epsilonMonitoringSupport.cpp
│ ├── epsilonMonitoringSupport.hpp
│ ├── epsilonThreadLocalData.hpp
│ ├── epsilon_globals.hpp
│ └── vmStructs_epsilon.hpp
```



## 二、源码解读

由于篇幅关系，源码的头文件及部分源码文件会直接省略。如果需要深入研究，请自行下载JDK源码查阅。



### 2.1 epsilonArguments.cpp

__src/hotspot/share/gc/epsilon/epsilonArguments.cpp__

```cpp
// 根据支持大页标志位，获取页大小的配置值
size_t EpsilonArguments::conservative_max_heap_alignment() {
  // 若支持大页，则获取大页的数值，否则获取虚拟机支持的页大小
  return UseLargePages ? os::large_page_size() : os::vm_page_size();
}

// Epsilon初始化逻方法
void EpsilonArguments::initialize() {
  // 先初始化父类
  GCArguments::initialize();

  assert(UseEpsilonGC, "Sanity");

  // 因为虚拟机不支持回收内存空间，所以运行足够长时间后，必然会出现OOM
  // 而出现OOM时的响应逻辑，也只剩下抛出OOM错误这唯一选项
  if (FLAG_IS_DEFAULT(ExitOnOutOfMemoryError)) {
    FLAG_SET_DEFAULT(ExitOnOutOfMemoryError, true);
  }

  // TLAB，即Thread Local Allocation Buffer，线程私有空间。
  // 为了降低多线程同时从堆内存申请内存竞争锁的时间开销。TLAB每次从堆
  // 内存申请一块相对较大的空间，线程需要创建对象时先从TLAB获取空间。
  // 这里调整了TLAB从堆内存申请空间的大小
  if (EpsilonMaxTLABSize < MinTLABSize) {
    log_warning(gc)("EpsilonMaxTLABSize < MinTLABSize, adjusting it to " SIZE_FORMAT, MinTLABSize);
    EpsilonMaxTLABSize = MinTLABSize;
  }

  if (!EpsilonElasticTLAB && EpsilonElasticTLABDecay) {
    log_warning(gc)("Disabling EpsilonElasticTLABDecay because EpsilonElasticTLAB is disabled");
    FLAG_SET_DEFAULT(EpsilonElasticTLABDecay, false);
  }

#ifdef COMPILER2
  // Enable loop strip mining: there are still non-GC safepoints, no need to make it worse
  if (FLAG_IS_DEFAULT(UseCountedLoopSafepoints)) {
    FLAG_SET_DEFAULT(UseCountedLoopSafepoints, true);
    if (FLAG_IS_DEFAULT(LoopStripMiningIter)) {
      FLAG_SET_DEFAULT(LoopStripMiningIter, 1000);
    }
  }
#endif
}

// 内存对齐
void EpsilonArguments::initialize_alignments() {
  size_t page_size = UseLargePages ? os::large_page_size() : os::vm_page_size();
  size_t align = MAX2((size_t)os::vm_allocation_granularity(), page_size);
  SpaceAlignment = align;
  HeapAlignment  = align;
}

// 创建Epsilon的堆实例
CollectedHeap* EpsilonArguments::create_heap() {
  return new EpsilonHeap();
}

```



### epsilonHeap.cpp

上面最后一行创建的 __EpsilonHeap__ 实例，会通过以下逻辑进行初始化。

__src/hotspot/share/gc/epsilon/epsilonHeap.cpp__

```cpp
jint EpsilonHeap::initialize() {
  // 获取堆内存对齐大小
  size_t align = HeapAlignment;
  // 对齐起始堆空间的大小
  size_t init_byte_size = align_up(InitialHeapSize, align);
  // 对齐最大堆空间的大小
  size_t max_byte_size  = align_up(MaxHeapSize, align);

  // 根据堆最大空间，去申请内存的保留空间
  ReservedHeapSpace heap_rs = Universe::reserve_heap(max_byte_size, align);
  _virtual_space.initialize(heap_rs, init_byte_size);

  // 提交起始堆空间范围
  MemRegion committed_region((HeapWord*)_virtual_space.low(),          (HeapWord*)_virtual_space.high());

  // 并对申请的内存空间进行初始化
  initialize_reserved_region(heap_rs);

  _space = new ContiguousSpace();
  _space->initialize(committed_region, /* clear_space = */ true, /* mangle_space = */ true);

  // Precompute hot fields
  _max_tlab_size = MIN2(CollectedHeap::max_tlab_size(), align_object_size(EpsilonMaxTLABSize / HeapWordSize));
  _step_counter_update = MIN2<size_t>(max_byte_size / 16, EpsilonUpdateCountersStep);
  _step_heap_print = (EpsilonPrintHeapSteps == 0) ? SIZE_MAX : (max_byte_size / EpsilonPrintHeapSteps);
  _decay_time_ns = (int64_t) EpsilonTLABDecayTime * NANOSECS_PER_MILLISEC;

  // Enable monitoring
  _monitoring_support = new EpsilonMonitoringSupport(this);
  _last_counter_update = 0;
  _last_heap_print = 0;

  // 设置BarrierSet
  BarrierSet::set_barrier_set(new EpsilonBarrierSet());

  // 初始化完成，打印配置参数
  EpsilonInitLogger::print();

  // 堆空间及相关参数初始化完毕，返回JNI_OK
  return JNI_OK;
}

void EpsilonHeap::post_initialize() {
  CollectedHeap::post_initialize();
}

void EpsilonHeap::initialize_serviceability() {
  _pool = new EpsilonMemoryPool(this);
  _memory_manager.add_pool(_pool);
}

GrowableArray<GCMemoryManager*> EpsilonHeap::memory_managers() {
  GrowableArray<GCMemoryManager*> memory_managers(1);
  memory_managers.append(&_memory_manager);
  return memory_managers;
}

GrowableArray<MemoryPool*> EpsilonHeap::memory_pools() {
  GrowableArray<MemoryPool*> memory_pools(1);
  memory_pools.append(_pool);
  return memory_pools;
}

size_t EpsilonHeap::unsafe_max_tlab_alloc(Thread* thr) const {
  // Return max allocatable TLAB size, and let allocation path figure out
  // the actual allocation size. Note: result should be in bytes.
  return _max_tlab_size * HeapWordSize;
}

EpsilonHeap* EpsilonHeap::heap() {
  return named_heap<EpsilonHeap>(CollectedHeap::Epsilon);
}
```



以下是 __Epsilon__ 堆空间扩容

```Cpp
HeapWord* EpsilonHeap::allocate_work(size_t size, bool verbose) {
  assert(is_object_aligned(size), "Allocation size should be aligned: " SIZE_FORMAT, size);

  // 分配的内存起始地址
  HeapWord* res = NULL;
  
  // 循环尝试申请内存：可能成功获取size指定的内存块，或出现OOM错误退出
  while (true) {
    // 尝试从_space获取指定大小size的内存空间
    res = _space->par_allocate(size);
    if (res != NULL) {
      break;
    }

    // 上面申请内存失败，加锁后开始扩容
    {
      MutexLocker ml(Heap_lock);

      // Try to allocate under the lock, assume another thread was able to expand
      res = _space->par_allocate(size);
      if (res != NULL) {
        break;
      }

      // 计算剩余空间 和 需要的内存大小
      size_t space_left = max_capacity() - capacity();
      // 如果申请的size比EpsilonMinHeapExpand要小，则按照EpsilonMinHeapExpand
      size_t want_space = MAX2(size, EpsilonMinHeapExpand);

      if (want_space < space_left) {
        // 所需空间比剩余空间要小，可以分配
        bool expand = _virtual_space.expand_by(want_space);
        assert(expand, "Should be able to expand");
      } else if (size < space_left) {
        // No space to expand in bulk, and this allocation is still possible,
        // take all the remaining space:
        // 虽然按照EpsilonMinHeapExpand申请不到空间，但是按照更小的size依然有成功的可能
        // 所以继续按照size尝试扩容对空间
        bool expand = _virtual_space.expand_by(space_left);
        assert(expand, "Should be able to expand");
      } else {
        // 如果上述盛情两种size都无法实现，表示堆空间无法扩展提供内存，只能返回NULL
        return NULL;
      }

      // 到这里是成功申请内存，需要修改_space的高位地址表示成功扩容
      _space->set_end((HeapWord *) _virtual_space.high());
    }
  }

  // 计算已使用堆空间
  size_t used = _space->used();

  // 申请成功，更新计数器
  if (verbose) {
    size_t last = _last_counter_update;
    if ((used - last >= _step_counter_update) && Atomic::cmpxchg(&_last_counter_update, last, used) == last) {
      _monitoring_support->update_counters();
    }
  }

  // ...and print the occupancy line, if needed
  if (verbose) {
    size_t last = _last_heap_print;
    if ((used - last >= _step_heap_print) && Atomic::cmpxchg(&_last_heap_print, last, used) == last) {
      print_heap_info(used);
      print_metaspace_info();
    }
  }

  assert(is_object_aligned(res), "Object should be aligned: " PTR_FORMAT, p2i(res));
  
  // 返回申请的空间
  return res;
}
```



TLAB是和线程绑定的，所以申请TLAB需要线程上下文

```cpp
HeapWord* EpsilonHeap::allocate_new_tlab(size_t min_size,
                                         size_t requested_size,
                                         size_t* actual_size) {
  // 获取当前申请TLAB的线程信息
  Thread* thread = Thread::current();

  // Defaults in case elastic paths are not taken
  bool fits = true;
  size_t size = requested_size;
  size_t ergo_tlab = requested_size;
  int64_t time = 0;

  if (EpsilonElasticTLAB) {
    ergo_tlab = EpsilonThreadLocalData::ergo_tlab_size(thread);

    if (EpsilonElasticTLABDecay) {
      int64_t last_time = EpsilonThreadLocalData::last_tlab_time(thread);
      time = (int64_t) os::javaTimeNanos();

      assert(last_time <= time, "time should be monotonic");

      // If the thread had not allocated recently, retract the ergonomic size.
      // This conserves memory when the thread had initial burst of allocations,
      // and then started allocating only sporadically.
      if (last_time != 0 && (time - last_time > _decay_time_ns)) {
        ergo_tlab = 0;
        EpsilonThreadLocalData::set_ergo_tlab_size(thread, 0);
      }
    }

    // 当前TLAB大小可以满足当次内存申请，可以直接执行
    // 否则，要弹性增大TLAB的大小
    fits = (requested_size <= ergo_tlab);
    if (!fits) {
      size = (size_t) (ergo_tlab * EpsilonTLABElasticity);
    }
  }

  // Always honor boundaries
  size = clamp(size, min_size, _max_tlab_size);

  // Always honor alignment
  size = align_up(size, MinObjAlignment);

  // Check that adjustments did not break local and global invariants
  assert(is_object_aligned(size),
         "Size honors object alignment: " SIZE_FORMAT, size);
  assert(min_size <= size,
         "Size honors min size: "  SIZE_FORMAT " <= " SIZE_FORMAT, min_size, size);
  assert(size <= _max_tlab_size,
         "Size honors max size: "  SIZE_FORMAT " <= " SIZE_FORMAT, size, _max_tlab_size);
  assert(size <= CollectedHeap::max_tlab_size(),
         "Size honors global max size: "  SIZE_FORMAT " <= " SIZE_FORMAT, size, CollectedHeap::max_tlab_size());

  if (log_is_enabled(Trace, gc)) {
    ResourceMark rm;
    log_trace(gc)("TLAB size for \"%s\" (Requested: " SIZE_FORMAT "K, Min: " SIZE_FORMAT
                          "K, Max: " SIZE_FORMAT "K, Ergo: " SIZE_FORMAT "K) -> " SIZE_FORMAT "K",
                  thread->name(),
                  requested_size * HeapWordSize / K,
                  min_size * HeapWordSize / K,
                  _max_tlab_size * HeapWordSize / K,
                  ergo_tlab * HeapWordSize / K,
                  size * HeapWordSize / K);
  }

  // 完成准备工作，传入size去获取TLAB块
  HeapWord* res = allocate_work(size);

  // TLAB申请成功
  if (res != NULL) {
    // 记录内存size
    *actual_size = size;
    if (EpsilonElasticTLABDecay) {
      EpsilonThreadLocalData::set_last_tlab_time(thread, time);
    }
    if (EpsilonElasticTLAB && !fits) {
      // If we requested expansion, this is our new ergonomic TLAB size
      EpsilonThreadLocalData::set_ergo_tlab_size(thread, size);
    }
  } else {
    // Allocation failed, reset ergonomics to try and fit smaller TLABs
    if (EpsilonElasticTLAB) {
      EpsilonThreadLocalData::set_ergo_tlab_size(thread, 0);
    }
  }

  // 返回TLAB专属内存空间
  return res;
}

HeapWord* EpsilonHeap::mem_allocate(size_t size, bool *gc_overhead_limit_was_exceeded) {
  *gc_overhead_limit_was_exceeded = false;
  return allocate_work(size);
}

HeapWord* EpsilonHeap::allocate_loaded_archive_space(size_t size) {
  // Cannot use verbose=true because Metaspace is not initialized
  return allocate_work(size, /* verbose = */false);
}

// 触发收集对空间的垃圾回收
// 但是从上文知道，Epsilon永远不会执行垃圾回收操作，所以这个方法也仅仅打印日志就结束了
void EpsilonHeap::collect(GCCause::Cause cause) {
  switch (cause) {
    case GCCause::_metadata_GC_threshold:
    case GCCause::_metadata_GC_clear_soft_refs:
      // Receiving these causes means the VM itself entered the safepoint for metadata collection.
      // While Epsilon does not do GC, it has to perform sizing adjustments, otherwise we would
      // re-enter the safepoint again very soon.

      assert(SafepointSynchronize::is_at_safepoint(), "Expected at safepoint");
      log_info(gc)("GC request for \"%s\" is handled", GCCause::to_string(cause));
      MetaspaceGC::compute_new_size();
      print_metaspace_info();
      break;
    default:
      log_info(gc)("GC request for \"%s\" is ignored", GCCause::to_string(cause));
  }
  _monitoring_support->update_counters();
}

void EpsilonHeap::do_full_collection(bool clear_all_soft_refs) {
  collect(gc_cause());
}

// 遍历对空间内的所有对象
void EpsilonHeap::object_iterate(ObjectClosure *cl) {
  _space->object_iterate(cl);
}

void EpsilonHeap::print_on(outputStream *st) const {
  st->print_cr("Epsilon Heap");

  _virtual_space.print_on(st);

  if (_space != NULL) {
    st->print_cr("Allocation space:");
    _space->print_on(st);
  }

  MetaspaceUtils::print_on(st);
}

bool EpsilonHeap::print_location(outputStream* st, void* addr) const {
  return BlockLocationPrinter<EpsilonHeap>::print_location(st, addr);
}

void EpsilonHeap::print_tracing_info() const {
  print_heap_info(used());
  print_metaspace_info();
}

void EpsilonHeap::print_heap_info(size_t used) const {
  size_t reserved  = max_capacity();
  size_t committed = capacity();

  if (reserved != 0) {
    log_info(gc)("Heap: " SIZE_FORMAT "%s reserved, " SIZE_FORMAT "%s (%.2f%%) committed, "
                 SIZE_FORMAT "%s (%.2f%%) used",
            byte_size_in_proper_unit(reserved),  proper_unit_for_byte_size(reserved),
            byte_size_in_proper_unit(committed), proper_unit_for_byte_size(committed),
            committed * 100.0 / reserved,
            byte_size_in_proper_unit(used),      proper_unit_for_byte_size(used),
            used * 100.0 / reserved);
  } else {
    log_info(gc)("Heap: no reliable data");
  }
}

void EpsilonHeap::print_metaspace_info() const {
  MetaspaceCombinedStats stats = MetaspaceUtils::get_combined_statistics();
  size_t reserved  = stats.reserved();
  size_t committed = stats.committed();
  size_t used      = stats.used();

  if (reserved != 0) {
    log_info(gc, metaspace)("Metaspace: " SIZE_FORMAT "%s reserved, " SIZE_FORMAT "%s (%.2f%%) committed, "
                            SIZE_FORMAT "%s (%.2f%%) used",
            byte_size_in_proper_unit(reserved),  proper_unit_for_byte_size(reserved),
            byte_size_in_proper_unit(committed), proper_unit_for_byte_size(committed),
            committed * 100.0 / reserved,
            byte_size_in_proper_unit(used),      proper_unit_for_byte_size(used),
            used * 100.0 / reserved);
  } else {
    log_info(gc, metaspace)("Metaspace: no reliable data");
  }
}
```



### epsilonMemoryPool.cpp

__src/hotspot/share/gc/epsilon/epsilonMemoryPool.cpp__

```cpp
/*
 * Copyright (c) 2017, 2018, Red Hat, Inc. All rights reserved.
 * DO NOT ALTER OR REMOVE COPYRIGHT NOTICES OR THIS FILE HEADER.
 *
 * This code is free software; you can redistribute it and/or modify it
 * under the terms of the GNU General Public License version 2 only, as
 * published by the Free Software Foundation.
 *
 * This code is distributed in the hope that it will be useful, but WITHOUT
 * ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
 * FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License
 * version 2 for more details (a copy is included in the LICENSE file that
 * accompanied this code).
 *
 * You should have received a copy of the GNU General Public License version
 * 2 along with this work; if not, write to the Free Software Foundation,
 * Inc., 51 Franklin St, Fifth Floor, Boston, MA 02110-1301 USA.
 *
 * Please contact Oracle, 500 Oracle Parkway, Redwood Shores, CA 94065 USA
 * or visit www.oracle.com if you need additional information or have any
 * questions.
 *
 */

#include "precompiled.hpp"
#include "gc/epsilon/epsilonHeap.hpp"
#include "gc/epsilon/epsilonMemoryPool.hpp"

EpsilonMemoryPool::EpsilonMemoryPool(EpsilonHeap* heap) :
        CollectedMemoryPool("Epsilon Heap",
                            heap->capacity(),
                            heap->max_capacity(),
                            false),
        _heap(heap) {
  assert(UseEpsilonGC, "sanity");
}

MemoryUsage EpsilonMemoryPool::get_memory_usage() {
  size_t initial_sz = initial_size();
  size_t max_sz     = max_size();
  size_t used       = used_in_bytes();
  size_t committed  = committed_in_bytes();

  return MemoryUsage(initial_sz, used, committed, max_sz);
}
```



src/hotspot/share/gc/epsilon/epsilonMonitoringSupport.cpp

```cpp
/*
 * Copyright (c) 2017, 2018, Red Hat, Inc. All rights reserved.
 * DO NOT ALTER OR REMOVE COPYRIGHT NOTICES OR THIS FILE HEADER.
 *
 * This code is free software; you can redistribute it and/or modify it
 * under the terms of the GNU General Public License version 2 only, as
 * published by the Free Software Foundation.
 *
 * This code is distributed in the hope that it will be useful, but WITHOUT
 * ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
 * FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License
 * version 2 for more details (a copy is included in the LICENSE file that
 * accompanied this code).
 *
 * You should have received a copy of the GNU General Public License version
 * 2 along with this work; if not, write to the Free Software Foundation,
 * Inc., 51 Franklin St, Fifth Floor, Boston, MA 02110-1301 USA.
 *
 * Please contact Oracle, 500 Oracle Parkway, Redwood Shores, CA 94065 USA
 * or visit www.oracle.com if you need additional information or have any
 * questions.
 *
 */

#include "precompiled.hpp"
#include "gc/epsilon/epsilonMonitoringSupport.hpp"
#include "gc/epsilon/epsilonHeap.hpp"
#include "gc/shared/generationCounters.hpp"
#include "memory/allocation.hpp"
#include "memory/metaspaceCounters.hpp"
#include "memory/resourceArea.hpp"
#include "runtime/perfData.hpp"
#include "services/memoryService.hpp"

class EpsilonSpaceCounters: public CHeapObj<mtGC> {
  friend class VMStructs;

private:
  PerfVariable* _capacity;
  PerfVariable* _used;
  char*         _name_space;

public:
  EpsilonSpaceCounters(const char* name,
                 int ordinal,
                 size_t max_size,
                 size_t initial_capacity,
                 GenerationCounters* gc) {
    if (UsePerfData) {
      EXCEPTION_MARK;
      ResourceMark rm;

      const char* cns = PerfDataManager::name_space(gc->name_space(), "space", ordinal);

      _name_space = NEW_C_HEAP_ARRAY(char, strlen(cns)+1, mtGC);
      strcpy(_name_space, cns);

      const char* cname = PerfDataManager::counter_name(_name_space, "name");
      PerfDataManager::create_string_constant(SUN_GC, cname, name, CHECK);

      cname = PerfDataManager::counter_name(_name_space, "maxCapacity");
      PerfDataManager::create_constant(SUN_GC, cname, PerfData::U_Bytes, (jlong)max_size, CHECK);

      cname = PerfDataManager::counter_name(_name_space, "capacity");
      _capacity = PerfDataManager::create_variable(SUN_GC, cname, PerfData::U_Bytes, initial_capacity, CHECK);

      cname = PerfDataManager::counter_name(_name_space, "used");
      _used = PerfDataManager::create_variable(SUN_GC, cname, PerfData::U_Bytes, (jlong) 0, CHECK);

      cname = PerfDataManager::counter_name(_name_space, "initCapacity");
      PerfDataManager::create_constant(SUN_GC, cname, PerfData::U_Bytes, initial_capacity, CHECK);
    }
  }

  ~EpsilonSpaceCounters() {
    FREE_C_HEAP_ARRAY(char, _name_space);
  }

  inline void update_all(size_t capacity, size_t used) {
    _capacity->set_value(capacity);
    _used->set_value(used);
  }
};

class EpsilonGenerationCounters : public GenerationCounters {
private:
  EpsilonHeap* _heap;
public:
  EpsilonGenerationCounters(EpsilonHeap* heap) :
          GenerationCounters("Heap", 1, 1, 0, heap->max_capacity(), heap->capacity()),
          _heap(heap)
  {};

  virtual void update_all() {
    _current_size->set_value(_heap->capacity());
  }
};

EpsilonMonitoringSupport::EpsilonMonitoringSupport(EpsilonHeap* heap) {
  _heap_counters  = new EpsilonGenerationCounters(heap);
  _space_counters = new EpsilonSpaceCounters("Heap", 0, heap->max_capacity(), 0, _heap_counters);
}

void EpsilonMonitoringSupport::update_counters() {
  MemoryService::track_memory_usage();

  if (UsePerfData) {
    EpsilonHeap* heap = EpsilonHeap::heap();
    size_t used = heap->used();
    size_t capacity = heap->capacity();
    _heap_counters->update_all();
    _space_counters->update_all(capacity, used);
    MetaspaceCounters::update_performance_counters();
  }
}
```



## 三、链接

- [JEP 318: Epsilon: A No-Op Garbage Collector (Experimental)](https://openjdk.java.net/jeps/318)

