---
title: NFS-Ganesha对接CephFS时iops很低
date: 2024-07-17 17:01:33
tags: [ganesha,nfs,cephfs]
categories: ceph
---

NFS-Ganesha对接CephFS时iops很低，记录排查过程。Ceph版本14.2.11；NFS-Ganesha版本2.8.4。

<!-- more -->

# 现象 (1)

Nfs-ganesha对接cephfs。使用两种挂载方式(fuse和nfsv4)，分别压测:

```
[global]
    ioengine=libaio
    buffered=0
    direct=1
    iodepth=128
    rw=randwrite
    bs=4k
    numjobs=1
    thread
    size=300G
    runtime=120

[file1]
    filename=/mnt/test/testfile.empty102
```

- ceph-fuse: iops=1500
- nfsv4: iops=955


# 排查 (2)

## pool的状态(`ceph osd pool stats`)

    - fuse压测时，cephfs data pool的写带宽6MiB/s；metadata pool几乎没有写(1.6K，或者nothing is going on)；
    - nfsv4压测时，cephfs data pool的写带宽3.8MiB/s；metadata pool的写带宽1.9MiB/s；
    
注意到data pool写带宽(3.8)是metadata pool写带宽(1.9)的2倍；设置`bs=512k`，重新压测nfsv4发现data pool的写带宽是metadata pool的256倍；设置`bs=1024k`，data pool的写带宽是metadata pool的512倍，以此类推。

## OSD日志

鉴于上面data pool和metadata pool写带宽的比例关系，猜测：nfsv4压测时，假如data pool和metadata pool的iops是1:1的话，那么metadata pool的io-size应该是2K左右。
所以，统计所有OSD的日志，grep包含`' lat '`的行(级别为15)，然后按pool分类，再相加，发现总数基本一样，相差不超过1%。也就是说：**data pool和metadata pool的iops确实是1:1**。并且metadata pool的io-size是1930字节左右，接近2K。

## MDS perf统计

nfs压测前：

```
ceph daemon  /var/run/ceph/mds.{name}.asok perf reset all
```

nfs压测后：

```
ceph daemon  /var/run/ceph/mds.{name}.asok perf dump
```

发现MDS基本没有什么写操作，**可以排除attr修改的可能**。


## MDS日志

```
 5 mds.0.log _submit_thread 154413621245~1910 : EUpdate cap update [metablob 0x1, 2 dirs]
```

- fuse压测：`_submit_thread`日志只有38行。
- nfsv4压测：`_submit_thread`日志有114243行，除以压测时间120秒，得每秒952行，和iops基本一致。再次证明：**data pool和metadata pool的iops是1:1，每进行一次数据写，MDS都会写一次journal，大小1910字节，接近2K**。


```
mds.0.locker handle_client_caps  on 0x1000000002e tid 1097259 follows 1 op update flags 0x1
...
mds.0.locker _do_cap_update dirty Fw issued pAsxLsXsxFsxcrwb wanted pAsxXsxFsxcrwb on [inode 0x1000000002e [2,head] /perf-test-13/testfile.empty102
...
```

这个日志出现的次数和`_submit_thread`基本一样。

## 设置`mds_log_pause`

可以通过`mds_log_pause`命令MDS是否写journal:

```
ceph daemon /var/run/ceph/mds.{name}.asok config set mds_log_pause {true or false}
```

- nfsv4压测：设置true，fio的iops立即跌到0，设置false，iops立即增长，可反复验证。
- fuse压测：设置为true，fio压测可以继续。

再次说明：**nfsv4压测时，每进行一次数据写，MDS都会写一次journal，不让写journal就不能正常写文件。data pool和metadata pool的iops是1:1**


## dump journal events

```
cephfs-journal-tool --rank=<fs>:<rank> event get list > journal.events

tail journal.events

2024-07-16 11:49:02.641177 0x24357f3b04 UPDATE:  (cap update)
  testfile.empty102
2024-07-16 11:49:02.643862 0x24357f4170 UPDATE:  (cap update)
  testfile.empty102
2024-07-16 11:49:20.385734 0x24357f47dc UPDATE:  (cap update)
  testfile.empty102
2024-07-16 11:49:35.607546 0x24357f4e22 SESSION:  ()
2024-07-16 11:49:35.618058 0x24357f4fe8 SESSION:  ()
2024-07-16 11:50:02.584154 0x24357f5067 SESSION:  ()
2024-07-16 11:50:02.595054 0x24357f522d SESSION:  ()
```

- nfsv4压测：压测时间段内，有非常多的`UPDATE:  (cap update)`，因为list截断了统计不完全，也有92640之多；总共应该也是114243。
- fuse压测：只有几十行`UPDATE:  (cap update)`。

所以至此可以断定：**nfsv4 iops低就是频繁flush journal导致的；而journal event就是cap update**。

## cap update处理流程

Cephfs的客户端(nfs-ganesha)发来capability update (`CEPH_CAP_OP_UPDATE`)请求。

- 生成EUpdate并放入`MDLog::pending_events` (代码摘自ceph-14.2.11 mds)

```cpp
void Locker::dispatch(const Message::const_ref &m) -->
void Locker::handle_client_caps(const MClientCaps::const_ref &m) -->
bool Locker::_do_cap_update(CInode *in, Capability *cap, 
                            int dirty, snapid_t follows,
                            const MClientCaps::const_ref &m, const MClientCaps::ref &ack,
                            bool *need_flush)  -->
{
    // ......

    // do the update.
    EUpdate *le = new EUpdate(mds->mdlog, "cap update");
    mds->mdlog->start_entry(le);

    // ......

    mds->mdlog->submit_entry(le, new C_Locker_FileUpdate_finish(this, in, mut, update_flags,
                                                                ack, client));

}
```

- 处理EUpdate：生成journal entry，并flush到journal，journal是存在metadata pool上的 (代码摘自ceph-14.2.11 mds)

```cpp
void MDLog::_submit_thread()
{
  while (!mds->is_daemon_stopping()) {
      // ......

      map<uint64_t,list<PendingEvent> >::iterator it = pending_events.begin();

      // ......

      PendingEvent data = it->second.front();

      // ......

      if (data.le) {
        LogEvent *le = data.le;
        LogSegment *ls = le->_segment;
        // encode it, with event type
        bufferlist bl;
        le->encode_with_header(bl, features);

        uint64_t write_pos = journaler->get_write_pos();

        le->set_start_off(write_pos);
        if (le->get_type() == EVENT_SUBTREEMAP)
          ls->offset = write_pos;

        dout(5) << "_submit_thread " << write_pos << "~" << bl.length()
                << " : " << *le << dendl;

        // journal it.
        const uint64_t new_write_pos = journaler->append_entry(bl);  // bl is destroyed.
        ls->end = new_write_pos;

        MDSLogContextBase *fin;
        if (data.fin) {
          fin = dynamic_cast<MDSLogContextBase*>(data.fin);
          ceph_assert(fin);
          fin->set_write_pos(new_write_pos);
        } else {
          fin = new C_MDL_Flushed(this, new_write_pos);
        }

        journaler->wait_for_flush(fin);

        if (data.flush)
          journaler->flush();

        if (logger)
          logger->set(l_mdl_wrpos, ls->end);

        delete le;
      } else {

        // ......

      }

      // ......
  }
}
```

## 谁发的capability update (`CEPH_CAP_OP_UPDATE`)请求

打开nfs-ganesha端的libcephfs的日志(ceph.conf里设置`debug_client=20`)，可以看到：

```
2024-07-16 19:22:07.468 7f0b003e8700 10 client.230209841 get_caps 0x10000000031.head(...) have pAsxLsXsxFsxcrwb need AsFw want Fb revoking -
2024-07-16 19:22:07.468 7f0b003e8700 10 client.230209841  snaprealm snaprealm(0x1 nref=3 c=0 seq=1 parent=0x0 my_snaps=[] cached_snapc=1=[])
2024-07-16 19:22:07.468 7f0b003e8700  7 client.230209841 wrote to 4674625536, leaving file size at 23498956800
2024-07-16 19:22:07.468 7f0b003e8700 10 mark_caps_dirty 0x10000000031.head(...) - -> Fw
2024-07-16 19:22:07.468 7f0b003e8700  3 client.230209841 ll_write 0x7f0b0ce62cc0 4674621440~4096 = 4096
2024-07-16 19:22:07.468 7f0b003e8700  3 client.230209841 ll_fsync 0x7f0b0ce62cc0 0x10000000031
2024-07-16 19:22:07.468 7f0b003e8700  8 client.230209841 _fsync(0x7f0b0ce62cc0, data+metadata)
2024-07-16 19:22:07.468 7f0b003e8700  8 client.230209841 _fsync on 0x10000000031.head(...) (data+metadata)
2024-07-16 19:22:07.468 7f0b003e8700 15 inode.get on 0x7f0b14457f00 0x10000000031.head now 11
2024-07-16 19:22:07.468 7f0b003e8700 10 client.230209841 _flush 0x10000000031.head(...)
2024-07-16 19:22:07.468 7f0b003e8700 15 client.230209841 using return-valued form of _fsync
2024-07-16 19:22:07.468 7f0b003e8700 10 client.230209841 check_caps on 0x10000000031.head(...) wanted pAsxXsxFsxcrwb used Fcb issued pAsxLsXsxFsxcrwb revoking - flags=3
2024-07-16 19:22:07.468 7f0b003e8700 10 client.230209841  cap mds.0 issued pAsxLsXsxFsxcrwb implemented pAsxLsXsxFsxcrwb revoking -
2024-07-16 19:22:07.468 7f0b003e8700 10 client.230209841 mark_caps_flushing (more) Fw 0x10000000031.head(...)
2024-07-16 19:22:07.468 7f0b003e8700 10 mark_caps_clean 0x10000000031.head(...)
2024-07-16 19:22:07.468 7f0b003e8700 10 client.230209841 send_cap 0x10000000031.head(...) mds.0 seq 2 used Fcb want pAsxXsxFsxcrwb flush Fw retain pAsxLsxXsxFsxcrwbl held pAsxLsXsxFsxcrwb revoking - dropping -
2024-07-16 19:22:07.468 7f0b003e8700 15 client.230209841 auth cap, requesting max_size 320360443904
2024-07-16 19:22:07.468 7f0b003e8700 15 client.230209841 waiting on data to flush
```

从log可知，`ceph_ll_fsync()`的参数`syncdataonly=false`，因为要sync `data+metadata`。

```cpp

int Client::_fsync(Inode *in, bool syncdataonly)
{
  // ......

  ldout(cct, 8) << "_fsync on " << *in << " " << (syncdataonly ? "(dataonly)":"(data+metadata)") << dendl;


  if (!syncdataonly && in->dirty_caps) {
    check_caps(in, CHECK_CAPS_NODELAY|CHECK_CAPS_SYNCHRONOUS);
    if (in->flushing_caps)
      flush_tid = last_flush_tid;
  } else ldout(cct, 10) << "no metadata needs to commit" << dendl;

  // ......

}

```

可见`syncdataonly=false`，会调用`check_caps`:

```cpp
void Client::check_caps(Inode *in, unsigned flags) -->
void Client::send_cap(Inode *in, MetaSession *session, Cap *cap,
                      int flags, int used, int want, int retain,
                      int flush, ceph_tid_t flush_tid)
{
    int op = CEPH_CAP_OP_UPDATE;

    // ......

    auto m = MClientCaps::create(op,
                                     in->ino,
                                     0,
                                     cap->cap_id, cap->seq,
                                     cap->implemented,
                                     want,
                                     flush,
                                     cap->mseq,
                                     cap_epoch_barrier);
    // ......

    session->con->send_message2(std::move(m));
}
```

# 根因 (3)

所以，根因是ganesha调用libcephfs的`ceph_ll_fsync`时传入`syncdataonly=false`。尝试通过fio的`fsync`和`fdatasync`选项来控制这个参数，也不起作用。而ceph-fuse是可以通过这两个选项控制的。具体对比：

| fio选项                  | nfs-ganesha        | ceph-fuse          |
|--------------------------|--------------------|--------------------|
| --direct=1               | sync data+metadata | no sync            |
| --direct=1 --fsync=1     | sync data+metadata | sync data+metadata |
| --direct=1 --fdatasync=1 | sync data+metadata | sync data-only     |

并且，ceph-fuse在`--direct=1 --fsync=1`时(也要sync data+metadata)，表现的和nfsv4一模一样：metadata pool上有很大的带宽，iops很低...

看ganesha的代码发现，所有对`ceph_ll_fsync`的调用参数`syncdataonly`都为false。暴力修改src/FSAL/FSAL_CEPH/handle.c中`ceph_fsal_write2()`函数中的调用参数为true，iops就会达到ceph-fuse的水平，并且metadata pool上没有带宽。问题是这么修改安全吗？

# 下一步 (4)

- 评估上述修改是否安全；
- 看ceph-fuse是如何支持fdatasync的；
- libcephfs给mds发capability update请求，mds的perf统计中为什么体现不出来？
- capability update为什么有1900多字节之多？
