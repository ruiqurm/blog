+++
title = "Rust for Linux笔记-2"
date = 2024-11-04T12:17:00+08:00
tags = ["rust-for-linux", "linux"]
categories = ["notes"]
draft = false
+++

今天看看rust-next分支下面的几个宏。目前写驱动用得比较多的应该时前面三个。其他的偏向于写宏时候的
工具。


## module {#module}

```rust
use kernel::prelude::*;
module!{
    type: MyDeviceDriverModule,
    name: "my_device_driver_module",
    author: "Rust for Linux Contributors",
    description: "My device driver requires firmware",
    license: "GPL",
    firmware: ["my_device_firmware1.bin", "my_device_firmware2.bin"],
}
struct MyDeviceDriverModule;
```

这个宏会按照一定的模板去生成rust代码。


### 生成\__init和__exit函数 {#生成-init和-exit函数}

模块初始化时，会类型linux驱动调用init和exit(或者register和unregister)。这里rfl封装了一层，
不是直接调用用户提供的函数，而是去调用impl了Module trait提供的初始化函数，初始化完成后，将
其绑定到一个静态全局变量上。

这里还考虑了如果CONFIG(Module)没开的情况（下面没贴出来）

```rust
static mut __MOD: Option<{type_}> = None;

#[no_mangle]
#[link_section = \".init.text\"]
pub unsafe extern \"C\" fn init_module() -> core::ffi::c_int {{
    // SAFETY: This function is inaccessible to the outside due to the double
    // module wrapping it. It is called exactly once by the C side via its
    // unique name.
    unsafe {{ __init() }}
}}

#[used]
#[link_section = \".init.data\"]
static __UNIQUE_ID___addressable_init_module: unsafe extern \"C\" fn() -> i32 = init_module;


unsafe fn __init() -> core::ffi::c_int {{
    match <{type_} as kernel::Module>::init(&super::super::THIS_MODULE) {{
        Ok(m) => {{
            // SAFETY: No data race, since `__MOD` can only be accessed by this
            // module and there only `__init` and `__exit` access it. These
            // functions are only called once and `__exit` cannot be called
            // before or during `__init`.
            unsafe {{
                __MOD = Some(m);
            }}
            return 0;
        }}
        Err(e) => {{
            return e.to_errno();
        }}
    }}
}}
```

对于exit的情况,这里不是像C一样调用某个函数，而是等待drop函数自动调用。也就是用户需要为自己实现
的type实现drop。最后会将\__MOD置为None.


### 生成mod_info {#生成mod-info}

在模板的最后面是一些mod_info。目前支持author, descripten,license等等。

modinfo使用一个ModInfoBuilder去构建。最后的mod信息会保存到buffer中。

```rust
struct ModInfoBuilder<'a> {
    module: &'a str,
    counter: usize,
    buffer: String,
}
```

每次写入会调用emit_base，其基本模板如下：

```rust
{cfg}
#[doc(hidden)]
#[link_section = \".modinfo\"]
#[used]
pub static __{module}_{counter}: [u8; {length}] = *{string};
```

-   cfg是一个"#[cfg(not(MODULE))]" 或者"#[cfg(MODULE)]"
-   module为当前模块名
-   counter是一个会自增的int
-   string是一个如{field}={content}或者{module}.{field}={content}\\0的modinfo


## vtable {#vtable}

rust的trait和kernel的vtable十分相似，他们的区别之处在于二者对于未实现函数的表示。Rust中是提
供默认实现（一般是返回Error::EINVAL），而linux则返回NULL函数指针。

这个宏的主要作用是为rust trait生成HAS_XXX的宏。对于trait，所有的HAS_\*都为false；对于impl，
实现了的function为定义其HAS_\*变量为true。


### OperationsVTable {#operationsvtable}

block/mq中实现了一个OperationVTable的例子。

```rust
#[macros::vtable]
pub trait Operations: Sized {
    /// Called by the kernel to queue a request with the driver. If `is_last` is
    /// `false`, the driver is allowed to defer committing the request.
    fn queue_rq(rq: ARef<Request<Self>>, is_last: bool) -> Result;

    /// Called by the kernel to indicate that queued requests should be submitted.
    fn commit_rqs();

    /// Called by the kernel to poll the device for completed requests. Only
    /// used for poll queues.
    fn poll() -> bool {
        crate::build_error(crate::error::VTABLE_DEFAULT_ERROR)
    }
}

pub(crate) struct OperationsVTable<T: Operations>(PhantomData<T>);
impl<T: Operations> OperationsVTable<T> {
 // ...
    unsafe extern "C" fn commit_rqs_callback(_hctx: *mut bindings::blk_mq_hw_ctx) {
        T::commit_rqs()
    }
}
```

Operations是rust用户需要实现的trait，而下面的VTable是rfl生成的用于和内核沟通的抽象层。

Vtable中实现了许多\*_callback风格的函数，这是让c侧调用的函数。这些callback函数还负责将c指针类
型转为rust的安全抽象，最后去调用用户写的rust trait函数。

impl中还有一个VTABLE变量。这个变量将上面的callback存到FFI的blk_mq_ops结构体中，该结构体是
由Rust bindgen产生的。

```rust
impl<T: Operations> OperationsVTable<T> {
   // ...
   const VTABLE: bindings::blk_mq_ops = bindings::blk_mq_ops {
        queue_rq: Some(Self::queue_rq_callback),
        queue_rqs: None,
        commit_rqs: Some(Self::commit_rqs_callback),
        get_budget: None,
        put_budget: None,
        set_rq_budget_token: None,
        get_rq_budget_token: None,
        timeout: None,
        poll: if T::HAS_POLL {
            Some(Self::poll_callback)
        } else {
            None
        },
        complete: Some(Self::complete_callback),
        init_hctx: Some(Self::init_hctx_callback),
        exit_hctx: Some(Self::exit_hctx_callback),
        init_request: Some(Self::init_request_callback),
        exit_request: Some(Self::exit_request_callback),
        cleanup_rq: None,
        busy: None,
        map_queues: None,
        #[cfg(CONFIG_BLK_DEBUG_FS)]
        show_rq: None,
    };
}

/// FFI interface
#[repr(C)]
#[derive(Default, Copy, Clone)]
pub struct blk_mq_ops {
    pub queue_rq: ::core::option::Option<
        unsafe extern "C" fn(
            arg1: *mut blk_mq_hw_ctx,
            arg2: *const blk_mq_queue_data,
        ) -> blk_status_t,
    >,
    pub commit_rqs: ::core::option::Option<unsafe extern "C" fn(arg1: *mut blk_mq_hw_ctx)>,
    pub queue_rqs: ::core::option::Option<unsafe extern "C" fn(rqlist: *mut *mut request)>,
    pub get_budget:
        ::core::option::Option<unsafe extern "C" fn(arg1: *mut request_queue) -> core::ffi::c_int>,
    pub put_budget: ::core::option::Option<
        unsafe extern "C" fn(arg1: *mut request_queue, arg2: core::ffi::c_int),
    >,
    pub set_rq_budget_token:
        ::core::option::Option<unsafe extern "C" fn(arg1: *mut request, arg2: core::ffi::c_int)>,
    pub get_rq_budget_token:
        ::core::option::Option<unsafe extern "C" fn(arg1: *mut request) -> core::ffi::c_int>,
    pub timeout:
        ::core::option::Option<unsafe extern "C" fn(arg1: *mut request) -> blk_eh_timer_return>,
    pub poll: ::core::option::Option<
        unsafe extern "C" fn(
            arg1: *mut blk_mq_hw_ctx,
            arg2: *mut io_comp_batch,
        ) -> core::ffi::c_int,
    >,
    pub complete: ::core::option::Option<unsafe extern "C" fn(arg1: *mut request)>,
    pub init_hctx: ::core::option::Option<
        unsafe extern "C" fn(
            arg1: *mut blk_mq_hw_ctx,
            arg2: *mut core::ffi::c_void,
            arg3: core::ffi::c_uint,
        ) -> core::ffi::c_int,
    >,
    pub exit_hctx: ::core::option::Option<
        unsafe extern "C" fn(arg1: *mut blk_mq_hw_ctx, arg2: core::ffi::c_uint),
    >,
    pub init_request: ::core::option::Option<
        unsafe extern "C" fn(
            set: *mut blk_mq_tag_set,
            arg1: *mut request,
            arg2: core::ffi::c_uint,
            arg3: core::ffi::c_uint,
        ) -> core::ffi::c_int,
    >,
    pub exit_request: ::core::option::Option<
        unsafe extern "C" fn(set: *mut blk_mq_tag_set, arg1: *mut request, arg2: core::ffi::c_uint),
    >,
    pub cleanup_rq: ::core::option::Option<unsafe extern "C" fn(arg1: *mut request)>,
    pub busy: ::core::option::Option<unsafe extern "C" fn(arg1: *mut request_queue) -> bool_>,
    pub map_queues: ::core::option::Option<unsafe extern "C" fn(set: *mut blk_mq_tag_set)>,
    pub show_rq: ::core::option::Option<unsafe extern "C" fn(m: *mut seq_file, rq: *mut request)>,
}
```

使用时，将VTABLE传给c一侧的结构体初始化。之后就是C通过FFI接口调用rust了。

```rust
ops: OperationsVTable::<T>::build()
```


## pin_data {#pin-data}

文档中说和[pin-project-lite](https://crates.io/crates/pin-project-lite)比较类似。需要这个结构时使用#[pin_data]修饰整个结构体，对于特定的
field，使用#[pin]去修饰field


## paste 和 concat {#paste-和-concat}


### paste {#paste}

提供了一个固定的写法用于实现宏中的复制功能。文档中主要举了生成相同前缀宏和生成相同内容函数的两个
例子（手动泛型..）


### concat {#concat}

这个也是个宏工具，提供的是拼接的功能。文档中指出生成前缀也可以拼接方式实现

这个方式应该比较c-style
