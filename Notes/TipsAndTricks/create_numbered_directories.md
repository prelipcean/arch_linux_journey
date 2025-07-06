
## Bash

```
for i in {1..10}; do
  mkdir "Section_$(printf "%02d" $i)"
done
```

## Terminal

```
for i in {1..10}; do mkdir "Section_$(printf "%02d" $i)"; done
```