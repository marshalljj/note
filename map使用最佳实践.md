
## 存在则返回，不存在放入
### 原始
```java
if(map.containsKey(id)) {
    return map.get(id);
}else {
    User user = getById(id);
    map.put(user);
    return user;
}
```

### 最佳实践
```java
return map.computeIfAbsent(id, t ->getByid(id));
```
## 计数

### 原始
```java
for (String key : keys) {
    if (map.containsKey(key)) {
        map.put(key, map.get(key)+1);
    }else {
        map.put(key, 1);
    }
}
```

### 最佳实践
```java
for (String key : keys) {
    map.put(key, map.getOrDefault(key,0)+1);   
}
//或者
for (String key : keys) {
    map.compute(key, (k, v) -> (v == null) ? 1 : v+1);
}

```
