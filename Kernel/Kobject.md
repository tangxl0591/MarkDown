## Kobject

### kobject结构体
```c
struct kobject {
	const char		   *name;		// kobject 名称
	struct list_head	entry;		// kobject 在kset中的节点
	struct kobject		*parent;	// kobject 父节点
	struct kset			*kset;		// 所处的kset链表
	struct kobj_type	*ktype;
	struct kernfs_node	*sd;		// sysfs节点
	struct kref			kref;		// 引用计数
#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
	struct delayed_work	release;
#endif
	unsigned int state_initialized:1;	// 初始化标志
	unsigned int state_in_sysfs:1;		// sysfs 标志
	unsigned int state_add_uevent_sent:1;
	unsigned int state_remove_uevent_sent:1;
	unsigned int uevent_suppress:1;
};
```

### kobj_type结构体

```c
struct kobj_type {
	void (*release)(struct kobject *kobj);
	const struct sysfs_ops *sysfs_ops;		// kobject sysfs操作表
	struct attribute **default_attrs;		// sysfs默认属性
	const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
	const void *(*namespace)(struct kobject *kobj);
};
```

**基本操作函数**

kobject 引用计数减1
```c
/**
 * kobject_put - decrement refcount for object.
 * @kobj: object.
 *
 * Decrement the refcount, and if 0, call kobject_cleanup().
 */
void kobject_put(struct kobject *kobj)
```
kobject 引用计数加1

```c
/**
 * kobject_get - increment refcount for object.
 * @kobj: object.
 */
struct kobject *kobject_get(struct kobject *kobj)	//
```
kobject加入kset链表

```c
/* add the kobject to its kset's list */
static void kobj_kset_join(struct kobject *kobj)
```
kobject从kset链表删除

```c
/* remove the kobject from its kset's list */
static void kobj_kset_leave(struct kobject *kobj)
```
**初始化**
```c
static void kobject_init_internal(struct kobject *kobj)
{
	if (!kobj)
		return;
	kref_init(&kobj->kref);					// 初始化引用
	INIT_LIST_HEAD(&kobj->entry);			// 初始化链表
	kobj->state_in_sysfs = 0;
	kobj->state_add_uevent_sent = 0;
	kobj->state_remove_uevent_sent = 0;
	kobj->state_initialized = 1;				// 初始化标志
}
```
**添加kobject**
```c
    static __printf(3, 0) int kobject_add_varg(struct kobject *kobj,
    					   struct kobject *parent,
    					   const char *fmt, va_list vargs)
    {
    	int retval;

    	retval = kobject_set_name_vargs(kobj, fmt, vargs);				// 设置kobject名字
    	if (retval) {
    		printk(KERN_ERR "kobject: can not set name properly!\n");
    		return retval;
    	}
    	kobj->parent = parent;								// 设置parent节点
    	return kobject_add_internal(kobj);					// kobject add 实际操作函数
    }
```

**实际kobject add 操作**
```c
static int kobject_add_internal(struct kobject *kobj)
{
	int error = 0;
	struct kobject *parent;

	if (!kobj)
		return -ENOENT;

	if (!kobj->name || !kobj->name[0]) {
		WARN(1, "kobject: (%p): attempted to be registered with empty "
			 "name!\n", kobj);
		return -EINVAL;
	}

	parent = kobject_get(kobj->parent);			// kobj父节点引用计数加1

	/* join kset if set, use it as parent if we do not already have one */
	if (kobj->kset) {							// 加入kset中
		if (!parent)
			parent = kobject_get(&kobj->kset->kobj);	// 如果获取不到父节点从kset中获取第一个
		kobj_kset_join(kobj);					// 实际加入kset中
		kobj->parent = parent;
	}

	pr_debug("kobject: '%s' (%p): %s: parent: '%s', set: '%s'\n",
		 kobject_name(kobj), kobj, __func__,
		 parent ? kobject_name(parent) : "<NULL>",
		 kobj->kset ? kobject_name(&kobj->kset->kobj) : "<NULL>");

	error = create_dir(kobj);					// 创建sysfs文件
	if (error) {
		kobj_kset_leave(kobj);
		kobject_put(parent);
		kobj->parent = NULL;

		/* be noisy on error issues */
		if (error == -EEXIST)
			WARN(1, "%s failed for %s with "
			 "-EEXIST, don't try to register things with "
			 "the same name in the same directory.\n",
			 __func__, kobject_name(kobj));
		else
			WARN(1, "%s failed for %s (error: %d parent: %s)\n",
			 __func__, kobject_name(kobj), error,
			 parent ? kobject_name(parent) : "'none'");
	} else
		kobj->state_in_sysfs = 1;				// sysfs初始化完

	return error;
}
```

void kobject_del(struct kobject *kobj)
{
	struct kernfs_node *sd;

	if (!kobj)
		return;

	sd = kobj->sd;
	sysfs_remove_dir(kobj);
	sysfs_put(sd);

	kobj->state_in_sysfs = 0;
	kobj_kset_leave(kobj);
	kobject_put(kobj->parent);
	kobj->parent = NULL;
}

**实际kobject del 操作**
```c
void kobject_del(struct kobject *kobj)
{
	struct kernfs_node *sd;

	if (!kobj)
		return;

	sd = kobj->sd;
	sysfs_remove_dir(kobj);			// 从sysfs中删除文件夹
	sysfs_put(sd);					// sysfs 文件引用删除

	kobj->state_in_sysfs = 0;
	kobj_kset_leave(kobj);			// kobject 从kset删除
	kobject_put(kobj->parent);		// parent 节点删除
	kobj->parent = NULL;
}
```
