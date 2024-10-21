[合集 \- 算法(7\)](https://github.com)[1\.TimerWheel(计时轮)在Rust中的实现及源码解析06\-12](https://github.com/wmproxy/p/18243240/TimerWheel)[2\.Rust性能分析之测试及火焰图，附(lru,lfu,arc)测试06\-18](https://github.com/wmproxy/p/18253640)[3\.Lru\-k在Rust中的实现及源码解析06\-21](https://github.com/wmproxy/p/18260014)[4\.带有ttl的Lru在Rust中的实现及源码解析06\-24](https://github.com/wmproxy/p/18264295):[westworld加速](https://tianchuang88.com)[5\.Lfu缓存在Rust中的实现及源码解析06\-27](https://github.com/wmproxy/p/18270304)[6\.Rust宏之derive的设计及实战10\-18](https://github.com/wmproxy/p/18473898)7\.在Lua中实现Rust对象的绑定10\-21收起
**实现目标**：能将Rust对象快速的映射到lua中使用，尽可能的简化使用。


## 


## 功能目标


以`struct HcTestMacro`为例：


1. **类型构建**，在lua调用`local val = HcTestMacro.new()`可构建
2. **类型析构**，在lua调用`HcTestMacro.del(val)`可析建，仅限`light use**rdata`
3. **字段的映射**，假设有字段`hc`，我们需要能快速的进行字段的取值赋值


* **取值**：`val.hc`或者`val:get_hc()`均可进行取值
* **赋值**：`val.hc = "hclua"`或者`val:set_hc("hclua")`均可进行取值


4. **类型方法**，注册类方法，比如额外的方法`call1`，那我们就可以通过注册到lua虚拟机，由于lua虚拟机可能不会是全局唯一的，所以不好通过宏直接注册



```


|  | // 直接注册函数注册 |
| --- | --- |
|  | HcTestMacro::object_def(&mut lua, "ok", hclua::function1(HcTestMacro::ok)); |
|  | // 闭包注册单参数 |
|  | HcTestMacro::object_def(&mut lua, "call1", hclua::function1(|obj: &HcTestMacro| -> u32 { |
|  | obj.field |
|  | })); |
|  | // 闭包注册双参数 |
|  | HcTestMacro::object_def(&mut lua, "call2", hclua::function2(|obj: &mut HcTestMacro, val: u32| -> u32 { |
|  | obj.field + val |
|  | })); |


```

5. **静态方法**，有些静态类方法，即不实际化对象进行注册可相当于模块



```


|  | HcTestMacro::object_static_def(&mut lua, "sta_run", hclua::function0(|| -> String { |
| --- | --- |
|  | "test".to_string() |
|  | })); |


```

#### 完整示列代码



```


|  | use hclua_macro::ObjectMacro; |
| --- | --- |
|  |  |
|  | #[derive(ObjectMacro, Default)] |
|  | #[hclua_cfg(name = HcTest)] |
|  | #[hclua_cfg(light)] |
|  | struct HcTestMacro { |
|  | #[hclua_field] |
|  | field: u32, |
|  | #[hclua_field] |
|  | hc: String, |
|  | } |
|  |  |
|  | impl HcTestMacro { |
|  | fn ok(&self) { |
|  | println!("ok!!!!"); |
|  | } |
|  | } |
|  |  |
|  |  |
|  | fn main() { |
|  | let mut lua = hclua::Lua::new(); |
|  | let mut test = HcTestMacro::default(); |
|  | HcTestMacro::register(&mut lua); |
|  | // 直接注册函数注册 |
|  | HcTestMacro::object_def(&mut lua, "ok", hclua::function1(HcTestMacro::ok)); |
|  | // 闭包注册单参数 |
|  | HcTestMacro::object_def(&mut lua, "call1", hclua::function1(|obj: &HcTestMacro| -> u32 { |
|  | obj.field |
|  | })); |
|  | // 闭包注册双参数 |
|  | HcTestMacro::object_def(&mut lua, "call2", hclua::function2(|obj: &mut HcTestMacro, val: u32| -> u32 { |
|  | obj.field + val |
|  | })); |
|  | HcTestMacro::object_static_def(&mut lua, "sta_run", hclua::function0(|| -> String { |
|  | "test".to_string() |
|  | })); |
|  | lua.openlibs(); |
|  |  |
|  | let val = " |
|  | print(aaa); |
|  | print(\"cccxxxxxxxxxxxxxxx\"); |
|  | print(type(HcTest)); |
|  | local v = HcTest.new(); |
|  | print(\"call ok\", v:ok()) |
|  | print(\"call1\", v:call1()) |
|  | print(\"call2\", v:call2(2)) |
|  | print(\"kkkk\", v.hc) |
|  | v.hc = \"dddsss\"; |
|  | print(\"kkkk ok get_hc\", v:get_hc()) |
|  | v.hc = \"aa\"; |
|  | print(\"new kkkkk\", v.hc) |
|  | v:set_hc(\"dddddd\"); |
|  | print(\"new kkkkk1\", v.hc) |
|  | print(\"attemp\", v.hc1) |
|  | print(\"vvvvv\", v:call1()) |
|  | print(\"static run\", HcTest.sta_run()) |
|  | HcTest.del(v); |
|  | "; |
|  | let _: Option<()> = lua.exec_string(val); |
|  | } |


```

## 源码地址


[`hclua`](https://github.com) Rust中的lua绑定。


## 功能实现剥析


通过derive宏进行函数注册:`#[derive(ObjectMacro, Default)]`
通过attrib声明命名：`#[hclua_cfg(name = HcTest)]`，配置该类在lua
中的名字为`HcTest`，本质上在lua里注册全局的table，通过在该table下注册
`HcTest { new = function(), del = function() }`
通过attrib注册生命：`#[hclua_cfg(light)]`，表示该类型是`light userdata`即生命周期由Rust控制，默认为`userdata`即生命周期由Lua控制，通过`__gc`进行回收。
通过attrib声明字段：`#[hclua_field]`放到字段前面，即可以注册字段使用，在derive生成的时候判断是否有该字段，进行字段的映射。


## derive宏实现


主要源码在 [hclua\-macro](https://github.com) 实现， 完整代码可进行参考。


1. **声明并解析ItemStruct**



```


|  | #[proc_macro_derive(ObjectMacro, attributes(hclua_field, hclua_cfg))] |
| --- | --- |
|  | pub fn object_macro_derive(input: TokenStream) -> TokenStream { |
|  | let ItemStruct { |
|  | ident, |
|  | fields, |
|  | attrs, |
|  | .. |
|  | } = parse_macro_input!(input); |


```

2. **解析Config，即判断类名及是否light**



```


|  | let config = config::Config::parse_from_attributes(ident.to_string(), &attrs[..]).unwrap(); |
| --- | --- |


```

3. **解析字段并生成相应的函数**



```


|  | let functions: Vec<_> = fields |
| --- | --- |
|  | .iter() |
|  | .map(|field| { |
|  | let field_ident = field.ident.clone().unwrap(); |
|  | if field.attrs.iter().any(|attr| attr.path().is_ident("hclua_field")) { |
|  | let get_name = format_ident!("get_{}", field_ident); |
|  | let set_name = format_ident!("set_{}", field_ident); |
|  | let ty = field.ty.clone(); |
|  | quote! { |
|  | fn #get_name(&mut self) -> &#ty { |
|  | &self.#field_ident |
|  | } |
|  |  |
|  | fn #set_name(&mut self, val: #ty) { |
|  | self.#field_ident = val; |
|  | } |
|  | } |
|  | } else { |
|  | quote! {} |
|  | } |
|  | }) |
|  | .collect(); |
|  |  |
|  | let registers: Vec<_> = fields.iter().map(|field| { |
|  | let field_ident = field.ident.clone().unwrap(); |
|  | if field.attrs.iter().any(|attr| attr.path().is_ident("hclua_field")) { |
|  | let ty = field.ty.clone(); |
|  | let get_name = format_ident!("get_{}", field_ident); |
|  | let set_name = format_ident!("set_{}", field_ident); |
|  | quote!{ |
|  | hclua::LuaObject::add_object_method_get(lua, &stringify!(#field_ident), hclua::function1(|obj: &mut #ident| -> &#ty { |
|  | &obj.#field_ident |
|  | })); |
|  | // ... |
|  | } |
|  | } else { |
|  | quote!{} |
|  | } |
|  | }).collect(); |


```


> 通过生成TokenStream数组，在最终的时候进行源码展开`#(#functions)*`即可以得到我们的TokenStream拼接的效果。


4. **生成最终的代码**



```


|  | let name = config.name; |
| --- | --- |
|  | let is_light = config.light; |
|  | let gen = quote! { |
|  | impl #ident { |
|  | fn register_field(lua: &mut hclua::Lua) { |
|  | #(#registers)* |
|  | } |
|  |  |
|  | fn register(lua: &mut hclua::Lua) { |
|  | let mut obj = if #is_light { |
|  | hclua::LuaObject::<#ident>::new_light(lua.state(), &#name) |
|  | } else { |
|  | hclua::LuaObject::<#ident>::new(lua.state(), &#name) |
|  | }; |
|  | obj.create(); |
|  |  |
|  | Self::register_field(lua); |
|  | } |
|  |  |
|  | fn object_def(lua: &mut hclua::Lua, name: &str, param: P) |
|  | where |
|  | P: hclua::LuaPush, |
|  | { |
|  | hclua::LuaObject::<#ident>::object_def(lua, name, param); |
|  | } |
|  |  |
|  | #(#functions)* |
|  | } |
|  | // ... |
|  | }; |
|  | gen.into() |


```

这样子我们通过宏就实现了我们快速的实现方案。


## Field映射的实现


Lua对象映射中，`type(val)`为一个object变量，在这基础上进行访问的都将会触发元表的操作`metatable`


#### Field的获取


我们访问任何对象如`val.hc`：


1. 查找val中是否有hc的值，若存在直接返回
2. 查找object中对应的元表`lua_getmetatable`若为meta
3. 找到`__index`的key值，若不存在则返回空值
4. 调用`__index`函数，此时调用该数第一个参数为`val`，第二个参数为`hc`
5. 此时有两种可能，一种是访问函数跳转6，一种是访问变量跳转7，
6. 将直接取出meta\["hc"]返回给lua，如果是值即为值，为函数则返回给lua的后续调用，通常的形式表达为`val:hc()`即`val.hc(val)`实现调用，结束流程
7. 因为变量是一个动态值，我们并未存在metatable中，所以需要额外的调用取出正确值，我们将取出的函数手动继续在调用`lua_call(lua, 1, 1);`即可以实现字段的返回



> 注：在变量中该值是否为字段处理过程会有相对的差别，又需要高效的进行验证，这里用的是全局的静态变量来存储是否为该类型的字段值。



```


|  | lazy_static! { |
| --- | --- |
|  | static ref FIELD_CHECK: RwLock'static str)>> = RwLock::new(HashSet::new()); |
|  | } |


```

完整源码：



```


|  | extern "C" fn index_metatable(lua: *mut sys::lua_State) -> libc::c_int { |
| --- | --- |
|  | unsafe { |
|  | if lua_gettop(lua) < 2 { |
|  | let value = CString::new(format!("index field must use 2 top")).unwrap(); |
|  | return luaL_error(lua, value.as_ptr()); |
|  | } |
|  | } |
|  | if let Some(key) = String::lua_read_with_pop(lua, 2, 0) { |
|  | let typeid = Self::get_metatable_real_key(); |
|  | unsafe { |
|  | sys::lua_getglobal(lua, typeid.as_ptr()); |
|  | let is_field = LuaObject::is_field(&*key); |
|  | let key = CString::new(key).unwrap(); |
|  | let t = lua_getfield(lua, -1, key.as_ptr()); |
|  | if !is_field { |
|  | if t == sys::LUA_TFUNCTION { |
|  | return 1; |
|  | } else { |
|  | return 1; |
|  | } |
|  | } |
|  | lua_pushvalue(lua, 1); |
|  | lua_call(lua, 1, 1); |
|  | 1 |
|  | } |
|  | } else { |
|  | 0 |
|  | } |
|  | } |


```

此时字段的获取已经完成了。


#### Field的设置


此时我们需要设置对象`val.hc = "hclua"`：


1. 查找val中是否有hc的值，若有直接设置该值
2. 查找object中对应的元表`lua_getmetatable`若为meta
3. 找到`__newindex`的key值，若不存在则返回空值
4. 调用`__newindex`函数，此时调用该数第一个参数为`val`，第二个参数为`hc`，第三个参数为字符串`"hclua"`
5. 若此时判断第二个参数不是字段，则直接返回lua错误内容
6. 此时我们会在第二个参数的key值后面添加`__set`即为`hc__set`，我们查找meta\["hc\_\_set"] 若为空则返回失败，若为函数则转到7
7. 我们将调用该函数，并将第一个参数`val`，第三个参数`hclua`，并进行函数调用



```


|  | lua_pushvalue(lua, 1); |
| --- | --- |
|  | lua_pushvalue(lua, 3); |
|  | lua_call(lua, 2, 1); |


```

此时字段的设置已经完成了。


### 小结


Lua的处理速度较慢，为了高性能，通常有许多函数会放到Rust层或者底层进行处理，此时有一个快速的映射就可以方便代码的快速使用复用，而通过derive宏，我们可以快速的构建出想要的功能。


