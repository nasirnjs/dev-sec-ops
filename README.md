
The command sudo sysctl -w vm.max_map_count=262144 increases the limit for memory mappings, which is important for resource-heavy applications like Elasticsearch or SonarQube, especially in Docker environments.

`sudo sysctl -w vm.max_map_count=262144`