---
title: filecmp
date: 2021-02-05 10:29:00 +0800
category: Libs
---

Python2.3以上的版本默认自带了filecmp模块，无需额外安装。我们可以用这个模块来检查原式与目标文件的一致性，
filecmp可以实现文件、目录、遍历子目录的差异对比功能。

#### 代码
```
https://github.com/python/cpython/blob/2.7/Lib/filecmp.py
https://github.com/python/cpython/blob/3.7/Lib/filecmp.py
```

#### 接口

```python
Classes:
    dircmp                                   //文件夹比较
Functions:
    cmp(f1, f2, shallow=1) -> int            //单文件比较
    cmpfiles(a, b, common) -> ([], [], [])   //多文件比较
    clear_cache()                            //python3版本，清理缓存
```

#### 实例

```python
#本实例比较a、b两个文件夹,输出相同的、变化的、新增的、删除的文件名。
from filecmp import dircmp
import logging
import os

def walk_dir(dir_in, files_out):
    for root, _, files in os.walk(dir_in, followlinks=True):
        for f in files:
            file_out = os.path.join(root, f)
            files_out.append(file_out)

def get_diff_files(dcmp, same_items, change_items, add_items, del_items):
    # 获取相同的文件
    for name in dcmp.same_files:
        same_fn = os.path.join(dcmp.left, name)
        same_items.append(same_fn)

    # 获取变化的文件
    for name in dcmp.diff_files:
        change_fn = os.path.join(dcmp.left, name)
        change_items.append(change_fn)

    # 获取新增的文件
    left_only = dcmp.left_only
    if left_only:
        for name in left_only:
            left_fn = os.path.join(dcmp.left, name)
            if os.path.isdir(left_fn):
                walk_dir(left_fn, add_items)
            add_items.append(left_fn)

    # 获取删除的文件
    right_only = dcmp.right_only
    if right_only:
        for name in dcmp.right_only:
            right_fn = os.path.join(dcmp.right, name)
            if os.path.isdir(right_fn):
                walk_dir(right_fn, del_items)
            del_items.append(right_fn)

    # 递归获取子文件夹的变化情况
    for sub_dcmp in dcmp.subdirs.values():
        get_diff_files(sub_dcmp, same_items, change_items, add_items, del_items)

def main():
    src_dri = "/a/"
    dst_dir = "/b/"
    ignore_files = ['RCS', 'CVS', 'tags']
    dcmp = dircmp(src_dir, dst_dir, ignore_files)
    get_diff_files(dcmp, same_items, change_items, add_items, del_items)
    if not diff_items and not del_items:
        logging.info('no diff files')
        logging.info('========run stop========

')
    else:
        logging.info('same items:%s' % pprint.pformat(same_items))
        logging.info('change items:%s' % pprint.pformat(change_items))
        logging.info('add items:%s' % pprint.pformat(add_items))
        logging.info('del items:%s' % pprint.pformat(del_items))
        
if __name__ == '__main__':
    main()
```

#### 注意事项

```
1.dircmp使用的文件比较函数为：先比较文件stat(mode/size/mtime),不能确定再比较文件内容，
  所以如果b中文件是a文件的复制，需要保证两个文件的mode/size/mtime相同，
  但shutil.copy2中虽然复制了这几个参数，但由于mtime的精度不够用来表示纳秒级别的数据，
  所以需要特别注意，解决方案有两种：
  1) 两个文件都设置成1个相同的时间，比如1024，或者当前时间(只要保证在16位的范围能表示的值即可);
     os.utime(src, (1024, 1024))
     os.utime(dst, (1024, 1024))
  2) python3支持纳秒级别的接口，所以可以直接使用
     os.utime(dst, ns = (os.stat(src).st_ctime_ns, os.stat(src).st_mtime_ns));
2.python3版本的dircmp比较文件时会默认使用上次比较的缓存，所以如果要比较结果特别准确，无延时，需要调用clear_cache()。
```



