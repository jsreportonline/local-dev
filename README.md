#jsreportonline development

1. download an install [docker](https://www.docker.com/community-edition#/download)
2. clone this repository
3. run `clone-all.bat`
4. add the following to `/etc/hosts`
```sh
  local.net 127.0.0.1
  test.local.net 127.0.0.1
```
5. run `docker-compose up` (can take ~10 minutes based on the network)
6. open [http://local.net](http://local.net) in browser and register tenant with subdomain `test`