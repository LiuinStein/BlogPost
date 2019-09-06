### 0x00 docker鉴权的工作流程

当用户使用`docker pull`、`docker push`从一个registry拉取或上传镜像或者使用`docker login`命令登陆registry时，就会诱发registry和docker客户端间的鉴权授权机制。官方提供的registry镜像中，仅包含极为有限的权限管理机制，而且其用户的身份信息是存储在容器中的，这显然不适合大多数registry的适用场景。因此官方的registry对外提供一组接口，以适配第三方的鉴权服务器，鉴权的方式基于Json Web Token，下面我们来了解一下这种鉴权授权方式，首先看官方给出的一个流程图：

![](https://bucket.shaoqunliu.cn/image/0302.png)

1. 首先，用户在控制台使用了`docker pull`或`docker push`命令，此时docker守护进程（docker daemon）首先会与registry通信，试探一下什么权限都没有能不能拉取或上传镜像；
2. registry收到请求后，如果需要进一步的授权才能继续操作，那么返回一个`401 Unauthorized` HTTP响应，告诉docker daemon你需要进一步的权限才能进行操作；
3. 这个时候在客户端，用户就可以看到，我现在没有权限拉取或上传镜像，此时用户如想进行进一步的操作，需要使用`docker login`命令以登陆registry；
4. 在使用`docker login`命令并输入正确的用户名密码后，授权服务器返回一个标识权限信息的`Bearer token`；
5.  有了token之后，客户端重新发起对registry的请求，并在HTTP Authorization请求头中带入这个token，以示身份；
6. registry对token信息进行鉴别，如果合法即允许进行下一步操作，至此整个授权鉴权流程结束。

好了，现在让我们来逐一理解上述步骤。

### 0x01 Registry是怎么找到鉴权服务器的

这个需要在registry的启动和部署过程中进行配置，具体可配置的选项有如下几个，下面给出其在`docker-compose.yml`文件中的配置代码：

```yml
services:
  registry:
    environment:
      # 配置授权方式为token
      - REGISTRY_AUTH=token
      # 配置token授权服务器地址
      - REGISTRY_AUTH_TOKEN_REALM=https://hub.c.shaoqunliu.cn/api/service/v1/auth/token
      # token服务名称，可随便写一个
      - REGISTRY_AUTH_TOKEN_SERVICE="An example registry"
      # token发行者名称，可随便写一个
      - REGISTRY_AUTH_TOKEN_ISSUER="Shaoqun Liu"
      # HTTPS证书地址，建议申请一个免费的受信的HTTPS证书，不要自签
      - REGISTRY_AUTH_TOKEN_ROOTCERTBUNDLE=/root/ssl/hub.crt
```

[详细说明可参考此处官方文档](https://docs.docker.com/registry/configuration/#token)

通过配置，registry知道了，授权的话该找谁，应该从何处找，这个问题。

### 0x02 docker login的基本原理

好了，这个时候，我们使用`docker login hub.c.shaoqunliu.cn`来登陆我们的registry，输入用户名密码，并按下回车键之后，首先你的docker客户端守护进程，会试探性地向registry发送一个HTTP请求，以判断是否需要鉴权，请求信息通过日志记录如下：

```
192.168.1.111 - - [23/Apr/2019:06:00:46 +0000] "GET /v2/ HTTP/1.1" 401 "-" "-" "docker/18.09.2 go/go1.10.4 git-commit/6247962 kernel/4.15.0-47-generic os/linux arch/amd64 UpstreamClient(Docker-Client/18.09.2 \x5C(linux\x5C))" "-" "-" 
```

这是一个GET请求，请求地址为registry http api的基本地址，也就是只含有版本信息不含有其他参数的地址：

```
https://hub.c.shaoqunliu.cn/v2/
```

使用的User-Agent如下：

```
docker/18.09.2 go/go1.10.4 git-commit/6247962 kernel/4.15.0-47-generic os/linux arch/amd64 UpstreamClient(Docker-Client/18.09.2 \\(linux\\))
```

> 有了这个信息我们日后如果需要把所有服务都部署在一个域名下，就可以使用nginx通过判断请求UA的方式来进行代理转发了。

此时，registry返回HTTP错误代码401，告知docker客户端需要进一步授权。客户端收到401之后，马上请求你在registry配置文件中配置的授权服务器地址信息，记录如下：

```
192.168.1.111 - user [23/Apr/2019:06:00:46 +0000] "GET /api/service/v1/auth/token?account=user&client_id=docker&offline_token=true&service=An+example+registry HTTP/1.1" 200 "-" "Basic dXNlcjpwd2Q=" "docker/18.09.2 go/go1.10.4 git-commit/6247962 kernel/4.15.0-47-generic os/linux arch/amd64 UpstreamClient(Docker-Client/18.09.2 \x5C(linux\x5C))" "-" "-" 
```

依旧是一个GET请求，请求的地址格式为：

```
你之前配置的Regsirty Token鉴权的REALM?account=登陆所使用的用户名&client_id=docker&offline_token=true&service=你之前配置的Regsirty Token鉴权的service选项
```

例如使用了上文中配置文件，所请求的地址即为：

```
https://hub.c.shaoqunliu.cn/api/service/v1/auth/token?account=user&client_id=docker&offline_token=true&service=An%20example%20registry
```

> nginx在记录日志时，会把URL中的空格替换成加号，所以你在上面的日志信息中看到的是`An+example+registry`，其实际请求的应该是`An%20example%20registry`，`%20`即为空格。

同时要注意，其会使用http basic方式进行鉴权，也就是说`docker login`会把你登陆用的用户名和密码通过简单的编码之后放在HTTP请求头`Authorization`中进行传输。编码方式为：

```
Base64Encoder(用户名:密码)
```

例如当我们输入的用户名为`user`密码为`pwd`时，对`user:pwd`计算base64得`dXNlcjpwd2Q=`，将这串字符前面加上`Basic空格`之后放到`Authorization`请求头中即为`Basic dXNlcjpwd2Q=`。

授权服务器对这个请求进行鉴权，如果鉴权成功，那就通过Json Web Token生成一段token进行返回，下面给出一个例子：

```json
{
    "expiresIn":172800000,
    "issuedAt":1555992796539,     "token":"eyJhbGciOiJSUzI1NiIsIng1YyI6WyJNSUlGanpDQ0JIZWdBd0lCQWdJUURySjNlNnozaS9Ub0ZyTnU4R2Jua3pBTkJna3Foa2lHOXcwQkFRc0ZBREJ5TVFzd0NRWURWUVFHRXdKRFRqRWxNQ01HQTFVRUNoTWNWSEoxYzNSQmMybGhJRlJsWTJodWIyeHZaMmxsY3l3Z1NXNWpMakVkTUJzR0ExVUVDeE1VUkc5dFlXbHVJRlpoYkdsa1lYUmxaQ0JUVTB3eEhUQWJCZ05WQkFNVEZGUnlkWE4wUVhOcFlTQlVURk1nVWxOQklFTkJNQjRYRFRFNU1ERXlPREF3TURBd01Gb1hEVEl3TURFeU9ERXlNREF3TUZvd0hqRWNNQm9HQTFVRUF4TVRhSFZpTG1NdWMyaGhiM0YxYm14cGRTNWpiakNDQVNJd0RRWUpLb1pJaHZjTkFRRUJCUUFEZ2dFUEFEQ0NBUW9DZ2dFQkFOcEFLT0trM3E0djY3UG9WZ3dqZjhLQldUQ1dqVkpLVUxKZ1BheForOTJtRklRK21tQklrQ2UzL2RhZEp4RGRBVTV4bzZkZWFaeVhHdVU0N3FBNjh2Z3ZOV2RtQkhNb0k5OCt5NUVMV0hhMWpHcTBpZS9YUk9PMTh2dndkQ3ZiLzA2VDg2MXBteGFVSXRGN1hwVzlRcGkvckJxc1hTWUlTb0pOZ2pONGh3bVZsdEptSm5NVEtCNmdUMk5rK0hDU2hHa1dOS1E2bGNtNVl1SitQbkgwUjU4QXZxc3laZ2o5dmphd0VMbUdHdGZpWWk5TWFkMzFtZjA5QTNjU1RwS2NPeEJ1dW1rNXZWNEdkM2hqc09kWkZpaGpGQWMzbnE2ai9iQlJUQWdKZ05SaFd3U25qSmpxaWx0bTUyK2psOEF6clc4Z2tPOVBGRzZlZVlhdk44TXNaZjBDQXdFQUFhT0NBbk13Z2dKdk1COEdBMVVkSXdRWU1CYUFGSC9UbWZPZ1J3NHhBRlpXSW82M3pKN2R5Z0dLTUIwR0ExVWREZ1FXQkJSNmJCaFFZZGVlaGtZNmM0ZzVmTWxreFVGZ3BqQWVCZ05WSFJFRUZ6QVZnaE5vZFdJdVl5NXphR0Z2Y1hWdWJHbDFMbU51TUE0R0ExVWREd0VCL3dRRUF3SUZvREFkQmdOVkhTVUVGakFVQmdnckJnRUZCUWNEQVFZSUt3WUJCUVVIQXdJd1RBWURWUjBnQkVVd1F6QTNCZ2xnaGtnQmh2MXNBUUl3S2pBb0JnZ3JCZ0VGQlFjQ0FSWWNhSFIwY0hNNkx5OTNkM2N1WkdsbmFXTmxjblF1WTI5dEwwTlFVekFJQmdabmdRd0JBZ0V3ZlFZSUt3WUJCUVVIQVFFRWNUQnZNQ0VHQ0NzR0FRVUZCekFCaGhWb2RIUndPaTh2YjJOemNDNWtZMjlqYzNBdVkyNHdTZ1lJS3dZQkJRVUhNQUtHUG1oMGRIQTZMeTlqWVdObGNuUnpMbVJwWjJsMFlXeGpaWEowZG1Gc2FXUmhkR2x2Ymk1amIyMHZWSEoxYzNSQmMybGhWRXhUVWxOQlEwRXVZM0owTUFrR0ExVWRFd1FDTUFBd2dnRUVCZ29yQmdFRUFkWjVBZ1FDQklIMUJJSHlBUEFBZGdDNzJkKzhINHB4dFpPVUk1ZXFrbnRIT0ZlVkNxdFM2QnFRbG1RMmpoN1JoUUFBQVdpVDUzck5BQUFFQXdCSE1FVUNJUUQ1UHRlOEFHTDkxN3hoWUgxdVdnaGFwajRNbWV5eXRVeVRpdXdLekhoZVV3SWdFUWJlTTJhKzRWVVJjRGFSdVNFWDdOdUczVERhejhJRkhFaEd5cWZrb2lRQWRnQ0hkYi9uV1h6NGpFT1pYNzN6YnY5V2pVZFdOdjlLdFdEQnRPci9YcUNERHdBQUFXaVQ1M3VuQUFBRUF3QkhNRVVDSUcvWUdIM0loT2tNS1NCbWpwTGY4b092dzBFL3ZJWTkrckdLdmZ2bkNvOTJBaUVBdkFzRHJIc2hHWGxDKzgzVHJYSmNIOGdHK3pkelNDT0JYdTl4YlE1VkJHWXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBRThEL1MyV00xakNlREswOHdpSEFqbFQ1cjdzbWcwcjJUOXh5eWtaWnMrT3gyWlpLTVgwUHh0NlMyUVJLNE9sanZRMHhydVNxWW5XT3VWdFQyME9ZaUlyUmpTWkwrbWRTZjZTV3RYK1p5Y25XdmF0Mjh0TGN1NGo2YzhzSk5nYS9hNUNNWi9vL201YWlCN3g4YTljQThITVdMUExHZWVIRVVUREpWZThaQU5sdFhVK2ZzTHFWQ2t5T1hITEVMb3Y0NlhyS1k2a2cyOXpmbmx4c0VpOCsyTXhMOEZNcVdRZWFMOTRyNnV2OGtFTnhHOVh6cURHSlhhd2dCbVpXamlmWit5ZWdGcFQ2RnRIYno0WDZ0U3FUWUEyUUtUTnpHUGRBS2VYTjdrcGR0MlVGYzRZamI1Z1o5K1l3QURjaXMwUmJvdnVqUisrT3JKeTNEZkNJQ1d2ZDcwPSJdfQ.eyJhY2Nlc3MiOltdLCJpc3MiOiJBIGNlcnRhaW4gcG93ZXJmdWwgZGV2ZWxvcGVyIHN1cm5hbWVkIExpdSIsInN1YiI6InVzZXIiLCJhdWQiOiJBIGRvY2tlciByZWdpc3RyeSBkZXZlbG9wZWQgYnkgYSBzYW5lIGRldmVsb3BlciAtIFNoYW9xdW4gTGl1IiwiZXhwIjoxNTU2MTcyMDQ2LCJpYXQiOjE1NTU5OTkyNDYsImp0aSI6ImUxZjdjY2FlLTA4MzgtNDM3NC04MmYxLWFkNjg1YWNlYjYyMiJ9.xbj39wCZJd4-XtY6EEN7dUgFZ82NhiDkKmFOygCkRFcP1nHFJHYOIMqeralGps0q0_p4xxuYj0_NArRAFEdKrTMxwvY_mA1kNPH9CeTRIAL97UAWrgyNJJpkFgTDUuzPuT1EruAkaTOABcWLUj6EOFxFv6QhxLnWiuSSaPrt6kL8wanFjLJPoHRlYmjNv8uRTfGpeMcjZ9ricifttGK7leHVduWzA382Q3rc8IcafJkBdniUrpWYO2eunIVtLnHDRERxa4Tzcd48SEp6iTXszTsrsV60eSZVWI3a5rkkw38GONbcdxIyl1BfLAIhOWXOTvK7b7vID5H_cvvzfE4ojg"
}
```

`issuedAt`为token的签发时间，以unix时间戳的形式给定，同时`expiresIn`为token的有效时间，以毫秒为单位计。我们将在下面着重讲解那个token是怎么生成的。

### 0x03 docker pull/push鉴权的基本原理



### 0x04 Json Web Token的生成原理

搞懂这个Json Web Token的生成原理，花了我大量的时间。首先这个Json Web Token分为三段，每一段之间用一个点号`.`分隔，分别为Header.Claims.Sign，我们先来分割一下上面那个返回实例中的token，得到其Header部分为：

```
eyJhbGciOiJSUzI1NiIsIng1YyI6WyJNSUlGanpDQ0JIZWdBd0lCQWdJUURySjNlNnozaS9Ub0ZyTnU4R2Jua3pBTkJna3Foa2lHOXcwQkFRc0ZBREJ5TVFzd0NRWURWUVFHRXdKRFRqRWxNQ01HQTFVRUNoTWNWSEoxYzNSQmMybGhJRlJsWTJodWIyeHZaMmxsY3l3Z1NXNWpMakVkTUJzR0ExVUVDeE1VUkc5dFlXbHVJRlpoYkdsa1lYUmxaQ0JUVTB3eEhUQWJCZ05WQkFNVEZGUnlkWE4wUVhOcFlTQlVURk1nVWxOQklFTkJNQjRYRFRFNU1ERXlPREF3TURBd01Gb1hEVEl3TURFeU9ERXlNREF3TUZvd0hqRWNNQm9HQTFVRUF4TVRhSFZpTG1NdWMyaGhiM0YxYm14cGRTNWpiakNDQVNJd0RRWUpLb1pJaHZjTkFRRUJCUUFEZ2dFUEFEQ0NBUW9DZ2dFQkFOcEFLT0trM3E0djY3UG9WZ3dqZjhLQldUQ1dqVkpLVUxKZ1BheForOTJtRklRK21tQklrQ2UzL2RhZEp4RGRBVTV4bzZkZWFaeVhHdVU0N3FBNjh2Z3ZOV2RtQkhNb0k5OCt5NUVMV0hhMWpHcTBpZS9YUk9PMTh2dndkQ3ZiLzA2VDg2MXBteGFVSXRGN1hwVzlRcGkvckJxc1hTWUlTb0pOZ2pONGh3bVZsdEptSm5NVEtCNmdUMk5rK0hDU2hHa1dOS1E2bGNtNVl1SitQbkgwUjU4QXZxc3laZ2o5dmphd0VMbUdHdGZpWWk5TWFkMzFtZjA5QTNjU1RwS2NPeEJ1dW1rNXZWNEdkM2hqc09kWkZpaGpGQWMzbnE2ai9iQlJUQWdKZ05SaFd3U25qSmpxaWx0bTUyK2psOEF6clc4Z2tPOVBGRzZlZVlhdk44TXNaZjBDQXdFQUFhT0NBbk13Z2dKdk1COEdBMVVkSXdRWU1CYUFGSC9UbWZPZ1J3NHhBRlpXSW82M3pKN2R5Z0dLTUIwR0ExVWREZ1FXQkJSNmJCaFFZZGVlaGtZNmM0ZzVmTWxreFVGZ3BqQWVCZ05WSFJFRUZ6QVZnaE5vZFdJdVl5NXphR0Z2Y1hWdWJHbDFMbU51TUE0R0ExVWREd0VCL3dRRUF3SUZvREFkQmdOVkhTVUVGakFVQmdnckJnRUZCUWNEQVFZSUt3WUJCUVVIQXdJd1RBWURWUjBnQkVVd1F6QTNCZ2xnaGtnQmh2MXNBUUl3S2pBb0JnZ3JCZ0VGQlFjQ0FSWWNhSFIwY0hNNkx5OTNkM2N1WkdsbmFXTmxjblF1WTI5dEwwTlFVekFJQmdabmdRd0JBZ0V3ZlFZSUt3WUJCUVVIQVFFRWNUQnZNQ0VHQ0NzR0FRVUZCekFCaGhWb2RIUndPaTh2YjJOemNDNWtZMjlqYzNBdVkyNHdTZ1lJS3dZQkJRVUhNQUtHUG1oMGRIQTZMeTlqWVdObGNuUnpMbVJwWjJsMFlXeGpaWEowZG1Gc2FXUmhkR2x2Ymk1amIyMHZWSEoxYzNSQmMybGhWRXhUVWxOQlEwRXVZM0owTUFrR0ExVWRFd1FDTUFBd2dnRUVCZ29yQmdFRUFkWjVBZ1FDQklIMUJJSHlBUEFBZGdDNzJkKzhINHB4dFpPVUk1ZXFrbnRIT0ZlVkNxdFM2QnFRbG1RMmpoN1JoUUFBQVdpVDUzck5BQUFFQXdCSE1FVUNJUUQ1UHRlOEFHTDkxN3hoWUgxdVdnaGFwajRNbWV5eXRVeVRpdXdLekhoZVV3SWdFUWJlTTJhKzRWVVJjRGFSdVNFWDdOdUczVERhejhJRkhFaEd5cWZrb2lRQWRnQ0hkYi9uV1h6NGpFT1pYNzN6YnY5V2pVZFdOdjlLdFdEQnRPci9YcUNERHdBQUFXaVQ1M3VuQUFBRUF3QkhNRVVDSUcvWUdIM0loT2tNS1NCbWpwTGY4b092dzBFL3ZJWTkrckdLdmZ2bkNvOTJBaUVBdkFzRHJIc2hHWGxDKzgzVHJYSmNIOGdHK3pkelNDT0JYdTl4YlE1VkJHWXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBRThEL1MyV00xakNlREswOHdpSEFqbFQ1cjdzbWcwcjJUOXh5eWtaWnMrT3gyWlpLTVgwUHh0NlMyUVJLNE9sanZRMHhydVNxWW5XT3VWdFQyME9ZaUlyUmpTWkwrbWRTZjZTV3RYK1p5Y25XdmF0Mjh0TGN1NGo2YzhzSk5nYS9hNUNNWi9vL201YWlCN3g4YTljQThITVdMUExHZWVIRVVUREpWZThaQU5sdFhVK2ZzTHFWQ2t5T1hITEVMb3Y0NlhyS1k2a2cyOXpmbmx4c0VpOCsyTXhMOEZNcVdRZWFMOTRyNnV2OGtFTnhHOVh6cURHSlhhd2dCbVpXamlmWit5ZWdGcFQ2RnRIYno0WDZ0U3FUWUEyUUtUTnpHUGRBS2VYTjdrcGR0MlVGYzRZamI1Z1o5K1l3QURjaXMwUmJvdnVqUisrT3JKeTNEZkNJQ1d2ZDcwPSJdfQ
```

使用Base64解码得到一个json串如下：

```json
{
    "alg":"RS256",
    "x5c":[
        "MIIFjzCCBHegAwIBAgIQDrJ3e6z3i/ToFrNu8GbnkzANBgkqhkiG9w0BAQsFADByMQswCQYDVQQGEwJDTjElMCMGA1UEChMcVHJ1c3RBc2lhIFRlY2hub2xvZ2llcywgSW5jLjEdMBsGA1UECxMURG9tYWluIFZhbGlkYXRlZCBTU0wxHTAbBgNVBAMTFFRydXN0QXNpYSBUTFMgUlNBIENBMB4XDTE5MDEyODAwMDAwMFoXDTIwMDEyODEyMDAwMFowHjEcMBoGA1UEAxMTaHViLmMuc2hhb3F1bmxpdS5jbjCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBANpAKOKk3q4v67PoVgwjf8KBWTCWjVJKULJgPaxZ+92mFIQ+mmBIkCe3/dadJxDdAU5xo6deaZyXGuU47qA68vgvNWdmBHMoI98+y5ELWHa1jGq0ie/XROO18vvwdCvb/06T861pmxaUItF7XpW9Qpi/rBqsXSYISoJNgjN4hwmVltJmJnMTKB6gT2Nk+HCShGkWNKQ6lcm5YuJ+PnH0R58AvqsyZgj9vjawELmGGtfiYi9Mad31mf09A3cSTpKcOxBuumk5vV4Gd3hjsOdZFihjFAc3nq6j/bBRTAgJgNRhWwSnjJjqiltm52+jl8AzrW8gkO9PFG6eeYavN8MsZf0CAwEAAaOCAnMwggJvMB8GA1UdIwQYMBaAFH/TmfOgRw4xAFZWIo63zJ7dygGKMB0GA1UdDgQWBBR6bBhQYdeehkY6c4g5fMlkxUFgpjAeBgNVHREEFzAVghNodWIuYy5zaGFvcXVubGl1LmNuMA4GA1UdDwEB/wQEAwIFoDAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwTAYDVR0gBEUwQzA3BglghkgBhv1sAQIwKjAoBggrBgEFBQcCARYcaHR0cHM6Ly93d3cuZGlnaWNlcnQuY29tL0NQUzAIBgZngQwBAgEwfQYIKwYBBQUHAQEEcTBvMCEGCCsGAQUFBzABhhVodHRwOi8vb2NzcC5kY29jc3AuY24wSgYIKwYBBQUHMAKGPmh0dHA6Ly9jYWNlcnRzLmRpZ2l0YWxjZXJ0dmFsaWRhdGlvbi5jb20vVHJ1c3RBc2lhVExTUlNBQ0EuY3J0MAkGA1UdEwQCMAAwggEEBgorBgEEAdZ5AgQCBIH1BIHyAPAAdgC72d+8H4pxtZOUI5eqkntHOFeVCqtS6BqQlmQ2jh7RhQAAAWiT53rNAAAEAwBHMEUCIQD5Pte8AGL917xhYH1uWghapj4MmeyytUyTiuwKzHheUwIgEQbeM2a+4VURcDaRuSEX7NuG3TDaz8IFHEhGyqfkoiQAdgCHdb/nWXz4jEOZX73zbv9WjUdWNv9KtWDBtOr/XqCDDwAAAWiT53unAAAEAwBHMEUCIG/YGH3IhOkMKSBmjpLf8oOvw0E/vIY9+rGKvfvnCo92AiEAvAsDrHshGXlC+83TrXJcH8gG+zdzSCOBXu9xbQ5VBGYwDQYJKoZIhvcNAQELBQADggEBAE8D/S2WM1jCeDK08wiHAjlT5r7smg0r2T9xyykZZs+Ox2ZZKMX0Pxt6S2QRK4OljvQ0xruSqYnWOuVtT20OYiIrRjSZL+mdSf6SWtX+ZycnWvat28tLcu4j6c8sJNga/a5CMZ/o/m5aiB7x8a9cA8HMWLPLGeeHEUTDJVe8ZANltXU+fsLqVCkyOXHLELov46XrKY6kg29zfnlxsEi8+2MxL8FMqWQeaL94r6uv8kENxG9XzqDGJXawgBmZWjifZ+yegFpT6FtHbz4X6tSqTYA2QKTNzGPdAKeXN7kpdt2UFc4Yjb5gZ9+YwADcis0RbovujR++OrJy3DfCICWvd70="
    ]
}
```

`alg`代表服务端签名所使用算法，`x5c`为签名公钥。在此处我使用的算法为`RS256`即`RSA`算法配合`SHA256`算法，`x5c`公钥信息是在前面在registry中所配置的HTTPS SSL证书的基础上经由一系列变换而来。`x5c`对应的是一个Json数组，这个数组中的每一个字符串都代表了证书链中的一个证书。好，现在我们来演示如何由一个SSL证书推导出这个字符串：

推导这个字符串，只需要SSL证书的公钥部分，也就是那个`crt`文件，这个文件用记事本打开里面有类似如下的内容：

```
-----BEGIN CERTIFICATE-----
MIIFjzCCBHegAwIBAgIQDrJ3e6z3i/ToFrNu8GbnkzANBgkqhkiG9w0BAQsFADByMQswCQYDVQQGEwJDTjElMCMGA1UEChMcVHJ1c3RBc2lhIFRlY2hub2xvZ2llcywgSW5jLjEdMBsGA1UECxMURG9tYWluIFZhbGlkYXRlZCBTU0wxHTAbBgNVBAMTFFRydXN0QXNpYSBUTFMgUlNBIENBMB4XDTE5MDEyODAwMDAwMFoXDTIwMDEyODEyMDAwMFowHjEcMBoGA1UEAxMTaHViLmMuc2hhb3F1bmxpdS5jbjCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBANpAKOKk3q4v67PoVgwjf8KBWTCWjVJKULJgPaxZ+92mFIQ+mmBIkCe3/dadJxDdAU5xo6deaZyXGuU47qA68vgvNWdmBHMoI98+y5ELWHa1jGq0ie/XROO18vvwdCvb/06T861pmxaUItF7XpW9Qpi/rBqsXSYISoJNgjN4hwmVltJmJnMTKB6gT2Nk+HCShGkWNKQ6lcm5YuJ+PnH0R58AvqsyZgj9vjawELmGGtfiYi9Mad31mf09A3cSTpKcOxBuumk5vV4Gd3hjsOdZFihjFAc3nq6j/bBRTAgJgNRhWwSnjJjqiltm52+jl8AzrW8gkO9PFG6eeYavN8MsZf0CAwEAAaOCAnMwggJvMB8GA1UdIwQYMBaAFH/TmfOgRw4xAFZWIo63zJ7dygGKMB0GA1UdDgQWBBR6bBhQYdeehkY6c4g5fMlkxUFgpjAeBgNVHREEFzAVghNodWIuYy5zaGFvcXVubGl1LmNuMA4GA1UdDwEB/wQEAwIFoDAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwTAYDVR0gBEUwQzA3BglghkgBhv1sAQIwKjAoBggrBgEFBQcCARYcaHR0cHM6Ly93d3cuZGlnaWNlcnQuY29tL0NQUzAIBgZngQwBAgEwfQYIKwYBBQUHAQEEcTBvMCEGCCsGAQUFBzABhhVodHRwOi8vb2NzcC5kY29jc3AuY24wSgYIKwYBBQUHMAKGPmh0dHA6Ly9jYWNlcnRzLmRpZ2l0YWxjZXJ0dmFsaWRhdGlvbi5jb20vVHJ1c3RBc2lhVExTUlNBQ0EuY3J0MAkGA1UdEwQCMAAwggEEBgorBgEEAdZ5AgQCBIH1BIHyAPAAdgC72d+8H4pxtZOUI5eqkntHOFeVCqtS6BqQlmQ2jh7RhQAAAWiT53rNAAAEAwBHMEUCIQD5Pte8AGL917xhYH1uWghapj4MmeyytUyTiuwKzHheUwIgEQbeM2a+4VURcDaRuSEX7NuG3TDaz8IFHEhGyqfkoiQAdgCHdb/nWXz4jEOZX73zbv9WjUdWNv9KtWDBtOr/XqCDDwAAAWiT53unAAAEAwBHMEUCIG/YGH3IhOkMKSBmjpLf8oOvw0E/vIY9+rGKvfvnCo92AiEAvAsDrHshGXlC+83TrXJcH8gG+zdzSCOBXu9xbQ5VBGYwDQYJKoZIhvcNAQELBQADggEBAE8D/S2WM1jCeDK08wiHAjlT5r7smg0r2T9xyykZZs+Ox2ZZKMX0Pxt6S2QRK4OljvQ0xruSqYnWOuVtT20OYiIrRjSZL+mdSf6SWtX+ZycnWvat28tLcu4j6c8sJNga/a5CMZ/o/m5aiB7x8a9cA8HMWLPLGeeHEUTDJVe8ZANltXU+fsLqVCkyOXHLELov46XrKY6kg29zfnlxsEi8+2MxL8FMqWQeaL94r6uv8kENxG9XzqDGJXawgBmZWjifZ+yegFpT6FtHbz4X6tSqTYA2QKTNzGPdAKeXN7kpdt2UFc4Yjb5gZ9+YwADcis0RbovujR++OrJy3DfCICWvd70=
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIErjCCA5agAwIBAgIQBYAmfwbylVM0jhwYWl7uLjANBgkqhkiG9w0BAQsFADBhMQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3d3cuZGlnaWNlcnQuY29tMSAwHgYDVQQDExdEaWdpQ2VydCBHbG9iYWwgUm9vdCBDQTAeFw0xNzEyMDgxMjI4MjZaFw0yNzEyMDgxMjI4MjZaMHIxCzAJBgNVBAYTAkNOMSUwIwYDVQQKExxUcnVzdEFzaWEgVGVjaG5vbG9naWVzLCBJbmMuMR0wGwYDVQQLExREb21haW4gVmFsaWRhdGVkIFNTTDEdMBsGA1UEAxMUVHJ1c3RBc2lhIFRMUyBSU0EgQ0EwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCgWa9X+ph+wAm8Yh1Fk1MjKbQ5QwBOOKVaZR/OfCh+F6f93u7vZHGcUU/lvVGgUQnbzJhR1UV2epJae+m7cxnXIKdD0/VS9btAgwJszGFvwoqXeaCqFoP71wPmXjjUwLT70+qvX4hdyYfOJcjeTz5QKtg8zQwxaK9x4JT9CoOmoVdVhEBAiD3DwR5fFgOHDwwGxdJWVBvktnoAzjdTLXDdbSVC5jZ0u8oq9BiTDv7jAlsB5F8aZgvSZDOQeFrwaOTbKWSEInEhnchKZTD1dz6aBlk1xGEI5PZWAnVAba/ofH33ktymaTDsE6xRDnW97pDkimCRak6CEbfe3dXw6OV5AgMBAAGjggFPMIIBSzAdBgNVHQ4EFgQUf9OZ86BHDjEAVlYijrfMnt3KAYowHwYDVR0jBBgwFoAUA95QNVbRTLtm8KPiGxvDl7I90VUwDgYDVR0PAQH/BAQDAgGGMB0GA1UdJQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjASBgNVHRMBAf8ECDAGAQH/AgEAMDQGCCsGAQUFBwEBBCgwJjAkBggrBgEFBQcwAYYYaHR0cDovL29jc3AuZGlnaWNlcnQuY29tMEIGA1UdHwQ7MDkwN6A1oDOGMWh0dHA6Ly9jcmwzLmRpZ2ljZXJ0LmNvbS9EaWdpQ2VydEdsb2JhbFJvb3RDQS5jcmwwTAYDVR0gBEUwQzA3BglghkgBhv1sAQIwKjAoBggrBgEFBQcCARYcaHR0cHM6Ly93d3cuZGlnaWNlcnQuY29tL0NQUzAIBgZngQwBAgEwDQYJKoZIhvcNAQELBQADggEBAK3dVOj5dlv4MzK2i233lDYvyJ3slFY2X2HKTYGte8nbK6i5/fsDImMYihAkp6VaNY/en8WZ5qcrQPVLuJrJDSXT04NnMeZOQDUoj/NHAmdfCBB/h1bZ5OGK6Sf1h5Yx/5wR4f3TUoPgGlnU7EuPISLNdMRiDrXntcImDAiRvkh5GJuH4YCVE6XEntqaNIgGkRwxKSgnU3Id3iuFbW9FUQ9Qqtb1GX91AJ7i4153TikGgYCdwYkBURD8gSVe8OAco6IfZOYt/TEwii1Ivi1CqnuUlWpsF1LdQNIdfbW3TSe0BhQa7ifbVIfvPWHYOu3rkg1ZeMo6XRU9B4n5VyJYRmE=
-----END CERTIFICATE-----
```

类似于这样的证书编码方式叫做PEM编码，我们首先需要将其转化为DER编码，假设上述使用pem编码的证书保存在文件`hub.crt`中，我们现在将其转化为DER编码并将结果保存于文件`hub.der`中：

```sh
openssl x509 -in hub.crt -outform der-out -out hub.der
```

然后对生成的以DER编码形式保存的证书取Base64即可得到这个`x5c`，在Java中我们可以使用如下代码获取文件的Base64：

```java
Base64.getEncoder().encodeToString(getByteArrayFromFile("/root/ssl/hub.der"))
```

有关这个`x5c`的具体细节可以参数[RFC-7515](https://tools.ietf.org/html/rfc7515#section-4.1.6)章节4.1.6，编码信息援引自：

> The certificate or certificate chain is represented as a JSON array of certificate value strings.  Each string in the array is a base64-encoded (Section 4 of [RFC4648] -- not base64url-encoded) DER [ITU.X690.2008] PKIX certificate value.

至此，Header部分全部信息我们就分析完毕了。此后我们分析其Claims部分，用同样的方法得到Claims部分然后用base64解码得：

```
{
    "access":[],
    "iss":"和你registry配置文件中填写的issuer字段相同",
    "sub":"用户登录的用户名",
    "aud":"和你registry配置文件中填写的service字段相同",
    "exp":1556172046,
    "iat":1555999246,
    "jti":"e1f7ccae-0838-4374-82f1-ad685aceb622"
}
```

`exp`字段代表token的有效截止时间，为unix时间戳形式，超过这个时间的token将无法使用。`iat`即代表token的签发时间，即issued at的简写。`jti`代表这个token的序列号，至于你想怎么生成这个序列号，随你的便，在此处我用的是UUID的方式来生成这个序列号，在Java中代码如下：

```java
UUID.randomUUID().toString()
```

同时，在此处，我们需要注意，docker扩充了标准中的Claims部分，在其中加入了一个`access`字段，这个字段用于标注当前用户对repository的权限信息，其同样为一个json数组，实例如下：

```json
"access": [
    {
        "type": "repository",
        "name": "samalba/my-app",
        "actions": [
            "pull",
            "push"
        ]
    }
]
```

这个json数组中包含若干json对象，每一个json对象用于对一个资源进行标注，其中type字段代表了所标注资源的类型，一般为repository即为image仓库，name为资源的名称，一般为docker image的定位符，action数组表明了当前registry所允许当前用户执行在这个资源上执行的操作。如上面的例子就代表了对当前registry中存储的`samalba/my-app`这个repository具有拉取(pull)和上传(push)的权限。

### 0x05 使用JJWT生成符合条件的Json Web Token

我们可以在Java中使用JJWT库来生成Json Web Token，首先在maven中添加相关依赖：

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.1</version>
</dependency>
```

然后代码如下：

```java
Date issuedAt = new Date();
int expiresIn = 48 * 3600 * 1000;
try {
    // SSL证书一般采用RSA算法
    KeyFactory keyFactory = KeyFactory.getInstance("RSA");
    // 从文件中获取RSA私钥，用于服务端签名
    EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(getByteArrayFromFile("/root/ssl/hub.pkcs8"));
    RSAPrivateKey key = (RSAPrivateKey) keyFactory.generatePrivate(keySpec);
    // 生成JWT头部x5c字段，注意x5c必需是一个json数组
    List<String> x5c = new ArrayList<>();
    x5c.add(Base64.getEncoder().encodeToString(getByteArrayFromFile("/root/ssl/hub.der")));
    // 生成Json Web Token
    String token = Jwts.builder()
            .setHeaderParam(JwsHeader.ALGORITHM, "JWT")
            // https://tools.ietf.org/html/rfc7515#section-4.1.6
            .setHeaderParam(JwsHeader.X509_CERT_CHAIN, x5c)
            // 对生成的JWT进行签名
            .signWith(SignatureAlgorithm.RS256, key)
            // docker registry扩充字段，用于标识权限
            .claim("access", new ArrayList<>())
            // 和你registry配置文件中填写的issuer字段相同
            .setIssuer("Shaoqun Liu")
            // 当前代码在Spring Security环境下，所以可以直接使用authentication.getName()以获取用户登录所使用的用户名
            .setSubject(authentication.getName())
            // 和你registry配置文件中填写的service字段相同
            .setAudience("An example docker registry")
            // 设置其他杂项
            .setExpiration(new Date(issuedAt.getTime() + expiresIn))
            .setIssuedAt(issuedAt)
            .setId(UUID.randomUUID().toString())
            // 生成字符串形式的Json Web Token
            .compact();
} catch (Exception e) {
    throw new IOException(e.getMessage());
}
```

### 0x06 参考文献

[RFC-7515 JSON Web Signature (JWS)](https://tools.ietf.org/html/rfc7515)

[Token Authentication Implementation](https://docs.docker.com/registry/spec/auth/jwt/)

[Token Authentication Specification](https://docs.docker.com/registry/spec/auth/token/)

[Token Scope Documentation](https://docs.docker.com/registry/spec/auth/scope/)

[Configuring a registry](https://docs.docker.com/registry/configuration/#token)

[Deploy a registry server](https://docs.docker.com/registry/deploying/#run-a-local-registry)

[Authorization for Private Docker Registry](https://medium.com/@maanadev/authorization-for-private-docker-registry-d1f6bf74552f)

[从Registry到Registry V2，一篇文章看懂token流程认证！](http://rdc.hundsun.com/portal/article/758.mhtml)

[docker login执行流程与原理](https://my.oschina.net/kingfsen/blog/2252203)

[java 读取证书的PublicKey](https://blog.csdn.net/u010071621/article/details/54890252)

[java读取openssl生成的private key文件生成密钥的问题](https://blog.csdn.net/zhang_xiaoyan/article/details/8911964)

在线工具：

[在线json代码格式化](https://www.json.cn/)

[在线base32编码解码](https://www.qqxiuzi.cn/bianma/base.php)

[在线base64编码解码](http://tool.oschina.net/encrypt?type=3)