
- when are objects composed? 
  - during application startup.
- how does an object obtain a reference to another one? 
  - constructor parameter
  - method parameter
- where are objects composed?
  - as early as possible
  
## object_connection  
### static
- connection stable
- use constructor
```
public static void main(String[] args) {
  Sender sender = new Sender(new Recipient());
  sender.send();
}
```

### dynamic
- connection is temporaty.
- use method parameter 
```
database.encryptUsing(encryption1);
database.encryptUsing(encryption2);
```
