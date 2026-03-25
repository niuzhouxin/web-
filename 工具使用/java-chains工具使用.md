打开docker,执行命令
```
docker run -d \
  --name java-chains \
  --restart=always \
  -p 8011:8011 \
  -p 58080:58080 \
  -p 50389:50389 \
  -p 50388:50388 \
  -p 3308:3308 \
  -p 13999:13999 \
  -p 50000:50000 \
  -p 11527:11527 \
  -e CHAINS_AUTH=true \
  -e CHAINS_PASS=114514 \
  javachains/javachains:latest
```
其中CHAINS_PASS是网页密码
执行成功后访问`http://localhost:8011/`