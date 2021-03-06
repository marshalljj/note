# iterator模式在跑批中的应用
在跑批任务中往往需要对数据进行遍历，我们看下如下的两种实现方式。
## 不使用iterator
最常见对方式就是分页处理：分页读，分页处理。代码如下：
```
public class Processor{
    public void execute() {
        int page = 0;
        int size = 1000;
        do {
            int offset = page * size;
            List<Object> list = reader.page(offset, size);
            if (CollectionUtils.isEmpty(list)) {
                break;
            }
            processList(list);
            page++;
        } while (true);
    }

    private void processList(List<Object> list) {
        list.forEach(item ->{
            processOne(item);
        });
    }
}
```
- 在同一个类里面搞定遍历和处理，违反了（SRP）.
- 将分页细节暴露于上层,使代码难以阅读。

## 使用iterator
```
public class Processor {
    public void execute() {
        Iterator iterator = new XXXIterator(reader);
        while (iterator.hasNext()) {
            processOne(iterator.next());
        }
    }
}

public class XXXIterator {
    private Reader reader;
    private Long currentId = 0L;
    private int limit = CommonConstants.QUERY_SIZE;
    private Queue<Object> queueBuffer = Lists.newLinkedList();

    public XXXIterator(Reader reader) {
        this.reader = reader;
    }

    public boolean hasNext() {
        if (queueBuffer.isEmpty()) {
            List<Object> newPage = reader.loadTopNFromId(currentId, limit);
            queueBuffer.addAll(newPage);
        }
        return !queueBuffer.isEmpty();
    }

    public Object next() {
        Object poll = queueBuffer.poll();
        currentId = poll.getId();
        return poll;
    }
}
```
使用iterator将分页遍历的实现细节隐藏于XXXIterator之内，使用的时候并不需要关心底层的存储细节。
