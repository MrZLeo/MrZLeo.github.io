---
tags:
  - wiki
---

> [!NOTE] Documentation
> [📑](https://www.kernel.org/doc/html/v6.6/filesystems/vfs.html)

# `filp_close` 的 async 逻辑

- VFS 关闭文件的逻辑并不是顺序执行的，而是存在一个 delay 操作：
```c
int filp_close(struct file *filp, fl_owner_t id)
{
	int retval;

	retval = filp_flush(filp, id); // flush operation, call filp->f_op->flush
	fput(filp); // real close logic

	return retval;
}
EXPORT_SYMBOL(filp_close);

void fput(struct file *file)
{
	if (atomic_long_dec_and_test(&file->f_count)) {
		struct task_struct *task = current;

		if (likely(!in_interrupt() && !(task->flags & PF_KTHREAD))) {
			init_task_work(&file->f_rcuhead, ____fput); // <- real task
			if (!task_work_add(task, &file->f_rcuhead, TWA_RESUME))
				return;
			/*
			 * After this task has run exit_task_work(),
			 * task_work_add() will fail.  Fall through to delayed
			 * fput to avoid leaking *file.
			 */
		}

		// put task to workqueue
		if (llist_add(&file->f_llist, &delayed_fput_list))
			schedule_delayed_work(&delayed_fput_work, 1); // 1 is delayed jiffies
	}
}
```

- 然而，对于真正的 close 系统调用，却并不使用这样的 delay 逻辑：

```c
SYSCALL_DEFINE1(close, unsigned int, fd)
{
	int retval;
	struct file *file;

	file = close_fd_get_file(fd);
	if (!file)
		return -EBADF;

	retval = filp_flush(file, current->files);

	/*
	 * We're returning to user space. Don't bother
	 * with any delayed fput() cases.
	 */
	__fput_sync(file); // <- __fput_sync 是同步操作，没有delay

	/* can't restart close syscall because file table entry was cleared */
	if (unlikely(retval == -ERESTARTSYS ||
		     retval == -ERESTARTNOINTR ||
		     retval == -ERESTARTNOHAND ||
		     retval == -ERESTART_RESTARTBLOCK))
		retval = -EINTR;

	return retval;
}
```