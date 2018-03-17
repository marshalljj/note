## method_form
as there is __side effects__ in java. we should conform some normal form to avoid side effect(functional method)

-------
- function
  `R apply(T)`: use input to get output
  - compute
  - transform
  - modify: in this case R = T
- consumer
  `void accept(T)`: no out put, 
  - it is also the last one to use T
  - context of chain
  - context of pipeline
- supplier
  `T get()`
  
------  
just as sql normal form, the form above matches mostly cases, but we should trade-off in complicated case.
