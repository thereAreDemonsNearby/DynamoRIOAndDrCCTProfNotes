# DynamoRIO介绍

## 基本介绍

* 了解DynamoRIO的必要性：DrCCTProf以DynamoRIO的插件的形式存在，与DynamoRIO中的其他插件（drmgr、drreg等）是平级的，要想使用DrCCTProf必须使用DynamoRIO API。

* 名称由来：Dynamo + RIO (Runtime Introspection and Optimization)

  <img src="/home/shiletong/code/tianhe3/dynamorio_struct.png" alt="dynamorio_struct" style="zoom:75%;" />

* 以basic block为单位
* 为了提升效率，引入了basic block cache和trace cache。每个basic block的代码执行之前，都会被加入basic block cache，此后执行该basic block中的代码时都从这个cache中获取。即 一个basic block只会被加入cache一次（除非被手动从cache中删除）。

## 使用

* event based，意味着大量使用回调函数（callback, hook）
  * 可以在回调函数中做插桩、修改指令等

```c++
static int global_count;

static void bb_event(void *drcontext, void *tag, instrlist_t *bb, bool for_trace, bool translating) {}
static void client_exit(void) {}
static void client_init(int argc, char const** argv) {}

DR_EXPORT void
dr_client_main(client_id_t id, int argc, const char *argv[])
{
    dr_register_exit_event(client_exit);
    dr_register_bb_event(bb_event);
    client_init(argc, argv);  
}
```

* client编译为动态链接库（so, dll），调用：`drrun -c client.so -- a.out`

* 指令表示：`instr_t`

* 基本块表示：`instrlist_t`，是`instr_t`的链表

* 回调的时机：基本块进入basic block cache或者进入trace cache时

  * `dr_register_bb_event`和`dr_register_trace_event`

  * 意味着回调函数被调用的机会只有一次

  * ```c++
    dr_emit_flags_t callback(void* drcontext, void* tag, instrlist_t* bb
                             bool for_trace, bool translating)
    {
        for (instr_t* ins = instrlist_first(bb); ins != NULL; 
             instr = instr_get_next(instr)) {
             // 插入新指令（插桩）或者修改当前指令
         }
    }
    ```



指令、基本块相关的操作：

* 判断当前指令类型，如`instr_is_call_direct`、`instr_is_floating`，或者直接获取操作码来判断：`instr_get_opcode(instr) == OP_jmp`
* 获取操作数或者结果寄存器：`instr_get_src`、`instr_get_dst`等
* 新指令创建：`instr_create_*`一系列函数，还有一些宏
* 在基本块中插入、删除、替换指令等
  * `instrlist_preinsert` `instrlist_remove`  `instrlist_replace`等
* 在基本块中插入函数调用，从而方便地完成插桩：`dr_insert_clean_call`，该函数调用实现在DynamoRIO的内存空间中

指令分为两种

* 应用指令（app）
* 元指令（meta）。元指令被认为不属于原程序的一部分，而是插桩的一部分，仅起到观察作用（observational）。
  * 通过`instrlist_meta_prelist`等指令可插入元指令
  * 通过`instr_is_meta`判断是否为元指令
  * `instrlist_first_app`等以app为后缀的指令不会访问到元指令
  * 添加或修改了非元指令的指令后，需要调用`instr_set_translation`
  * 前面提到的clean call也是元指令

多线程应用中需要注意加锁、thread local storage的应用等

重要扩展drmgr，四阶段

# DrCCTProf的介绍

DrCCTProf被开发为DynamoRIO的插件（extension）drcctlib，主要的意义是在DynamoRIO的基础上增加了获取调用栈（calling context，cct）的功能。

## 简单使用：

```c++
DR_EXPORT
bool
drcctlib_init_ex(bool (*filter)(instr_t *), file_t file,
                 void (*func1)(void *, instr_instrument_msg_t *),
                 void (*func2)(void *, int32_t, int32_t),
                 void (*func3)(void *, context_handle_t, int32_t, int32_t,
                               mem_ref_msg_t *, void **),
                 char flag);

DR_EXPORT
void
drcctlib_init(bool (*filter)(instr_t *), file_t file,
              void (*func1)(void *, instr_instrument_msg_t *), bool do_data_centric);
```

其中`func1` `func2` `func3`都是传递给DynamoRIO相关API的回调函数（即在基本块进入basic block cache时调用一次），在这些函数中可以发起一些修改或插桩。`func1` `func2` `func3`被保存在全局的结构体`global_client_cb`中，待合适的时候被调用。

以func1为例，其调用链：

* `drcctlib_init` -> `drcctlib_internal_init`
* `drcctlib_internal_init`将`drcctlib_event_bb_analysis`通过`drmgr_register_bb_instrumentation_event`注册为回调函数
* `drcctlib_event_bb_analysis`对基本块内每一条**“应该被插桩”**的指令，都调用`drcctlib_event_pre_instr`
* `drcctlib_event_pre_instr` -> `instr_instrument_client_cb` -> `global_client_cb.func_instr_analysis`
* `global_client_cb.func_instr_analysis`即传入的`func1`，可以在里面调用一些插桩函数，如`dr_insert_clean_call`等

**“应该被插桩”**的指令：

* call（直接、间接），return
* 用户感兴趣的指令
  *  通过`drtcctlib_init*`中的`filter`传入
  * 保存于`global_instr_filter`



`func1`的函数签名：

```c++
void (void* drcontext, instr_instrument_msg_t*);

typedef struct _instr_instrument_msg_t {
    instrlist_t *bb;
    instr_t *instr;
    bool interest_start;
    int32_t slot;
    int32_t state;
    struct _instr_instrument_msg_t *next;
} instr_instrument_msg_t;
```

## CCT相关

```c++
DR_EXPORT
context_handle_t
drcctlib_get_context_handle(void *drcontext, int32_t slot);

DR_EXPORT
context_t *
drcctlib_get_cct(context_handle_t ctxt_hndl, int max_depth);

DR_EXPORT
void
drcctlib_free_cct(context_t *contxt_list);

DR_EXPORT
context_t *
drcctlib_get_full_cct(context_handle_t ctxt_hndl);

DR_EXPORT
context_t *
drcctlib_get_full_cct(context_handle_t ctxt_hndl, int max_depth);

DR_EXPORT
void
drcctlib_free_full_cct(context_t *contxt_list);
```

一般的使用流程：

* 回调函数（前面提到的`func1`）的参数里面有`slot`，通过`drcctlib_get_context_handle`可以得到`context_handle_t`
  * `drcctlib_get_context_handle`使用参数`drcontext`可以得知当前的基本块，进而通过slot定位到`context_handle_t`
* 使用`context_handle_t`，可以得到表示calling context的`context_t`

`context_handle_t`、`context_t`和`slot`有对应关系

* `context_handle_t`和`context_t`存在一个一一对应关系，都代表某个函数中的某条指令。
* slot表示基本块中被插桩的指令的编号

<img src="/home/shiletong/code/tianhe3/do_something.png" alt="do_something"  />

<img src="/home/shiletong/code/tianhe3/do_something_bb.png" alt="do_something_bb" style="zoom:75%;" />

