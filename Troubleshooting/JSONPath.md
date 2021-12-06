# JSON Path support

`kubectl` supports JSONPath template.

JSONPath template is composed of JSONPath expressions enclosed by curly braces {}.

`kubectl` uses JSONPath expressions to filter on specific fields in the JSON object and format the output. In addition to the original JSONPath template syntax, the following functions and syntax are valid:

1. Use double quotes to quote text inside JSONPath expressions.
2. Use the `range`, `end` operators to iterate lists.
3. Use negative slice indices to step backwards through a list. Negative indices do not *wrap around* a list and are valid as long as `-index + listLength >= 0`.

> Note:
>
> - The `$` operator is optional since the expression always starts from the root object by default.
> - The result object is printed as its String() function.

| Function           | Description               | Example                                                      | Result                                            |
| ------------------ | ------------------------- | ------------------------------------------------------------ | ------------------------------------------------- |
| `text`             | the plain text            | `kind is {.kind}`                                            | `kind is List`                                    |
| `@`                | the current object        | `{@}`                                                        | the same as input                                 |
| `.` or `[]`        | child operator            | `{.kind}`, `{['kind']}` or `{['name\.type']}`                | `List`                                            |
| `..`               | recursive descent         | `{..name}`                                                   | `127.0.0.1 127.0.0.2 myself e2e`                  |
| `*`                | wildcard. Get all objects | `{.items[*].metadata.name}`                                  | `[127.0.0.1 127.0.0.2]`                           |
| `[start:end:step]` | subscript operator        | `{.users[0].name}`                                           | `myself`                                          |
| `[,]`              | union operator            | `{.items[*]['metadata.name', 'status.capacity']}`            | `127.0.0.1 127.0.0.2 map[cpu:4] map[cpu:8]`       |
| `?()`              | filter                    | `{.users[?(@.name=="e2e")].user.password}`                   | `secret`                                          |
| `range`, `end`     | iterate list              | `{range .items[*]}[{.metadata.name}, {.status.capacity}] {end}` | `[127.0.0.1, map[cpu:4]] [127.0.0.2, map[cpu:8]]` |
| `''`               | quote interpreted string  | `{range .items[*]}{.metadata.name}{'\t'}{end}`               | `127.0.0.1 127.0.0.2`                             |

```bash
kubectl get pods -o json
kubectl get pods -o=jsonpath='{@}'
kubectl get pods -o=jsonpath='{.items[0]}'
kubectl get pods -o=jsonpath='{.items[0].metadata.name}'
kubectl get pods -o=jsonpath="{.items[*]['metadata.name', 'status.capacity']}"
kubectl get pods -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.startTime}{"\n"}{end}'
kubectl get nodes -o=custom-columns=<COLUMN NAME>:<JSON PATH>
kubectl get nodes -o=custom-columns=NODE:.metadata.name,CPU:.status.capacity.cpu
kubectl get pods --sort-by=.metadata.name
```
