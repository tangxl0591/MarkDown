## KSet

### kset�ṹ��

```c
struct kset {
	struct list_head list;         // kobject �б�
	spinlock_t list_lock;
	struct kobject kobj;           // kset��Ƕkobject
	const struct kset_uevent_ops *uevent_ops;  // kset �¼�
};
```

### kset ��Ӻ���
```c
struct kset *kset_create_and_add(const char *name,
				 const struct kset_uevent_ops *uevent_ops,
				 struct kobject *parent_kobj)
{
	struct kset *kset;
	int error;

	kset = kset_create(name, uevent_ops, parent_kobj); // ��ʼ����Ƕkobject������kset
	if (!kset)
		return NULL;
	error = kset_register(kset);   // ע�ắ��
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
	kset->kobj.ktype = &kset_ktype; \\ktype����Ϊkset_ktype
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

	kset_init(k);                          // ʵ�ʳ�ʼ��kset��Ƕkobject
	err = kobject_add_internal(&k->kobj);  // ����sysfs�ļ�
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
