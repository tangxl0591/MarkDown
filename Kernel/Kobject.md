## Kobject

### kobject�ṹ��
```c
struct kobject {
	const char		   *name;		// kobject ����
	struct list_head	entry;		// kobject ��kset�еĽڵ�
	struct kobject		*parent;	// kobject ���ڵ�
	struct kset			*kset;		// ������kset����
	struct kobj_type	*ktype;
	struct kernfs_node	*sd;		// sysfs�ڵ�
	struct kref			kref;		// ���ü���
#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
	struct delayed_work	release;
#endif
	unsigned int state_initialized:1;	// ��ʼ����־
	unsigned int state_in_sysfs:1;		// sysfs ��־
	unsigned int state_add_uevent_sent:1;
	unsigned int state_remove_uevent_sent:1;
	unsigned int uevent_suppress:1;
};
```

### kobj_type�ṹ��

```c
struct kobj_type {
	void (*release)(struct kobject *kobj);
	const struct sysfs_ops *sysfs_ops;		// kobject sysfs������
	struct attribute **default_attrs;		// sysfsĬ������
	const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
	const void *(*namespace)(struct kobject *kobj);
};
```

**������������**

kobject ���ü�����1
```c
/**
 * kobject_put - decrement refcount for object.
 * @kobj: object.
 *
 * Decrement the refcount, and if 0, call kobject_cleanup().
 */
void kobject_put(struct kobject *kobj)
```
kobject ���ü�����1

```c
/**
 * kobject_get - increment refcount for object.
 * @kobj: object.
 */
struct kobject *kobject_get(struct kobject *kobj)	//
```
kobject����kset����

```c
/* add the kobject to its kset's list */
static void kobj_kset_join(struct kobject *kobj)
```
kobject��kset����ɾ��

```c
/* remove the kobject from its kset's list */
static void kobj_kset_leave(struct kobject *kobj)
```
**��ʼ��**
```c
static void kobject_init_internal(struct kobject *kobj)
{
	if (!kobj)
		return;
	kref_init(&kobj->kref);					// ��ʼ������
	INIT_LIST_HEAD(&kobj->entry);			// ��ʼ������
	kobj->state_in_sysfs = 0;
	kobj->state_add_uevent_sent = 0;
	kobj->state_remove_uevent_sent = 0;
	kobj->state_initialized = 1;				// ��ʼ����־
}
```
**���kobject**
```c
    static __printf(3, 0) int kobject_add_varg(struct kobject *kobj,
    					   struct kobject *parent,
    					   const char *fmt, va_list vargs)
    {
    	int retval;

    	retval = kobject_set_name_vargs(kobj, fmt, vargs);				// ����kobject����
    	if (retval) {
    		printk(KERN_ERR "kobject: can not set name properly!\n");
    		return retval;
    	}
    	kobj->parent = parent;								// ����parent�ڵ�
    	return kobject_add_internal(kobj);					// kobject add ʵ�ʲ�������
    }
```

**ʵ��kobject add ����**
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

	parent = kobject_get(kobj->parent);			// kobj���ڵ����ü�����1

	/* join kset if set, use it as parent if we do not already have one */
	if (kobj->kset) {							// ����kset��
		if (!parent)
			parent = kobject_get(&kobj->kset->kobj);	// �����ȡ�������ڵ��kset�л�ȡ��һ��
		kobj_kset_join(kobj);					// ʵ�ʼ���kset��
		kobj->parent = parent;
	}

	pr_debug("kobject: '%s' (%p): %s: parent: '%s', set: '%s'\n",
		 kobject_name(kobj), kobj, __func__,
		 parent ? kobject_name(parent) : "<NULL>",
		 kobj->kset ? kobject_name(&kobj->kset->kobj) : "<NULL>");

	error = create_dir(kobj);					// ����sysfs�ļ�
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
		kobj->state_in_sysfs = 1;				// sysfs��ʼ����

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

**ʵ��kobject del ����**
```c
void kobject_del(struct kobject *kobj)
{
	struct kernfs_node *sd;

	if (!kobj)
		return;

	sd = kobj->sd;
	sysfs_remove_dir(kobj);			// ��sysfs��ɾ���ļ���
	sysfs_put(sd);					// sysfs �ļ�����ɾ��

	kobj->state_in_sysfs = 0;
	kobj_kset_leave(kobj);			// kobject ��ksetɾ��
	kobject_put(kobj->parent);		// parent �ڵ�ɾ��
	kobj->parent = NULL;
}
```
