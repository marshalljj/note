
```sql
CREATE TABLE `orders` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `user_id` int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `users` (
  `id` int(10) NOT NULL AUTO_INCREMENT,
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `user_name` varchar(255) NOT NULL DEFAULT '',
  `age` int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;
```

process
```java
@Override
    public void run(String... strings) throws Exception {
        Node root = new Node("users", "id");
        root.addChildren(new Node("orders", "user_id"));
        processNode(root);


    }

    public void processNode(Node node) {
        Map map = sqlMapper.selectOne(node.tableName, node.key, 1);
        processChild(node.getChildren(), map);
        System.out.println(map);
    }

    private void processChild(List<Node> nodes, Map root) {
        nodes.forEach(node1 -> {
            if (node1.isSingle()) {
                Map map = sqlMapper.selectOne(node1.tableName, node1.key, 1);
                root.put(node1.tableName, map);
            } else {
                List<Map> maps = sqlMapper.selectList(node1.tableName, node1.key, 1);
                root.put(node1.tableName, maps);
            }
        });
    }

    public static class Node {
        private String tableName;
        private String key;
        private boolean single;
        private List<Node> children;

        public Node(String tableName, String key) {
            this.tableName = tableName;
            this.key = key;
            this.children = new ArrayList<>();
        }

        public void addChildren(Node child) {
            children.add(child);
        }

        public List<Node> getChildren() {
            return children;
        }

        public boolean isSingle() {
            return single;
        }
    }
```
```java
@Mapper
public interface SqlMapper {

    @Select("select * from ${tableName} where ${key}=#{value}")
    Map selectOne(@Param("tableName") String tableName, @Param("key") String key, @Param("value") Object value);

    @Select("select * from ${tableName} where ${key}=#{value}")
    List<Map> selectList(@Param("tableName") String tableName, @Param("key") String key, @Param("value") Object value);
}
```

## plan
- node can be parsed from xml. 
```xml
<node>
	<table>user</table>
	<key>id</key>
	<fileds>
		<filed>
			<name>age</name>
			<process>int2string</process>
		</filed>
		<filed>
			<name>tags</name>
			<process>string2array</process>
		</filed>
		<filed>
			<name>phones</name>
			<process>json2array</process>
		</filed>
		...
		<filed>
			<name>id</name>
			<process>none</process>
		</filed>
	</fileds>
	<node>
		...
	</node>
</node>
```
- scale up 
```xml
<instance>
	<datasource>
		<tag>test</tag>
		<key>product</key>
	</datasource>
	<kafka>
		...
	</kafka>
	<es>
		...
	</es>
	<node>
		<table>user</table>
		<key>id</key>
		<fileds>
			<filed>
				<name>age</name>
				<process>int2string</process>
			</filed>
			<filed>
				<name>tags</name>
				<process>string2array</process>
			</filed>
			<filed>
				<name>phones</name>
				<process>json2array</process>
			</filed>
			...
			<filed>
				<name>id</name>
				<process>none</process>
			</filed>
		</fileds>
		<node>
			...
		</node>
	</node>
</instance>
```
