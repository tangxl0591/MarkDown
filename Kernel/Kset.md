## KSet

### kset结构体

```c
struct kset {
	struct list_head list;         // kobject 列表
	spinlock_t list_lock;
	struct kobject kobj;           // kset内嵌kobject
	const struct kset_uevent_ops *uevent_ops;  // kset 事件
};
```

### kset 添加函数
```c
struct kset *kset_create_and_add(const char *name,
				 const struct kset_uevent_ops *uevent_ops,
				 struct kobject *parent_kobj)
{
	struct kset *kset;
	int error;

	kset = kset_create(name, uevent_ops, parent_kobj); // 初始化内嵌kobject，申请kset
	if (!kset)
		return NULL;
	error = kset_register(kset);   // 注册函数
	if (error) {
		kfree(kset);
		return NULL;
	}
	return kset;
}
```
```c
static struct kset *kset_create(const char *name,
				const struct kset_uevent_ops *uevent_ops,
				struct kobject *parent_kobj)
{
	struct kset *kset;
	int retval;

	kset = kzalloc(sizeof(*kset), GFP_KERNEL);
	if (!kset)
		return NULL;
	retval = kobject_set_name(&kset->kobj, "%s", name);
	if (retval) {
		kfree(kset);
		return NULL;
	}
	kset->uevent_ops = uevent_ops;
	kset->kobj.parent = parent_kobj;

	/*
	 * The kobject of this kset will have a type of kset_ktype and belong to
	 * no kset itself.  That way we can properly free it when it is
	 * finished being used.
	 */
	kset->kobj.ktype = &kset_ktype; \\ktype设置为kset_ktype
	kset->kobj.kset = NULL;

	return kset;
}
```

```c
int kset_register(struct kset *k)
{
	int err;

	if (!k)
		return -EINVAL;

	kset_init(k);                          // 实际初始化kset内嵌kobject
	err = kobject_add_internal(&k->kobj);  // 创建sysfs文件
	if (err)
		return err;
	kobject_uevent(&k->kobj, KOBJ_ADD);
	return 0;
}
```

```c
         +-------------------------+
         |          KSET           |
         |                         |
         |              +----------+
     +---+              | Kobject || <-------+
     |   +-------------------------|         |
     |   +---------------^ ^    ^--------+   |
     |   |                 |             |   |
     |   |                 |             |   |
    +v--------+       +---------+       +---------+
    | Kobject +-----> | Kobject +-----> | Kobject |
    +---------+       +---------+       +---------+
```
