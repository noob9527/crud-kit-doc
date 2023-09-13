# Crud Kit

### Features
- declarative query criteria
- join table on demands, no N+1 query
- unlike other ORM frameworks, it is JSON friendly, no circular reference
- unlike JPA, no object state, easy to understand
- easy to extend
- design for Spring Boot Project
- work with Mybatis seamlessly

### View Document
linux/mac:
```bash
./serve-doc.py 3000    # linux/mac
# or
./serve-doc.sh 3000
```
windows:
```cmd
python3 .\serve-doc.py 3000 # windows
```

### Drawbacks
- for now, only MySQL dialect is supported
- entity relation api doesn't support composite primary key and multi-column relation.

### Examples
Check out the [demo](../crud-kit-demo) to see typical usage in spring boot project.
