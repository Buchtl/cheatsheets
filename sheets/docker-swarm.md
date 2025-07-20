## Labels
```
# docker node update --label-add <key>=<value> <node-name-or-ID>
docker node update --label-add role=storage worker-1
# verify the label was added with
docker node inspect worker-1 --format '{{ json .Spec.Labels }}'
# Or get all node labels in a readable format
docker node ls
docker node inspect worker-1 --pretty
# remove label
docker node update --label-rm role worker-1
```
